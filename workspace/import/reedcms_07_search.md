# ReedCMS-07-Search.md

## Search Philosophy

**Integrated Content Discovery:** Search is not a separate system - it's seamlessly integrated into the 4-layer architecture using Redis native set operations for hyperperformant content discovery without external dependencies.

**Registry-Driven Intelligence:** Only content types marked as `is_searchable` in the Registry are indexed, preventing bloat and ensuring optimal performance.

## Search Architecture

### 4-Layer Search Integration
```
┌─────────────────────────────────────────────────────────┐
│                   CSV Layer                             │
│   Search Configuration: Searchable types + Stop words  │
│   search-words.csv defines initial word → content maps │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│              PostgreSQL Main Schema                     │
│   Source Content: All searchable snippet content       │
│   Triggers background Redis index updates              │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│              PostgreSQL UCG Schema                      │
│   Search Backup: Word mappings for recovery            │
│   Compressed, synced from Redis for consistency        │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                   Redis Layer                           │
│   Live Search Index: Word-entity associations          │
│   Sub-millisecond search operations via set intersect  │
└─────────────────────────────────────────────────────────┘
```

## Word-Entity Association System

### Progressive Filtering with Redis Sets
```redis
# Content tokenization creates word-entity relationships
# When snippet is saved, content is parsed and stored as associations

# Word taxonomy (inverted index)
SADD word:modern "snippet:homepage" "snippet:about" "snippet:tech-stack"
SADD word:rust "snippet:homepage" "snippet:architecture" "snippet:performance"  
SADD word:cms "snippet:homepage" "snippet:features" "snippet:comparison"

# Position tracking for relevance scoring
ZADD entity:snippet:homepage:words 1 "welcome" 3 "modern" 7 "rust" 9 "cms"
ZADD entity:snippet:about:words 2 "modern" 8 "architecture" 12 "rust"
```

### Progressive Search Algorithm
```redis
# Search query: "modern rust cms"
# Step 1: Get entities containing "modern"
SMEMBERS word:modern        # Returns: ["snippet:homepage", "snippet:about", "snippet:tech"]

# Step 2: Intersect with "rust" 
SINTER word:modern word:rust  # Returns: ["snippet:homepage", "snippet:about"]

# Step 3: Intersect with "cms"
SINTER word:modern word:rust word:cms  # Returns: ["snippet:homepage"]

# Each word exponentially reduces result set for optimal performance
```

## Registry-Driven Search Integration

### Searchable Type Detection
```rust
impl SearchIndexManager {
    pub async fn should_index_snippet(&self, snippet: &Snippet) -> bool {
        // Check registry for searchable flag
        self.registry.is_searchable(&snippet.snippet_type)
    }
    
    pub async fn get_searchable_fields(&self, snippet_type: &str) -> Vec<String> {
        // Only index fields marked as searchable in registry
        self.registry.get_searchable_fields(snippet_type)
            .unwrap_or_default()
    }
    
    pub async fn update_search_index(&self, snippet: &Snippet) -> Result<(), Error> {
        // Registry-gated indexing
        if !self.should_index_snippet(snippet).await {
            return Ok(()); // Skip non-searchable types
        }
        
        let searchable_fields = self.get_searchable_fields(&snippet.snippet_type).await;
        
        // Clear old word associations for this entity
        self.clear_entity_words(&snippet.id).await?;
        
        // Index only searchable fields
        for field in searchable_fields {
            if let Some(content) = snippet.data.get(&field) {
                self.index_field_content(&snippet.id, &field, content).await?;
            }
        }
        
        Ok(())
    }
}
```

## Content Tokenization

### Smart Word Extraction
```rust
pub struct ContentIndexer {
    redis: redis::Connection,
    stopwords: HashSet<String>,
    min_word_length: usize,
}

impl ContentIndexer {
    pub async fn index_snippet(&mut self, snippet: &Snippet) -> Result<(), IndexError> {
        let searchable_fields = self.registry.get_searchable_fields(&snippet.snippet_type);
        let mut all_content = String::new();
        
        // Extract content from searchable fields only
        for field in searchable_fields {
            if let Some(content) = snippet.data.get(&field).and_then(|v| v.as_str()) {
                all_content.push_str(content);
                all_content.push(' ');
            }
        }
        
        let tokens = self.tokenize_content(&all_content);
        
        // Clear old word associations for this entity
        self.clear_entity_words(&snippet.id).await?;
        
        // Create new word-entity associations
        for (position, word) in tokens.iter().enumerate() {
            // Add entity to word set
            let _: () = self.redis.sadd(
                format!("word:{}", word.to_lowercase()), 
                &snippet.id
            )?;
            
            // Track word position for relevance scoring
            let _: () = self.redis.zadd(
                format!("entity:{}:words", snippet.id),
                word.to_lowercase(),
                position as i64
            )?;
        }
        
        Ok(())
    }
    
    fn tokenize_content(&self, content: &str) -> Vec<String> {
        content
            .split_whitespace()
            .map(|word| word.trim_matches(|c: char| !c.is_alphanumeric()))
            .filter(|word| !word.is_empty() 
                && word.len() >= self.min_word_length
                && !self.stopwords.contains(*word))
            .map(|word| word.to_lowercase())
            .collect()
    }
}
```

## Search Query Engine

### Multi-Word Search with Relevance Scoring
```rust
pub struct SearchEngine {
    redis: redis::Connection,
    registry: MetaSnippetRegistry,
}

impl SearchEngine {
    pub async fn search(&mut self, query: &str) -> Result<SearchResults, SearchError> {
        let words = self.tokenize_query(query);
        
        if words.is_empty() {
            return Ok(SearchResults::empty());
        }
        
        // Progressive intersection for optimal performance
        let result_entities = self.progressive_word_intersection(&words).await?;
        
        // Score and rank results
        let scored_results = self.score_results(&result_entities, &words).await?;
        
        Ok(SearchResults::new(scored_results))
    }
    
    async fn progressive_word_intersection(&mut self, words: &[String]) -> Result<Vec<String>, SearchError> {
        if words.len() == 1 {
            // Single word - direct set lookup
            return Ok(self.redis.smembers(format!("word:{}", words[0]))?);
        }
        
        // Multi-word progressive intersection
        let temp_key = format!("search:temp:{}", uuid::Uuid::new_v4());
        
        // Start with first word
        let _: () = self.redis.sunionstore(&temp_key, &[format!("word:{}", words[0])])?;
        
        // Intersect with each additional word
        for word in &words[1..] {
            let _: () = self.redis.sinterstore(
                &temp_key, 
                &[&temp_key, &format!("word:{}", word)]
            )?;
        }
        
        let results: Vec<String> = self.redis.smembers(&temp_key)?;
        
        // Clean up temp key
        let _: () = self.redis.del(&temp_key)?;
        
        Ok(results)
    }
    
    async fn score_results(&mut self, entities: &[String], query_words: &[String]) -> Result<Vec<ScoredResult>, SearchError> {
        let mut scored = Vec::new();
        
        for entity_id in entities {
            let score = self.calculate_relevance_score(entity_id, query_words).await?;
            scored.push(ScoredResult {
                entity_id: entity_id.clone(),
                score,
            });
        }
        
        // Sort by relevance score (highest first)
        scored.sort_by(|a, b| b.score.partial_cmp(&a.score).unwrap_or(std::cmp::Ordering::Equal));
        
        Ok(scored)
    }
}
```

### Relevance Scoring Algorithm
```rust
impl SearchEngine {
    async fn calculate_relevance_score(&mut self, entity_id: &str, query_words: &[String]) -> Result<f64, SearchError> {
        let mut score = 0.0;
        
        // Get word positions for this entity
        let word_positions: Vec<(String, i64)> = self.redis.zrange_withscores(
            format!("entity:{}:words", entity_id), 0, -1
        )?;
        
        for query_word in query_words {
            if let Some((_word, position)) = word_positions.iter()
                .find(|(word, _)| word == query_word) {
                
                // Earlier positions get higher scores (title/header bonus)
                score += 1.0 / (*position as f64 + 1.0);
                
                // Title matches get significant bonus points
                if *position < 5 {  // Assuming title words are first
                    score += 2.0;
                }
            }
        }
        
        // Bonus for matching all query words
        let matched_words = query_words.iter()
            .filter(|word| word_positions.iter().any(|(w, _)| w == *word))
            .count();
            
        if matched_words == query_words.len() {
            score *= 1.5;  // 50% bonus for complete match
        }
        
        Ok(score)
    }
}
```

## Live Search Features

### Real-time Search as User Types
```rust
pub async fn live_search(&mut self, partial_query: &str) -> Result<Vec<String>, SearchError> {
    let words = self.tokenize_query(partial_query);
    
    // Even single character searches are instant
    if words.is_empty() {
        return Ok(Vec::new());
    }
    
    // Prefix matching for incomplete words
    let last_word = words.last().unwrap();
    let complete_words = &words[..words.len()-1];
    
    // Get entities matching complete words
    let mut candidates = if complete_words.is_empty() {
        // No complete words - search by prefix
        self.get_entities_by_word_prefix(last_word).await?
    } else {
        // Intersect complete words, then filter by prefix
        let complete_matches = self.progressive_word_intersection(complete_words).await?;
        self.filter_by_word_prefix(&complete_matches, last_word).await?
    };
    
    // Limit live search results for performance
    candidates.truncate(10);
    Ok(candidates)
}
```

### Faceted Search Integration
```rust
pub async fn faceted_search(&mut self, query: &SearchQuery) -> Result<FacetedResults, SearchError> {
    // Base text search
    let text_matches = if !query.text.is_empty() {
        self.search(&query.text).await?.entities
    } else {
        vec![]  // No text filter
    };
    
    // Type faceting using registry
    let type_matches = if !query.types.is_empty() {
        self.get_entities_by_types(&query.types).await?
    } else {
        vec![]  // No type filter
    };
    
    // Combine filters with intersection
    let final_results = match (text_matches.is_empty(), type_matches.is_empty()) {
        (false, false) => {
            // Both filters - intersect
            text_matches.into_iter()
                .filter(|id| type_matches.contains(id))
                .collect()
        },
        (false, true) => text_matches,   // Only text filter
        (true, false) => type_matches,   // Only type filter  
        (true, true) => vec![],          // No filters
    };
    
    // Generate facet counts
    let facets = self.generate_facet_counts(&final_results).await?;
    
    Ok(FacetedResults {
        entities: final_results,
        facets,
    })
}
```

## Background Sync Services

### Content → Redis Search Sync
```rust
pub struct ContentToSearchSyncService {
    postgres: PostgresClient,
    redis: RedisClient,
    registry: MetaSnippetRegistry,
}

impl ContentToSearchSyncService {
    pub async fn sync_changed_content(&self) -> Result<(), Error> {
        // Get recently changed content
        let changed_snippets = self.postgres
            .get_content_changed_since(Duration::from_minutes(5))
            .await?;
        
        for snippet in changed_snippets {
            // Only process searchable types (registry-gated)
            if self.registry.is_searchable(&snippet.snippet_type) {
                self.update_search_index(&snippet).await?;
            }
        }
        
        Ok(())
    }
    
    async fn update_search_index(&self, snippet: &Snippet) -> Result<(), Error> {
        // Remove old entries
        self.redis.remove_from_all_word_sets(&snippet.id).await?;
        
        // Extract searchable words from registry-defined fields
        let searchable_fields = self.registry.get_searchable_fields(&snippet.snippet_type);
        let words = self.extract_words_from_fields(snippet, &searchable_fields);
        
        // Add to new word sets
        for word in words {
            self.redis.sadd(&format!("word:{}", word), &snippet.id).await?;
        }
        
        Ok(())
    }
}
```

## CLI Integration

### Search Commands
```bash
# Full-text search
reed search "modern rust cms" /
  limit 20 /
  format json /
  include-scores

# Faceted search
reed search "welcome" /
  type "page" "hero-banner" /
  created-after "2025-01-01" /
  format table

# Live search simulation
reed search live "mod" /
  max-results 10 /
  show-progress

# Search within specific paths
reed search "team" /
  scope "content.about.*" /
  depth 2
```

### Index Management Commands
```bash
# Rebuild search index
reed search reindex /
  entity-type "snippet" /
  batch-size 100 /
  show-progress

# Index statistics
reed search stats /
  show-word-count /
  show-entity-count /
  show-orphaned-words

# Cleanup orphaned entries
reed search cleanup /
  dry-run /
  show-details
```

## Performance Characteristics

### Search Speed Benchmarks
```
Single word search:     < 1ms    (Redis SMEMBERS)
Two word intersection:  < 2ms    (Redis SINTER) 
Complex 5-word query:   < 10ms   (Multiple SINTER operations)
Live search (keystroke): < 5ms   (Prefix matching + intersection)
Faceted search:         < 15ms   (Multiple intersections + aggregation)
```

### Memory Efficiency
```redis
# Example storage requirements for 10,000 snippets with average 50 words each

# Word index: ~50,000 unique words
# Average 5 entities per word = 250,000 word-entity associations
# Redis memory: ~25MB for word index

# Position data: 10,000 entities × 50 words = 500,000 position entries  
# Redis memory: ~50MB for position scoring

# Total search index: ~75MB for 10,000 content items
# Compare: Elasticsearch would need 500MB+ for same dataset
```

### Scaling Characteristics
```redis
# Linear scaling with content volume
100 snippets    → 1MB search index     → <1ms search
1,000 snippets  → 10MB search index    → <5ms search  
10,000 snippets → 100MB search index   → <10ms search
100,000 snippets → 1GB search index    → <50ms search

# Redis native operations scale logarithmically
# No complex query planning or optimization needed
```

## Index Management

### Cleanup and Maintenance
```rust
pub struct SearchIndexManager {
    redis: redis::Connection,
}

impl SearchIndexManager {
    // Cleanup orphaned word entries
    pub async fn cleanup_word_index(&mut self) -> Result<CleanupStats, IndexError> {
        let mut removed_words = 0;
        let mut cleaned_entities = 0;
        
        // Scan all word keys
        let word_keys: Vec<String> = self.redis.scan_match("word:*")?;
        
        for word_key in word_keys {
            let entities: Vec<String> = self.redis.smembers(&word_key)?;
            let mut valid_entities = Vec::new();
            
            // Check if referenced entities still exist
            for entity_id in entities {
                if self.entity_exists(&entity_id).await? {
                    valid_entities.push(entity_id);
                } else {
                    cleaned_entities += 1;
                }
            }
            
            if valid_entities.is_empty() {
                // Remove word key if no valid entities
                let _: () = self.redis.del(&word_key)?;
                removed_words += 1;
            } else if valid_entities.len() != entities.len() {
                // Update with only valid entities
                let _: () = self.redis.del(&word_key)?;
                let _: () = self.redis.sadd(&word_key, valid_entities)?;
            }
        }
        
        Ok(CleanupStats {
            removed_words,
            cleaned_entities,
        })
    }
}
```

This search system provides enterprise-grade full-text search capabilities using only Redis and the 4-layer architecture pattern, maintaining the KISS principle while delivering hyperperformant results with registry-driven intelligence.

---

**Next: ReedCMS-08-Roadmap.md** - Learn about implementation phases, development priorities, and the strategic roadmap for ReedCMS.