# ReedCMS-09-Advanced-Features.md

## AI Core Plugin (Phase 1)

### Plugin Architecture - Graceful Degradation
```rust
// AI Plugin as Optional Dependency
pub struct ReedCMS {
    core_system: CoreSystem,           // Always available
    ai_plugin: Option<AICorePlugin>,   // Optional
}

// Core functionality never depends on AI
impl ReedCMS {
    pub async fn save_snippet(&mut self, snippet: &mut Snippet) -> Result<(), Error> {
        // Core always works
        self.core_system.validate_snippet(snippet)?;
        self.core_system.save_to_postgres(snippet).await?;
        self.core_system.update_redis_cache(snippet).await?;
        
        // AI enhancement only if enabled
        if let Some(ai) = &self.ai_plugin {
            ai.optimize_keywords_background(snippet).await.ok(); // Non-blocking
        }
        
        Ok(())
    }
}
```

**Plugin Deactivation:** System works identically without AI - fields exist but non-functional, admin UI gracefully degrades, CLI shows "AI plugin not enabled" errors.

### API Configuration
```env
# .env in project root
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_AI_API_KEY=...

AI_DEFAULT_PROVIDER=openai
AI_ENABLED=true
AI_MAX_TOKENS=2000
AI_TIMEOUT_MS=30000
```

### Registry AI Integration
```csv
# snippets.csv - AI standardmäßig verfügbar
snippet_name,field_name,field_type,ai_prompt_enabled,required,default_value
hero-banner,title,String,true,true,"Welcome"
hero-banner,ai_prompt,String,false,false,""
product-card,description,String,true,false,""
product-card,ai_prompt,String,false,false,""
```

### CLI AI Commands
```bash
# AI Provider Management
reed ai config set-provider openai
reed ai config test-connection

# Content Generation
reed snippet update $hero_banner /
  ai-prompt "Make title more engaging for B2B audience"

# UCG Keyword Optimization (triggered on content save)
reed ai optimize-keywords $snippet_id  # automatic background
```

## AI Features Implementation

### Snippet AI Prompt Field
```rust
// Auto-added to all snippets
pub struct SnippetWithAI {
    pub id: String,
    pub snippet_type: String,
    pub content: Value,
    pub ai_prompt: Option<String>,  // Always available
    pub ai_enabled: bool,           // Registry controlled
}

impl SnippetWithAI {
    pub async fn apply_ai_prompt(&mut self, ai_client: &AIClient) -> Result<(), AIError> {
        if !self.ai_enabled || self.ai_prompt.is_none() {
            return Ok(());
        }
        
        let context = serde_json::to_string(&self.content)?;
        let prompt = format!("Context: {}\nTask: {}", context, self.ai_prompt.unwrap());
        
        let enhanced = ai_client.enhance_content(&prompt).await?;
        self.content = serde_json::from_str(&enhanced)?;
        
        Ok(())
    }
}
```

### Content Generator for Editors
```rust
// Text field AI generation
pub async fn generate_text_content(
    field_value: &str,
    ai_prompt: &str,
    provider: AIProvider
) -> Result<String, AIError> {
    let prompt = if field_value.is_empty() {
        format!("Generate: {}", ai_prompt)
    } else {
        format!("Improve this text: '{}'\nTask: {}", field_value, ai_prompt)
    };
    
    match provider {
        AIProvider::OpenAI => openai_client.generate(&prompt).await,
        AIProvider::Claude => claude_client.generate(&prompt).await,
        AIProvider::Gemini => gemini_client.generate(&prompt).await,
    }
}
```

## UCG Keyword Optimization (Phase 3)

### Automatic Keyword Enhancement
```rust
// Triggered on content save
pub async fn optimize_ucg_keywords(snippet_id: &str) -> Result<(), OptimizationError> {
    let content = load_snippet_content(snippet_id).await?;
    let text_content = extract_text_fields(&content);
    
    // AI keyword extraction
    let ai_keywords = ai_client.extract_keywords(&text_content).await?;
    
    // Update search-words.csv
    let current_words = load_search_words_csv().await?;
    let enhanced_words = merge_keywords(current_words, ai_keywords, snippet_id);
    
    save_search_words_csv(enhanced_words).await?;
    
    // Update Redis search index
    update_redis_search_index(snippet_id, &ai_keywords).await?;
    
    Ok(())
}
```

### SEO & AI-GEO Widget
```rust
// Admin panel SEO suggestions
pub struct SEOWidget {
    pub current_title: String,
    pub ai_title_suggestions: Vec<String>,
    pub current_meta: String,
    pub ai_meta_suggestions: Vec<String>,
    pub geo_optimizations: Vec<GeoSuggestion>,
}

pub async fn generate_seo_suggestions(snippet: &Snippet) -> Result<SEOWidget, SEOError> {
    let content = extract_seo_content(snippet);
    
    let title_suggestions = ai_client.suggest_titles(&content).await?;
    let meta_suggestions = ai_client.suggest_meta_descriptions(&content).await?;
    let geo_suggestions = ai_client.suggest_geo_optimizations(&content).await?;
    
    Ok(SEOWidget {
        current_title: snippet.title.clone(),
        ai_title_suggestions: title_suggestions,
        current_meta: snippet.meta_description.clone(),
        ai_meta_suggestions: meta_suggestions,
        geo_optimizations: geo_suggestions,
    })
}
```

## Admin Panel Analytics (Phase 4)

### Redis-Based Error Logging
```redis
# Error tracking with priority
LPUSH errors:critical '{"type":"ai_timeout","timestamp":"2025-08-04T10:30:00Z","snippet_id":"hero1"}'
LPUSH errors:warning '{"type":"slow_query","duration_ms":150,"query":"search modern"}'
LPUSH errors:info '{"type":"cache_miss","key":"page:home:de_DE"}'

# Error priorities (AI-analyzed)
ZADD error_priorities 100 "ai_timeout"      # Critical
ZADD error_priorities 50 "slow_query"       # Warning  
ZADD error_priorities 10 "cache_miss"       # Info
```

### Error Log Analysis
```rust
pub struct ErrorAnalytics {
    redis: redis::Connection,
    ai_client: AIClient,
}

impl ErrorAnalytics {
    pub async fn analyze_errors(&mut self) -> Result<ErrorReport, AnalysisError> {
        // Get recent errors
        let critical: Vec<String> = self.redis.lrange("errors:critical", 0, 100)?;
        let warnings: Vec<String> = self.redis.lrange("errors:warning", 0, 100)?;
        
        // AI-powered error categorization
        let error_patterns = self.ai_client.analyze_error_patterns(&critical, &warnings).await?;
        
        // Update priorities
        for (error_type, priority) in error_patterns.priorities {
            self.redis.zadd("error_priorities", priority, &error_type)?;
        }
        
        Ok(ErrorReport {
            critical_count: critical.len(),
            warning_count: warnings.len(),
            top_issues: error_patterns.top_issues,
            suggested_fixes: error_patterns.suggested_fixes,
        })
    }
}
```

### System Status Analytics
```rust
// UCG & PostgreSQL health monitoring
pub struct SystemStatus {
    pub ucg_metrics: UCGMetrics,
    pub postgres_metrics: PostgresMetrics,
    pub redis_metrics: RedisMetrics,
}

pub async fn get_system_status() -> Result<SystemStatus, StatusError> {
    // UCG Analysis
    let ucg_size = redis.dbsize()?;
    let ucg_memory = redis.memory_usage()?;
    let ucg_integrity = check_ucg_integrity().await?;
    
    // PostgreSQL Analysis  
    let pg_size = postgres.query_one("SELECT pg_database_size(current_database())")?.get::<usize, i64>(0);
    let pg_connections = postgres.query_one("SELECT count(*) FROM pg_stat_activity")?.get::<usize, i64>(0);
    let pg_integrity = check_postgres_integrity().await?;
    
    // Redis Performance
    let redis_memory = redis.memory_usage()?;
    let redis_ops_per_sec = redis.info_commandstats()?;
    
    Ok(SystemStatus {
        ucg_metrics: UCGMetrics {
            total_entities: ucg_size,
            memory_usage_mb: ucg_memory / 1024 / 1024,
            integrity_score: ucg_integrity,
        },
        postgres_metrics: PostgresMetrics {
            database_size_mb: pg_size / 1024 / 1024,
            active_connections: pg_connections,
            integrity_score: pg_integrity,
        },
        redis_metrics: RedisMetrics {
            memory_usage_mb: redis_memory / 1024 / 1024,
            operations_per_second: redis_ops_per_sec,
        },
    })
}
```

## Redis Advanced Patterns

### Multi-Site Content Graph
```redis
# Site isolation with namespaces
HSET site:production:entity:snippet:hero1 type "hero-banner" data '{"title":"Welcome"}'
HSET site:staging:entity:snippet:hero1 type "hero-banner" data '{"title":"Test Welcome"}'

# Cross-site template sharing
SADD template:shared:hero-banner "site:production" "site:staging"
```

### Performance Caching
```redis
# Multi-layer cache hierarchy
SET cache:page:home:de_DE "<html>...</html>" EX 3600          # L1: Full page
HSET cache:context:home:de_DE bundle_hash "abc123" EX 1800    # L2: Context data
SET cache:snippet:hero1:rendered "<hero-banner>..." EX 900     # L3: Component

# Cache dependencies for invalidation
SADD cache:deps:snippet:hero1 "cache:page:home:de_DE" "cache:snippet:hero1:rendered"
```

### Analytics & Monitoring
```redis
# Time-series performance data
ZADD metrics:query_latency:daily 1722772800 "1.2"    # timestamp -> latency_ms
ZADD metrics:memory_usage:daily 1722772800 "245"     # timestamp -> memory_mb

# Feature usage tracking
HINCRBY features:daily ai_prompts 1
HINCRBY features:daily seo_suggestions 3
HINCRBY features:daily keyword_optimization 1
```

## CLI Advanced Commands

```bash
# AI management
reed ai provider set openai
reed ai test-connection
reed ai usage-stats --format table

# System monitoring  
reed status overview --include-ai
reed status errors --priority critical --limit 10
reed status performance --timeframe 24h

# Advanced operations
reed ai optimize-site $site_id --background
reed cache invalidate --pattern "page:*" --confirm
reed monitor health --alert-thresholds high
```

## Implementation Phases

**Phase 1 (Week 2-4):** AI Core Plugin
- API configuration & provider switching
- Basic AI prompt fields in all snippets
- CLI AI commands

**Phase 3 (Week 7):** AI-Enhanced Features  
- Automatic UCG keyword optimization
- Content generation for text fields
- SEO/GEO suggestion widgets

**Phase 4 (Week 8-10):** Admin Analytics
- Redis-based error analysis with AI prioritization  
- System status monitoring (Size, Integrity, Performance)
- Advanced Redis patterns for multi-site & caching

**Performance Targets:**
- AI API calls: <30s timeout, <2000 tokens
- UCG optimization: <100ms per content save
- Status analysis: <500ms for complete system check
- Error analysis: <1s for 100 recent errors

This AI-native approach makes ReedCMS the first CMS with built-in AI enhancement while maintaining KISS principles and sub-millisecond core performance.