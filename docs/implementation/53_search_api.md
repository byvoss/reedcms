# Search API

## Overview

Search API Implementation für ReedCMS. RESTful API und GraphQL Interface für Search Functionality.

## REST API

```rust
use axum::{
    Router,
    extract::{Query, Path, State},
    response::Json,
    http::StatusCode,
};
use serde::{Serialize, Deserialize};

/// Search API routes
pub fn search_routes() -> Router<AppState> {
    Router::new()
        .route("/api/search", get(search_handler))
        .route("/api/search/suggest", get(suggest_handler))
        .route("/api/search/facets", get(facets_handler))
        .route("/api/search/index/:id", post(index_handler))
        .route("/api/search/index/:id", delete(remove_handler))
        .route("/api/search/reindex", post(reindex_handler))
        .route("/api/search/status", get(status_handler))
}

/// Main search handler
async fn search_handler(
    State(state): State<AppState>,
    Query(params): Query<SearchParams>,
) -> Result<Json<SearchResponse>, ApiError> {
    // Build search query
    let query = SearchQuery {
        query: params.q.clone(),
        filters: build_filters(&params),
        options: SearchOptions {
            limit: params.limit.unwrap_or(20),
            offset: params.offset.unwrap_or(0),
            sort_by: params.sort_by.unwrap_or(SortField::Relevance),
            sort_order: params.sort_order.unwrap_or(SortOrder::Desc),
            query_mode: params.mode.unwrap_or(QueryMode::Simple),
            fuzzy_distance: params.fuzzy.unwrap_or(2),
            field_boosts: parse_field_boosts(&params.boost),
            highlight: params.highlight.unwrap_or(true),
            include_facets: params.facets.unwrap_or(false),
        },
        facets: parse_facet_config(&params.facet_fields),
    };
    
    // Execute search
    let results = state.search_engine
        .search(&query)
        .await
        .map_err(|e| ApiError::internal(e))?;
    
    // Build response
    let response = SearchResponse {
        query: params.q,
        total: results.total,
        took_ms: results.search_time_ms,
        results: results.results.into_iter()
            .map(|r| SearchResultDto::from(r))
            .collect(),
        facets: results.facets.map(|f| FacetsDto::from(f)),
        pagination: PaginationInfo {
            limit: query.options.limit,
            offset: query.options.offset,
            total: results.total,
            pages: (results.total + query.options.limit - 1) / query.options.limit,
        },
    };
    
    Ok(Json(response))
}

/// Search parameters
#[derive(Debug, Deserialize)]
struct SearchParams {
    q: String,
    #[serde(rename = "type")]
    content_type: Option<String>,
    category: Option<String>,
    tags: Option<Vec<String>>,
    author: Option<String>,
    date_from: Option<String>,
    date_to: Option<String>,
    limit: Option<usize>,
    offset: Option<usize>,
    sort_by: Option<SortField>,
    sort_order: Option<SortOrder>,
    mode: Option<QueryMode>,
    fuzzy: Option<u8>,
    boost: Option<String>,
    highlight: Option<bool>,
    facets: Option<bool>,
    facet_fields: Option<Vec<String>>,
}

/// Search response
#[derive(Debug, Serialize)]
struct SearchResponse {
    query: String,
    total: usize,
    took_ms: u64,
    results: Vec<SearchResultDto>,
    facets: Option<FacetsDto>,
    pagination: PaginationInfo,
}

#[derive(Debug, Serialize)]
struct SearchResultDto {
    id: String,
    content_type: String,
    title: Option<String>,
    excerpt: Option<String>,
    highlights: Vec<String>,
    score: f32,
    url: Option<String>,
    metadata: serde_json::Value,
}

#[derive(Debug, Serialize)]
struct PaginationInfo {
    limit: usize,
    offset: usize,
    total: usize,
    pages: usize,
}
```

## Search Suggestions

```rust
/// Autocomplete suggestions handler
async fn suggest_handler(
    State(state): State<AppState>,
    Query(params): Query<SuggestParams>,
) -> Result<Json<SuggestResponse>, ApiError> {
    // Get suggestions
    let suggestions = state.search_engine
        .suggest(&params.q, params.limit.unwrap_or(10))
        .await
        .map_err(|e| ApiError::internal(e))?;
    
    Ok(Json(SuggestResponse {
        query: params.q,
        suggestions: suggestions.into_iter()
            .map(|s| SuggestionDto {
                text: s.text,
                score: s.score,
                payload: s.payload,
            })
            .collect(),
    }))
}

#[derive(Debug, Deserialize)]
struct SuggestParams {
    q: String,
    limit: Option<usize>,
    #[serde(rename = "type")]
    content_type: Option<String>,
}

#[derive(Debug, Serialize)]
struct SuggestResponse {
    query: String,
    suggestions: Vec<SuggestionDto>,
}

#[derive(Debug, Serialize)]
struct SuggestionDto {
    text: String,
    score: f32,
    payload: Option<serde_json::Value>,
}

/// Suggestion engine
pub struct SuggestionEngine {
    suggester: Arc<Suggester>,
    analyzer: Arc<TextAnalyzer>,
}

impl SuggestionEngine {
    /// Get suggestions for query
    pub async fn suggest(
        &self,
        query: &str,
        limit: usize,
    ) -> Result<Vec<Suggestion>> {
        // Analyze query
        let tokens = self.analyzer.tokenize(query);
        
        if tokens.is_empty() {
            return Ok(vec![]);
        }
        
        // Get suggestions for last token
        let last_token = &tokens[tokens.len() - 1];
        let suggestions = self.suggester
            .suggest(last_token, limit)
            .await?;
        
        // Build complete suggestions
        let prefix = if tokens.len() > 1 {
            tokens[..tokens.len() - 1].join(" ") + " "
        } else {
            String::new()
        };
        
        Ok(suggestions.into_iter()
            .map(|s| Suggestion {
                text: format!("{}{}", prefix, s.text),
                score: s.score,
                payload: s.payload,
            })
            .collect())
    }
}
```

## Faceted Search API

```rust
/// Facets handler
async fn facets_handler(
    State(state): State<AppState>,
    Query(params): Query<FacetParams>,
) -> Result<Json<FacetResponse>, ApiError> {
    // Build facet query
    let query = FacetQuery {
        field: params.field.clone(),
        query: params.q.clone(),
        filters: build_filters_from_facets(&params),
        limit: params.limit.unwrap_or(10),
        min_count: params.min_count.unwrap_or(1),
        sort: params.sort.unwrap_or(FacetSort::Count),
    };
    
    // Get facet values
    let facets = state.search_engine
        .get_facets(&query)
        .await
        .map_err(|e| ApiError::internal(e))?;
    
    Ok(Json(FacetResponse {
        field: params.field,
        values: facets.into_iter()
            .map(|f| FacetValueDto {
                value: f.value,
                count: f.count,
                selected: params.selected.as_ref()
                    .map(|s| s.contains(&f.value))
                    .unwrap_or(false),
            })
            .collect(),
    }))
}

#[derive(Debug, Deserialize)]
struct FacetParams {
    field: String,
    q: Option<String>,
    selected: Option<Vec<String>>,
    limit: Option<usize>,
    min_count: Option<u64>,
    sort: Option<FacetSort>,
}

#[derive(Debug, Serialize)]
struct FacetResponse {
    field: String,
    values: Vec<FacetValueDto>,
}

#[derive(Debug, Serialize)]
struct FacetValueDto {
    value: String,
    count: u64,
    selected: bool,
}

#[derive(Debug, Clone, Deserialize)]
enum FacetSort {
    Count,
    Value,
}
```

## GraphQL API

```rust
use async_graphql::{
    Object, InputObject, SimpleObject, ComplexObject,
    Context, FieldResult, ID,
};

/// GraphQL search schema
pub struct SearchSchema;

#[Object]
impl SearchSchema {
    /// Search content
    async fn search(
        &self,
        ctx: &Context<'_>,
        input: SearchInput,
    ) -> FieldResult<SearchResult> {
        let search_engine = ctx.data::<Arc<SearchEngine>>()?;
        
        let query = build_search_query(input);
        let results = search_engine.search(&query).await?;
        
        Ok(SearchResult::from(results))
    }
    
    /// Get search suggestions
    async fn suggestions(
        &self,
        ctx: &Context<'_>,
        query: String,
        limit: Option<i32>,
    ) -> FieldResult<Vec<Suggestion>> {
        let search_engine = ctx.data::<Arc<SearchEngine>>()?;
        
        let suggestions = search_engine
            .suggest(&query, limit.unwrap_or(10) as usize)
            .await?;
        
        Ok(suggestions)
    }
    
    /// Get facet values
    async fn facets(
        &self,
        ctx: &Context<'_>,
        field: String,
        query: Option<String>,
        limit: Option<i32>,
    ) -> FieldResult<FacetResult> {
        let search_engine = ctx.data::<Arc<SearchEngine>>()?;
        
        let facet_query = FacetQuery {
            field: field.clone(),
            query,
            filters: SearchFilters::default(),
            limit: limit.unwrap_or(10) as usize,
            min_count: 1,
            sort: FacetSort::Count,
        };
        
        let values = search_engine.get_facets(&facet_query).await?;
        
        Ok(FacetResult { field, values })
    }
}

#[derive(InputObject)]
struct SearchInput {
    query: String,
    content_type: Option<String>,
    filters: Option<SearchFiltersInput>,
    pagination: Option<PaginationInput>,
    options: Option<SearchOptionsInput>,
}

#[derive(InputObject)]
struct SearchFiltersInput {
    category: Option<String>,
    tags: Option<Vec<String>>,
    author: Option<String>,
    date_range: Option<DateRangeInput>,
}

#[derive(InputObject)]
struct DateRangeInput {
    from: Option<String>,
    to: Option<String>,
}

#[derive(InputObject)]
struct PaginationInput {
    limit: Option<i32>,
    offset: Option<i32>,
}

#[derive(InputObject)]
struct SearchOptionsInput {
    sort_by: Option<String>,
    sort_order: Option<String>,
    highlight: Option<bool>,
    fuzzy: Option<bool>,
}

#[derive(SimpleObject)]
struct SearchResult {
    total: i32,
    took_ms: i32,
    results: Vec<SearchHit>,
    facets: Option<SearchFacets>,
}

#[ComplexObject]
impl SearchResult {
    /// Has more results
    async fn has_more(&self, limit: i32, offset: i32) -> bool {
        self.total > offset + limit
    }
    
    /// Calculate pages
    async fn pages(&self, limit: i32) -> i32 {
        (self.total + limit - 1) / limit
    }
}

#[derive(SimpleObject)]
struct SearchHit {
    id: ID,
    content_type: String,
    title: Option<String>,
    excerpt: Option<String>,
    highlights: Vec<String>,
    score: f32,
    metadata: serde_json::Value,
}

#[derive(SimpleObject)]
struct SearchFacets {
    categories: Vec<FacetValue>,
    tags: Vec<FacetValue>,
    authors: Vec<FacetValue>,
}

#[derive(SimpleObject)]
struct FacetValue {
    value: String,
    count: i32,
}

#[derive(SimpleObject)]
struct Suggestion {
    text: String,
    score: f32,
}

#[derive(SimpleObject)]
struct FacetResult {
    field: String,
    values: Vec<FacetValue>,
}
```

## Advanced Search Features

```rust
/// Advanced search builder
pub struct AdvancedSearchBuilder {
    must: Vec<SearchClause>,
    should: Vec<SearchClause>,
    must_not: Vec<SearchClause>,
    filters: Vec<FilterClause>,
}

impl AdvancedSearchBuilder {
    pub fn new() -> Self {
        Self {
            must: Vec::new(),
            should: Vec::new(),
            must_not: Vec::new(),
            filters: Vec::new(),
        }
    }
    
    /// Add required clause
    pub fn must(mut self, field: &str, value: &str) -> Self {
        self.must.push(SearchClause::Term {
            field: field.to_string(),
            value: value.to_string(),
        });
        self
    }
    
    /// Add optional clause
    pub fn should(mut self, field: &str, value: &str) -> Self {
        self.should.push(SearchClause::Term {
            field: field.to_string(),
            value: value.to_string(),
        });
        self
    }
    
    /// Add exclusion clause
    pub fn must_not(mut self, field: &str, value: &str) -> Self {
        self.must_not.push(SearchClause::Term {
            field: field.to_string(),
            value: value.to_string(),
        });
        self
    }
    
    /// Add filter
    pub fn filter(mut self, filter: FilterClause) -> Self {
        self.filters.push(filter);
        self
    }
    
    /// Build search query
    pub fn build(self) -> SearchQuery {
        SearchQuery {
            query: self.build_query_string(),
            filters: SearchFilters {
                clauses: self.filters,
            },
            options: SearchOptions::default(),
            facets: FacetConfig::default(),
        }
    }
    
    fn build_query_string(&self) -> String {
        let mut parts = Vec::new();
        
        // Must clauses (AND)
        if !self.must.is_empty() {
            let must_str = self.must.iter()
                .map(|c| format!("({})", c.to_string()))
                .collect::<Vec<_>>()
                .join(" AND ");
            parts.push(must_str);
        }
        
        // Should clauses (OR)
        if !self.should.is_empty() {
            let should_str = self.should.iter()
                .map(|c| c.to_string())
                .collect::<Vec<_>>()
                .join(" OR ");
            parts.push(format!("({})", should_str));
        }
        
        // Must not clauses (NOT)
        if !self.must_not.is_empty() {
            let not_str = self.must_not.iter()
                .map(|c| format!("NOT {}", c.to_string()))
                .collect::<Vec<_>>()
                .join(" AND ");
            parts.push(format!("({})", not_str));
        }
        
        parts.join(" AND ")
    }
}

#[derive(Debug, Clone)]
enum SearchClause {
    Term { field: String, value: String },
    Phrase { field: String, phrase: String },
    Range { field: String, from: String, to: String },
    Wildcard { field: String, pattern: String },
}

impl ToString for SearchClause {
    fn to_string(&self) -> String {
        match self {
            SearchClause::Term { field, value } => {
                format!("{}:{}", field, value)
            }
            SearchClause::Phrase { field, phrase } => {
                format!("{}:\"{}\"", field, phrase)
            }
            SearchClause::Range { field, from, to } => {
                format!("{}:[{} TO {}]", field, from, to)
            }
            SearchClause::Wildcard { field, pattern } => {
                format!("{}:{}", field, pattern)
            }
        }
    }
}

#[derive(Debug, Clone)]
pub enum FilterClause {
    Term { field: String, value: String },
    Terms { field: String, values: Vec<String> },
    Range { field: String, gt: Option<f64>, lt: Option<f64> },
    Exists { field: String },
}
```

## Search Analytics

```rust
/// Search analytics collector
pub struct SearchAnalytics {
    storage: Arc<AnalyticsStorage>,
    config: AnalyticsConfig,
}

impl SearchAnalytics {
    /// Track search query
    pub async fn track_search(
        &self,
        query: &str,
        results_count: usize,
        took_ms: u64,
        user_id: Option<&str>,
    ) -> Result<()> {
        let event = SearchEvent {
            id: Uuid::new_v4(),
            timestamp: chrono::Utc::now(),
            query: query.to_string(),
            results_count,
            took_ms,
            user_id: user_id.map(|s| s.to_string()),
            session_id: None,
            clicked_results: Vec::new(),
        };
        
        self.storage.store_event(event).await?;
        
        Ok(())
    }
    
    /// Track result click
    pub async fn track_click(
        &self,
        search_id: Uuid,
        result_id: &str,
        position: usize,
    ) -> Result<()> {
        let click = ClickEvent {
            search_id,
            result_id: result_id.to_string(),
            position,
            timestamp: chrono::Utc::now(),
        };
        
        self.storage.store_click(click).await?;
        
        Ok(())
    }
    
    /// Get search statistics
    pub async fn get_statistics(
        &self,
        period: StatsPeriod,
    ) -> Result<SearchStatistics> {
        let stats = self.storage.get_statistics(period).await?;
        
        Ok(SearchStatistics {
            total_searches: stats.total_searches,
            unique_queries: stats.unique_queries,
            avg_results: stats.avg_results,
            avg_response_time: stats.avg_response_time,
            popular_queries: stats.popular_queries,
            no_result_queries: stats.no_result_queries,
            click_through_rate: stats.calculate_ctr(),
        })
    }
}

#[derive(Debug, Serialize)]
struct SearchStatistics {
    total_searches: u64,
    unique_queries: u64,
    avg_results: f64,
    avg_response_time: f64,
    popular_queries: Vec<PopularQuery>,
    no_result_queries: Vec<String>,
    click_through_rate: f64,
}

#[derive(Debug, Serialize)]
struct PopularQuery {
    query: String,
    count: u64,
    avg_results: f64,
    ctr: f64,
}
```

## Search API Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SearchApiConfig {
    pub max_results: usize,
    pub default_limit: usize,
    pub max_query_length: usize,
    pub enable_suggestions: bool,
    pub enable_analytics: bool,
    pub rate_limiting: RateLimitConfig,
    pub cors: CorsConfig,
}

impl Default for SearchApiConfig {
    fn default() -> Self {
        Self {
            max_results: 1000,
            default_limit: 20,
            max_query_length: 200,
            enable_suggestions: true,
            enable_analytics: true,
            rate_limiting: RateLimitConfig::default(),
            cors: CorsConfig::default(),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RateLimitConfig {
    pub enabled: bool,
    pub requests_per_minute: u32,
    pub burst_size: u32,
}

impl Default for RateLimitConfig {
    fn default() -> Self {
        Self {
            enabled: true,
            requests_per_minute: 60,
            burst_size: 10,
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CorsConfig {
    pub allowed_origins: Vec<String>,
    pub allowed_methods: Vec<String>,
    pub allowed_headers: Vec<String>,
    pub max_age: u32,
}

impl Default for CorsConfig {
    fn default() -> Self {
        Self {
            allowed_origins: vec!["*".to_string()],
            allowed_methods: vec!["GET".to_string(), "POST".to_string()],
            allowed_headers: vec!["Content-Type".to_string()],
            max_age: 3600,
        }
    }
}
```

## Summary

Diese Search API bietet:
- **RESTful API** - Standard HTTP Search Endpoints
- **GraphQL Interface** - Flexible Query Language
- **Autocomplete** - Search Suggestions
- **Faceted Search** - Category Aggregations
- **Advanced Search** - Complex Query Building
- **Analytics** - Search Usage Tracking

Complete Search API für ReedCMS.