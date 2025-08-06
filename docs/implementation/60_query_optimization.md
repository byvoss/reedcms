# Query Optimization

## Overview

Query Optimization Implementation für ReedCMS. Database Query Optimization mit Monitoring und Auto-Tuning.

## Query Optimizer

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use sqlx::{PgPool, postgres::PgQueryResult};

/// Main query optimization service
pub struct QueryOptimizer {
    analyzer: Arc<QueryAnalyzer>,
    planner: Arc<QueryPlanner>,
    cache: Arc<QueryCache>,
    monitor: Arc<QueryMonitor>,
    config: OptimizerConfig,
}

impl QueryOptimizer {
    pub fn new(pool: Arc<PgPool>, config: OptimizerConfig) -> Self {
        let analyzer = Arc::new(QueryAnalyzer::new(pool.clone()));
        let planner = Arc::new(QueryPlanner::new());
        let cache = Arc::new(QueryCache::new(config.cache.clone()));
        let monitor = Arc::new(QueryMonitor::new(pool.clone()));
        
        Self {
            analyzer,
            planner,
            cache,
            monitor,
            config,
        }
    }
    
    /// Optimize and execute query
    pub async fn execute_optimized<T>(
        &self,
        query: Query,
    ) -> Result<Vec<T>>
    where
        T: for<'r> sqlx::FromRow<'r, sqlx::postgres::PgRow> + Send + Unpin,
    {
        let start = std::time::Instant::now();
        
        // Check query cache
        if let Some(cached) = self.cache.get::<Vec<T>>(&query).await? {
            self.monitor.record_cache_hit(&query, start.elapsed()).await;
            return Ok(cached);
        }
        
        // Analyze query
        let analysis = self.analyzer.analyze(&query).await?;
        
        // Optimize query
        let optimized = self.planner.optimize(query, &analysis).await?;
        
        // Execute query
        let result = self.execute_query::<T>(&optimized).await?;
        
        // Cache result if appropriate
        if self.should_cache(&optimized, &analysis) {
            self.cache.set(&optimized, &result).await?;
        }
        
        // Record metrics
        self.monitor.record_execution(&optimized, &analysis, start.elapsed()).await;
        
        Ok(result)
    }
    
    /// Build optimized query
    pub async fn build_query(&self, builder: QueryBuilder) -> Result<Query> {
        // Analyze builder intent
        let intent = self.analyze_intent(&builder)?;
        
        // Generate optimized query
        let query = match intent {
            QueryIntent::SingleRow => self.build_single_row_query(builder)?,
            QueryIntent::MultiRow => self.build_multi_row_query(builder)?,
            QueryIntent::Aggregation => self.build_aggregation_query(builder)?,
            QueryIntent::Join => self.build_join_query(builder)?,
        };
        
        Ok(query)
    }
    
    /// Execute raw query
    async fn execute_query<T>(&self, query: &Query) -> Result<Vec<T>>
    where
        T: for<'r> sqlx::FromRow<'r, sqlx::postgres::PgRow> + Send + Unpin,
    {
        match query {
            Query::Select(sql, params) => {
                let mut query = sqlx::query_as::<_, T>(sql);
                
                for param in params {
                    query = query.bind(param);
                }
                
                let rows = query
                    .fetch_all(&*self.analyzer.pool)
                    .await?;
                
                Ok(rows)
            }
            _ => Err(OptimizerError::UnsupportedQuery),
        }
    }
    
    /// Check if query should be cached
    fn should_cache(&self, query: &Query, analysis: &QueryAnalysis) -> bool {
        // Don't cache if disabled
        if !self.config.enable_cache {
            return false;
        }
        
        // Don't cache writes
        if query.is_write() {
            return false;
        }
        
        // Cache based on cost and frequency
        analysis.estimated_cost > self.config.cache_threshold_cost ||
        analysis.access_frequency > self.config.cache_threshold_frequency
    }
}

/// Query representation
#[derive(Debug, Clone, Hash, Eq, PartialEq)]
pub enum Query {
    Select(String, Vec<SqlValue>),
    Insert(String, Vec<SqlValue>),
    Update(String, Vec<SqlValue>),
    Delete(String, Vec<SqlValue>),
}

impl Query {
    fn is_write(&self) -> bool {
        matches!(self, Query::Insert(..) | Query::Update(..) | Query::Delete(..))
    }
}
```

## Query Analyzer

```rust
/// Query analysis service
pub struct QueryAnalyzer {
    pool: Arc<PgPool>,
    stats_cache: Arc<RwLock<HashMap<String, TableStats>>>,
}

impl QueryAnalyzer {
    /// Analyze query
    pub async fn analyze(&self, query: &Query) -> Result<QueryAnalysis> {
        let mut analysis = QueryAnalysis::default();
        
        // Parse query
        let parsed = self.parse_query(query)?;
        
        // Get table statistics
        for table in &parsed.tables {
            let stats = self.get_table_stats(table).await?;
            analysis.table_stats.insert(table.clone(), stats);
        }
        
        // Analyze indexes
        analysis.available_indexes = self.analyze_indexes(&parsed).await?;
        
        // Estimate cost
        analysis.estimated_cost = self.estimate_cost(&parsed, &analysis.table_stats)?;
        
        // Check for common issues
        analysis.issues = self.detect_issues(&parsed, &analysis)?;
        
        // Get execution plan
        if self.should_explain(&parsed) {
            analysis.execution_plan = Some(self.get_execution_plan(query).await?);
        }
        
        Ok(analysis)
    }
    
    /// Get table statistics
    async fn get_table_stats(&self, table: &str) -> Result<TableStats> {
        // Check cache
        if let Some(stats) = self.stats_cache.read().await.get(table) {
            return Ok(stats.clone());
        }
        
        // Query statistics
        let stats = sqlx::query_as!(
            TableStats,
            r#"
            SELECT
                schemaname,
                tablename,
                n_live_tup as row_count,
                n_dead_tup as dead_rows,
                last_vacuum,
                last_autovacuum,
                last_analyze,
                last_autoanalyze
            FROM pg_stat_user_tables
            WHERE tablename = $1
            "#,
            table
        )
        .fetch_one(&*self.pool)
        .await?;
        
        // Cache stats
        self.stats_cache.write().await.insert(table.to_string(), stats.clone());
        
        Ok(stats)
    }
    
    /// Analyze available indexes
    async fn analyze_indexes(&self, parsed: &ParsedQuery) -> Result<Vec<IndexInfo>> {
        let mut indexes = Vec::new();
        
        for table in &parsed.tables {
            let table_indexes = sqlx::query_as!(
                IndexInfo,
                r#"
                SELECT
                    i.indexname as name,
                    i.tablename,
                    pg_get_indexdef(x.indexrelid) as definition,
                    x.indisunique as is_unique,
                    x.indisprimary as is_primary,
                    pg_relation_size(x.indexrelid) as size_bytes
                FROM pg_indexes i
                JOIN pg_class c ON c.relname = i.indexname
                JOIN pg_index x ON x.indexrelid = c.oid
                WHERE i.tablename = $1
                "#,
                table
            )
            .fetch_all(&*self.pool)
            .await?;
            
            indexes.extend(table_indexes);
        }
        
        Ok(indexes)
    }
    
    /// Get execution plan
    async fn get_execution_plan(&self, query: &Query) -> Result<ExecutionPlan> {
        let explain_query = match query {
            Query::Select(sql, _) => format!("EXPLAIN (ANALYZE, BUFFERS) {}", sql),
            _ => return Err(OptimizerError::ExplainNotSupported),
        };
        
        let rows = sqlx::query(&explain_query)
            .fetch_all(&*self.pool)
            .await?;
        
        let plan_text = rows
            .iter()
            .map(|row| row.get::<String, _>(0))
            .collect::<Vec<_>>()
            .join("\n");
        
        Ok(ExecutionPlan::parse(&plan_text)?)
    }
    
    /// Detect common query issues
    fn detect_issues(
        &self,
        parsed: &ParsedQuery,
        analysis: &QueryAnalysis,
    ) -> Result<Vec<QueryIssue>> {
        let mut issues = Vec::new();
        
        // Check for missing indexes
        for condition in &parsed.where_conditions {
            if !self.has_index_for_condition(condition, &analysis.available_indexes) {
                issues.push(QueryIssue::MissingIndex {
                    table: condition.table.clone(),
                    column: condition.column.clone(),
                });
            }
        }
        
        // Check for N+1 queries
        if parsed.query_type == QueryType::Select && parsed.limit == Some(1) {
            issues.push(QueryIssue::PotentialNPlusOne);
        }
        
        // Check for large table scans
        for (table, stats) in &analysis.table_stats {
            if stats.row_count > 100000 && !self.has_limiting_condition(table, parsed) {
                issues.push(QueryIssue::LargeTableScan {
                    table: table.clone(),
                    row_count: stats.row_count,
                });
            }
        }
        
        // Check for missing JOIN conditions
        if parsed.tables.len() > 1 && parsed.join_conditions.is_empty() {
            issues.push(QueryIssue::MissingJoinCondition);
        }
        
        Ok(issues)
    }
}

/// Query analysis result
#[derive(Debug, Default)]
pub struct QueryAnalysis {
    pub table_stats: HashMap<String, TableStats>,
    pub available_indexes: Vec<IndexInfo>,
    pub estimated_cost: f64,
    pub access_frequency: u64,
    pub issues: Vec<QueryIssue>,
    pub execution_plan: Option<ExecutionPlan>,
}

#[derive(Debug, Clone)]
pub struct TableStats {
    pub schemaname: String,
    pub tablename: String,
    pub row_count: i64,
    pub dead_rows: i64,
    pub last_vacuum: Option<chrono::DateTime<chrono::Utc>>,
    pub last_autovacuum: Option<chrono::DateTime<chrono::Utc>>,
    pub last_analyze: Option<chrono::DateTime<chrono::Utc>>,
    pub last_autoanalyze: Option<chrono::DateTime<chrono::Utc>>,
}

#[derive(Debug)]
pub enum QueryIssue {
    MissingIndex { table: String, column: String },
    LargeTableScan { table: String, row_count: i64 },
    PotentialNPlusOne,
    MissingJoinCondition,
    SuboptimalJoinOrder,
    RedundantDistinct,
    UnusedIndex { index_name: String },
}
```

## Query Planner

```rust
/// Query planning and optimization
pub struct QueryPlanner {
    strategies: Vec<Box<dyn OptimizationStrategy>>,
}

impl QueryPlanner {
    pub fn new() -> Self {
        let strategies: Vec<Box<dyn OptimizationStrategy>> = vec![
            Box::new(IndexStrategy::new()),
            Box::new(JoinOrderStrategy::new()),
            Box::new(PredicatePushdownStrategy::new()),
            Box::new(SubqueryStrategy::new()),
            Box::new(PaginationStrategy::new()),
        ];
        
        Self { strategies }
    }
    
    /// Optimize query
    pub async fn optimize(
        &self,
        mut query: Query,
        analysis: &QueryAnalysis,
    ) -> Result<Query> {
        // Apply each optimization strategy
        for strategy in &self.strategies {
            if strategy.applicable(&query, analysis) {
                query = strategy.optimize(query, analysis).await?;
            }
        }
        
        Ok(query)
    }
}

/// Optimization strategy trait
#[async_trait]
trait OptimizationStrategy: Send + Sync {
    fn applicable(&self, query: &Query, analysis: &QueryAnalysis) -> bool;
    async fn optimize(&self, query: Query, analysis: &QueryAnalysis) -> Result<Query>;
}

/// Index usage optimization
struct IndexStrategy;

impl IndexStrategy {
    fn new() -> Self {
        Self
    }
}

#[async_trait]
impl OptimizationStrategy for IndexStrategy {
    fn applicable(&self, query: &Query, analysis: &QueryAnalysis) -> bool {
        matches!(query, Query::Select(..)) && !analysis.available_indexes.is_empty()
    }
    
    async fn optimize(&self, query: Query, analysis: &QueryAnalysis) -> Result<Query> {
        if let Query::Select(sql, params) = query {
            // Add index hints if beneficial
            let optimized_sql = self.add_index_hints(&sql, analysis)?;
            Ok(Query::Select(optimized_sql, params))
        } else {
            Ok(query)
        }
    }
}

/// Join order optimization
struct JoinOrderStrategy;

impl JoinOrderStrategy {
    fn new() -> Self {
        Self
    }
    
    /// Calculate optimal join order
    fn calculate_join_order(&self, tables: &[TableRef], stats: &HashMap<String, TableStats>) -> Vec<TableRef> {
        // Simple heuristic: order by table size
        let mut ordered = tables.to_vec();
        ordered.sort_by_key(|t| {
            stats.get(&t.name)
                .map(|s| s.row_count)
                .unwrap_or(i64::MAX)
        });
        ordered
    }
}

#[async_trait]
impl OptimizationStrategy for JoinOrderStrategy {
    fn applicable(&self, query: &Query, _analysis: &QueryAnalysis) -> bool {
        if let Query::Select(sql, _) = query {
            sql.to_lowercase().contains(" join ")
        } else {
            false
        }
    }
    
    async fn optimize(&self, query: Query, analysis: &QueryAnalysis) -> Result<Query> {
        if let Query::Select(sql, params) = query {
            // Parse and reorder joins
            let parsed = parse_sql(&sql)?;
            let optimal_order = self.calculate_join_order(&parsed.tables, &analysis.table_stats);
            let reordered_sql = rebuild_query_with_join_order(&parsed, &optimal_order)?;
            
            Ok(Query::Select(reordered_sql, params))
        } else {
            Ok(query)
        }
    }
}
```

## Query Cache

```rust
/// Query result cache
pub struct QueryCache {
    cache: Arc<CacheManager>,
    config: QueryCacheConfig,
}

impl QueryCache {
    pub fn new(config: QueryCacheConfig) -> Self {
        let cache = Arc::new(CacheManager::new(config.cache_config.clone()).unwrap());
        
        Self { cache, config }
    }
    
    /// Get cached query result
    pub async fn get<T>(&self, query: &Query) -> Result<Option<T>>
    where
        T: CacheValue,
    {
        if !self.should_cache_query(query) {
            return Ok(None);
        }
        
        let key = self.build_cache_key(query);
        self.cache.get(&key).await
    }
    
    /// Cache query result
    pub async fn set<T>(&self, query: &Query, result: &T) -> Result<()>
    where
        T: CacheValue,
    {
        if !self.should_cache_query(query) {
            return Ok(());
        }
        
        let key = self.build_cache_key(query);
        let ttl = self.calculate_ttl(query);
        
        self.cache.set(&key, result, Some(ttl)).await
    }
    
    /// Invalidate cached queries
    pub async fn invalidate_table(&self, table: &str) -> Result<()> {
        let tags = vec![format!("table:{}", table)];
        self.cache.invalidate_by_tags(&tags).await?;
        Ok(())
    }
    
    /// Build cache key for query
    fn build_cache_key(&self, query: &Query) -> String {
        use std::hash::{Hash, Hasher};
        use std::collections::hash_map::DefaultHasher;
        
        let mut hasher = DefaultHasher::new();
        query.hash(&mut hasher);
        
        format!("query:{:x}", hasher.finish())
    }
    
    /// Check if query should be cached
    fn should_cache_query(&self, query: &Query) -> bool {
        match query {
            Query::Select(sql, _) => {
                // Don't cache queries with random/time functions
                !sql.to_lowercase().contains("random()") &&
                !sql.to_lowercase().contains("now()") &&
                !sql.to_lowercase().contains("current_timestamp")
            }
            _ => false,
        }
    }
    
    /// Calculate TTL for query
    fn calculate_ttl(&self, query: &Query) -> u64 {
        match query {
            Query::Select(sql, _) => {
                if sql.contains("COUNT(") || sql.contains("SUM(") {
                    self.config.aggregation_ttl
                } else {
                    self.config.default_ttl
                }
            }
            _ => 0,
        }
    }
}
```

## Query Monitoring

```rust
/// Query performance monitoring
pub struct QueryMonitor {
    pool: Arc<PgPool>,
    metrics: Arc<QueryMetrics>,
    slow_query_log: Arc<SlowQueryLog>,
}

impl QueryMonitor {
    /// Record query execution
    pub async fn record_execution(
        &self,
        query: &Query,
        analysis: &QueryAnalysis,
        duration: Duration,
    ) {
        // Update metrics
        self.metrics.record_query(query, duration).await;
        
        // Check for slow query
        if duration > self.slow_query_threshold() {
            self.slow_query_log.log(query, analysis, duration).await;
        }
        
        // Update query statistics
        self.update_query_stats(query, duration).await;
    }
    
    /// Get slow queries
    pub async fn get_slow_queries(&self, limit: usize) -> Result<Vec<SlowQuery>> {
        self.slow_query_log.get_recent(limit).await
    }
    
    /// Analyze query patterns
    pub async fn analyze_patterns(&self) -> Result<QueryPatternAnalysis> {
        let patterns = self.metrics.get_patterns().await?;
        
        let mut analysis = QueryPatternAnalysis {
            most_frequent: Vec::new(),
            slowest: Vec::new(),
            most_variable: Vec::new(),
            recommendations: Vec::new(),
        };
        
        // Find most frequent queries
        analysis.most_frequent = patterns
            .iter()
            .sorted_by_key(|p| p.execution_count)
            .rev()
            .take(10)
            .cloned()
            .collect();
        
        // Find slowest queries
        analysis.slowest = patterns
            .iter()
            .sorted_by_key(|p| p.avg_duration)
            .rev()
            .take(10)
            .cloned()
            .collect();
        
        // Generate recommendations
        for pattern in &patterns {
            if pattern.avg_duration > Duration::from_secs(1) {
                analysis.recommendations.push(Recommendation {
                    query_pattern: pattern.pattern.clone(),
                    suggestion: "Consider adding indexes or optimizing query".to_string(),
                    priority: Priority::High,
                });
            }
        }
        
        Ok(analysis)
    }
    
    /// Monitor live queries
    pub async fn get_active_queries(&self) -> Result<Vec<ActiveQuery>> {
        let queries = sqlx::query_as!(
            ActiveQuery,
            r#"
            SELECT
                pid,
                usename as username,
                application_name,
                client_addr::text as client_addr,
                query_start,
                state,
                query
            FROM pg_stat_activity
            WHERE state != 'idle'
                AND pid != pg_backend_pid()
            ORDER BY query_start
            "#
        )
        .fetch_all(&*self.pool)
        .await?;
        
        Ok(queries)
    }
    
    /// Kill long-running query
    pub async fn kill_query(&self, pid: i32) -> Result<()> {
        sqlx::query!("SELECT pg_terminate_backend($1)", pid)
            .execute(&*self.pool)
            .await?;
        
        Ok(())
    }
}

/// Query metrics
pub struct QueryMetrics {
    queries: Arc<RwLock<HashMap<QueryPattern, QueryStats>>>,
}

impl QueryMetrics {
    /// Record query execution
    pub async fn record_query(&self, query: &Query, duration: Duration) {
        let pattern = QueryPattern::from_query(query);
        let mut queries = self.queries.write().await;
        
        let stats = queries.entry(pattern).or_insert_with(QueryStats::default);
        stats.record_execution(duration);
    }
    
    /// Get query patterns
    pub async fn get_patterns(&self) -> Result<Vec<QueryPatternStats>> {
        let queries = self.queries.read().await;
        
        Ok(queries
            .iter()
            .map(|(pattern, stats)| QueryPatternStats {
                pattern: pattern.clone(),
                execution_count: stats.count,
                avg_duration: stats.average_duration(),
                min_duration: stats.min_duration,
                max_duration: stats.max_duration,
                std_deviation: stats.std_deviation(),
            })
            .collect())
    }
}

#[derive(Debug, Clone, Hash, Eq, PartialEq)]
struct QueryPattern {
    query_type: String,
    tables: Vec<String>,
    has_joins: bool,
    has_subquery: bool,
    has_aggregation: bool,
}

#[derive(Debug, Default)]
struct QueryStats {
    count: u64,
    total_duration: Duration,
    min_duration: Duration,
    max_duration: Duration,
    sum_squared_duration: u128,
}

impl QueryStats {
    fn record_execution(&mut self, duration: Duration) {
        self.count += 1;
        self.total_duration += duration;
        
        if self.count == 1 || duration < self.min_duration {
            self.min_duration = duration;
        }
        
        if duration > self.max_duration {
            self.max_duration = duration;
        }
        
        let micros = duration.as_micros();
        self.sum_squared_duration += micros * micros;
    }
    
    fn average_duration(&self) -> Duration {
        if self.count == 0 {
            Duration::from_secs(0)
        } else {
            self.total_duration / self.count as u32
        }
    }
    
    fn std_deviation(&self) -> f64 {
        if self.count < 2 {
            return 0.0;
        }
        
        let mean = self.total_duration.as_micros() as f64 / self.count as f64;
        let variance = (self.sum_squared_duration as f64 / self.count as f64) - (mean * mean);
        variance.sqrt()
    }
}
```

## Auto-Indexing

```rust
/// Automatic index recommendation and creation
pub struct AutoIndexer {
    optimizer: Arc<QueryOptimizer>,
    analyzer: Arc<IndexAnalyzer>,
    config: AutoIndexConfig,
}

impl AutoIndexer {
    /// Analyze and recommend indexes
    pub async fn analyze_workload(&self) -> Result<Vec<IndexRecommendation>> {
        let mut recommendations = Vec::new();
        
        // Get query patterns
        let patterns = self.optimizer.monitor.analyze_patterns().await?;
        
        // Analyze each slow query
        for pattern in patterns.slowest {
            let missing_indexes = self.analyzer
                .find_missing_indexes(&pattern)
                .await?;
            
            for index in missing_indexes {
                recommendations.push(IndexRecommendation {
                    table: index.table,
                    columns: index.columns,
                    estimated_improvement: index.estimated_improvement,
                    create_statement: index.generate_create_statement(),
                });
            }
        }
        
        // Deduplicate and prioritize
        recommendations.sort_by(|a, b| {
            b.estimated_improvement
                .partial_cmp(&a.estimated_improvement)
                .unwrap()
        });
        
        recommendations.dedup_by(|a, b| {
            a.table == b.table && a.columns == b.columns
        });
        
        Ok(recommendations)
    }
    
    /// Auto-create beneficial indexes
    pub async fn auto_create_indexes(&self) -> Result<Vec<String>> {
        if !self.config.auto_create {
            return Ok(Vec::new());
        }
        
        let recommendations = self.analyze_workload().await?;
        let mut created = Vec::new();
        
        for rec in recommendations {
            if rec.estimated_improvement > self.config.min_improvement_threshold {
                // Create index
                match self.create_index(&rec).await {
                    Ok(_) => {
                        created.push(rec.create_statement);
                        info!("Auto-created index: {}", rec.create_statement);
                    }
                    Err(e) => {
                        warn!("Failed to create index: {}", e);
                    }
                }
                
                // Limit number of auto-created indexes
                if created.len() >= self.config.max_auto_indexes {
                    break;
                }
            }
        }
        
        Ok(created)
    }
}

#[derive(Debug)]
pub struct IndexRecommendation {
    pub table: String,
    pub columns: Vec<String>,
    pub estimated_improvement: f64,
    pub create_statement: String,
}

impl IndexRecommendation {
    fn generate_create_statement(&self) -> String {
        format!(
            "CREATE INDEX CONCURRENTLY idx_{}_{} ON {} ({})",
            self.table,
            self.columns.join("_"),
            self.table,
            self.columns.join(", ")
        )
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OptimizerConfig {
    pub enable_cache: bool,
    pub cache_threshold_cost: f64,
    pub cache_threshold_frequency: u64,
    pub analyze_execution_plans: bool,
    pub slow_query_threshold_ms: u64,
    pub cache: QueryCacheConfig,
    pub auto_index: AutoIndexConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QueryCacheConfig {
    pub cache_config: CacheConfig,
    pub default_ttl: u64,
    pub aggregation_ttl: u64,
    pub max_cached_queries: usize,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AutoIndexConfig {
    pub enabled: bool,
    pub auto_create: bool,
    pub analysis_interval: Duration,
    pub min_improvement_threshold: f64,
    pub max_auto_indexes: usize,
}

impl Default for OptimizerConfig {
    fn default() -> Self {
        Self {
            enable_cache: true,
            cache_threshold_cost: 100.0,
            cache_threshold_frequency: 10,
            analyze_execution_plans: true,
            slow_query_threshold_ms: 1000,
            cache: QueryCacheConfig {
                cache_config: CacheConfig::default(),
                default_ttl: 300,
                aggregation_ttl: 60,
                max_cached_queries: 1000,
            },
            auto_index: AutoIndexConfig {
                enabled: true,
                auto_create: false,
                analysis_interval: Duration::from_secs(3600),
                min_improvement_threshold: 0.5,
                max_auto_indexes: 5,
            },
        }
    }
}
```

## Summary

Diese Query Optimization Implementation bietet:
- **Query Analysis** - Execution Plans, Statistics
- **Query Planning** - Join Order, Index Usage
- **Query Caching** - Result Cache mit Invalidation
- **Performance Monitoring** - Slow Query Log, Patterns
- **Auto-Indexing** - Index Recommendations
- **Live Monitoring** - Active Query Management

Comprehensive Query Optimization für Database Performance.