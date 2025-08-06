# Search Engine

## Overview

Search Engine Implementation für ReedCMS. Full-text Search mit Fuzzy Matching, Faceted Search und Multi-language Support.

## Search Engine Core

```rust
use tantivy::{
    schema::*,
    Index,
    IndexWriter,
    IndexReader,
    ReloadPolicy,
    query::QueryParser,
    collector::TopDocs,
};
use std::sync::Arc;
use tokio::sync::RwLock;

/// Main search engine
pub struct SearchEngine {
    index: Arc<Index>,
    writer: Arc<RwLock<IndexWriter>>,
    reader: IndexReader,
    schema: Arc<SearchSchema>,
    config: SearchConfig,
    analyzers: Arc<AnalyzerRegistry>,
}

impl SearchEngine {
    pub fn new(config: SearchConfig) -> Result<Self> {
        // Create index directory
        std::fs::create_dir_all(&config.index_path)?;
        
        // Build schema
        let schema = SearchSchema::build()?;
        
        // Create or open index
        let index = if config.index_path.join("meta.json").exists() {
            Index::open_in_dir(&config.index_path)?
        } else {
            Index::create_in_dir(&config.index_path, schema.schema.clone())?
        };
        
        // Create writer
        let writer = index.writer(config.writer_memory_mb * 1_024_000)?;
        
        // Create reader
        let reader = index
            .reader_builder()
            .reload_policy(ReloadPolicy::OnCommit)
            .try_into()?;
        
        // Initialize analyzers
        let analyzers = Arc::new(AnalyzerRegistry::new(&config));
        
        Ok(Self {
            index: Arc::new(index),
            writer: Arc::new(RwLock::new(writer)),
            reader,
            schema: Arc::new(schema),
            config,
            analyzers,
        })
    }
    
    /// Index content
    pub async fn index_content(&self, content: &SearchableContent) -> Result<()> {
        let mut writer = self.writer.write().await;
        
        // Create document
        let doc = self.create_document(content)?;
        
        // Remove old version if exists
        if let Some(id_field) = self.schema.id_field {
            writer.delete_term(Term::from_field_text(id_field, &content.id));
        }
        
        // Add document
        writer.add_document(doc)?;
        
        // Commit if auto-commit enabled
        if self.config.auto_commit {
            writer.commit()?;
        }
        
        Ok(())
    }
    
    /// Search content
    pub async fn search(
        &self,
        query: &SearchQuery,
    ) -> Result<SearchResults> {
        let searcher = self.reader.searcher();
        
        // Parse query
        let parsed_query = self.parse_query(&query.query, &query.options)?;
        
        // Apply filters
        let filtered_query = self.apply_filters(parsed_query, &query.filters)?;
        
        // Create collector
        let collector = self.create_collector(&query.options);
        
        // Execute search
        let start = std::time::Instant::now();
        let top_docs = searcher.search(&filtered_query, &collector)?;
        let search_time = start.elapsed();
        
        // Collect results
        let mut results = Vec::new();
        let mut facets = if query.options.include_facets {
            Some(self.collect_facets(&searcher, &filtered_query, &query.facets).await?)
        } else {
            None
        };
        
        for (score, doc_address) in top_docs {
            let doc = searcher.doc(doc_address)?;
            let result = self.extract_result(&doc, score, &query.options)?;
            results.push(result);
        }
        
        // Apply highlighting if requested
        if query.options.highlight {
            self.apply_highlighting(&mut results, &query.query).await?;
        }
        
        Ok(SearchResults {
            total: searcher.num_docs() as usize,
            results,
            facets,
            search_time_ms: search_time.as_millis() as u64,
            query: query.query.clone(),
        })
    }
    
    /// Create document from content
    fn create_document(&self, content: &SearchableContent) -> Result<Document> {
        let mut doc = Document::new();
        
        // Add ID field
        doc.add_text(self.schema.id_field.unwrap(), &content.id);
        
        // Add content type
        doc.add_text(self.schema.type_field.unwrap(), &content.content_type);
        
        // Add title with boost
        if let Some(title) = &content.title {
            doc.add_text(self.schema.title_field.unwrap(), title);
        }
        
        // Add body
        doc.add_text(self.schema.body_field.unwrap(), &content.body);
        
        // Add metadata fields
        for (key, value) in &content.metadata {
            if let Some(field) = self.schema.get_field(key) {
                match value {
                    MetadataValue::Text(text) => doc.add_text(field, text),
                    MetadataValue::Number(num) => doc.add_i64(field, *num),
                    MetadataValue::Date(date) => doc.add_date(field, date),
                    MetadataValue::Tags(tags) => {
                        for tag in tags {
                            doc.add_text(field, tag);
                        }
                    }
                }
            }
        }
        
        // Add timestamp
        doc.add_date(
            self.schema.timestamp_field.unwrap(),
            &content.updated_at,
        );
        
        // Add language if detected
        if let Some(lang) = self.detect_language(&content.body) {
            doc.add_text(self.schema.language_field.unwrap(), &lang);
        }
        
        Ok(doc)
    }
}

/// Searchable content
#[derive(Debug, Clone)]
pub struct SearchableContent {
    pub id: String,
    pub content_type: String,
    pub title: Option<String>,
    pub body: String,
    pub metadata: HashMap<String, MetadataValue>,
    pub tags: Vec<String>,
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Clone)]
pub enum MetadataValue {
    Text(String),
    Number(i64),
    Date(chrono::DateTime<chrono::Utc>),
    Tags(Vec<String>),
}
```

## Search Schema

```rust
/// Search schema definition
pub struct SearchSchema {
    pub schema: Schema,
    pub id_field: Option<Field>,
    pub type_field: Option<Field>,
    pub title_field: Option<Field>,
    pub body_field: Option<Field>,
    pub timestamp_field: Option<Field>,
    pub language_field: Option<Field>,
    pub metadata_fields: HashMap<String, Field>,
}

impl SearchSchema {
    /// Build search schema
    pub fn build() -> Result<Self> {
        let mut schema_builder = Schema::builder();
        
        // Core fields
        let id_field = schema_builder.add_text_field(
            "id",
            TextOptions::default()
                .set_stored()
                .set_indexing_options(
                    TextFieldIndexing::default()
                        .set_tokenizer("raw")
                        .set_index_option(IndexRecordOption::Basic)
                ),
        );
        
        let type_field = schema_builder.add_text_field(
            "content_type",
            TextOptions::default()
                .set_stored()
                .set_indexing_options(
                    TextFieldIndexing::default()
                        .set_tokenizer("raw")
                        .set_index_option(IndexRecordOption::WithFreqsAndPositions)
                ),
        );
        
        let title_field = schema_builder.add_text_field(
            "title",
            TextOptions::default()
                .set_stored()
                .set_indexing_options(
                    TextFieldIndexing::default()
                        .set_tokenizer("default")
                        .set_index_option(IndexRecordOption::WithFreqsAndPositions)
                ),
        );
        
        let body_field = schema_builder.add_text_field(
            "body",
            TextOptions::default()
                .set_stored()
                .set_indexing_options(
                    TextFieldIndexing::default()
                        .set_tokenizer("default")
                        .set_index_option(IndexRecordOption::WithFreqsAndPositions)
                ),
        );
        
        let timestamp_field = schema_builder.add_date_field(
            "timestamp",
            DateOptions::default()
                .set_stored()
                .set_indexed()
                .set_fast(Cardinality::SingleValue),
        );
        
        let language_field = schema_builder.add_text_field(
            "language",
            TextOptions::default()
                .set_stored()
                .set_indexing_options(
                    TextFieldIndexing::default()
                        .set_tokenizer("raw")
                        .set_index_option(IndexRecordOption::Basic)
                ),
        );
        
        // Dynamic metadata fields
        let mut metadata_fields = HashMap::new();
        
        // Add common metadata fields
        metadata_fields.insert(
            "author".to_string(),
            schema_builder.add_text_field(
                "author",
                TextOptions::default()
                    .set_stored()
                    .set_indexing_options(
                        TextFieldIndexing::default()
                            .set_tokenizer("raw")
                            .set_index_option(IndexRecordOption::Basic)
                    ),
            ),
        );
        
        metadata_fields.insert(
            "category".to_string(),
            schema_builder.add_text_field(
                "category",
                TextOptions::default()
                    .set_stored()
                    .set_indexing_options(
                        TextFieldIndexing::default()
                            .set_tokenizer("raw")
                            .set_index_option(IndexRecordOption::Basic)
                    ),
            ),
        );
        
        metadata_fields.insert(
            "tags".to_string(),
            schema_builder.add_text_field(
                "tags",
                TextOptions::default()
                    .set_stored()
                    .set_indexing_options(
                        TextFieldIndexing::default()
                            .set_tokenizer("raw")
                            .set_index_option(IndexRecordOption::Basic)
                    ),
            ),
        );
        
        let schema = schema_builder.build();
        
        Ok(Self {
            schema,
            id_field: Some(id_field),
            type_field: Some(type_field),
            title_field: Some(title_field),
            body_field: Some(body_field),
            timestamp_field: Some(timestamp_field),
            language_field: Some(language_field),
            metadata_fields,
        })
    }
    
    /// Get field by name
    pub fn get_field(&self, name: &str) -> Option<Field> {
        self.metadata_fields.get(name).copied()
    }
}
```

## Query Parser

```rust
use tantivy::query::*;

/// Advanced query parser
pub struct AdvancedQueryParser {
    schema: Arc<SearchSchema>,
    analyzers: Arc<AnalyzerRegistry>,
}

impl AdvancedQueryParser {
    /// Parse search query
    pub fn parse(
        &self,
        query_str: &str,
        options: &SearchOptions,
    ) -> Result<Box<dyn Query>> {
        // Handle empty query
        if query_str.trim().is_empty() {
            return Ok(Box::new(AllQuery));
        }
        
        // Parse query based on mode
        let query = match options.query_mode {
            QueryMode::Simple => self.parse_simple(query_str)?,
            QueryMode::Advanced => self.parse_advanced(query_str)?,
            QueryMode::Fuzzy => self.parse_fuzzy(query_str, options.fuzzy_distance)?,
            QueryMode::Phrase => self.parse_phrase(query_str)?,
        };
        
        // Apply field boosting
        let boosted_query = self.apply_field_boost(query, &options.field_boosts)?;
        
        Ok(boosted_query)
    }
    
    /// Parse simple query
    fn parse_simple(&self, query_str: &str) -> Result<Box<dyn Query>> {
        let mut sub_queries: Vec<Box<dyn Query>> = Vec::new();
        
        // Search in title with higher boost
        if let Some(title_field) = self.schema.title_field {
            let title_query = TermQuery::new(
                Term::from_field_text(title_field, query_str),
                IndexRecordOption::WithFreqsAndPositions,
            );
            sub_queries.push(Box::new(BoostQuery::new(
                Box::new(title_query),
                2.0,
            )));
        }
        
        // Search in body
        if let Some(body_field) = self.schema.body_field {
            let body_query = TermQuery::new(
                Term::from_field_text(body_field, query_str),
                IndexRecordOption::WithFreqsAndPositions,
            );
            sub_queries.push(Box::new(body_query));
        }
        
        // Combine with OR
        Ok(Box::new(BooleanQuery::union(sub_queries)))
    }
    
    /// Parse fuzzy query
    fn parse_fuzzy(
        &self,
        query_str: &str,
        max_distance: u8,
    ) -> Result<Box<dyn Query>> {
        let mut sub_queries: Vec<Box<dyn Query>> = Vec::new();
        
        // Fuzzy search in title
        if let Some(title_field) = self.schema.title_field {
            let fuzzy_query = FuzzyTermQuery::new(
                Term::from_field_text(title_field, query_str),
                max_distance,
                true, // prefix
            );
            sub_queries.push(Box::new(BoostQuery::new(
                Box::new(fuzzy_query),
                1.5,
            )));
        }
        
        // Fuzzy search in body
        if let Some(body_field) = self.schema.body_field {
            let fuzzy_query = FuzzyTermQuery::new(
                Term::from_field_text(body_field, query_str),
                max_distance,
                true,
            );
            sub_queries.push(Box::new(fuzzy_query));
        }
        
        Ok(Box::new(BooleanQuery::union(sub_queries)))
    }
    
    /// Parse phrase query
    fn parse_phrase(&self, query_str: &str) -> Result<Box<dyn Query>> {
        let mut sub_queries: Vec<Box<dyn Query>> = Vec::new();
        
        // Phrase search in body
        if let Some(body_field) = self.schema.body_field {
            let terms: Vec<Term> = query_str
                .split_whitespace()
                .map(|word| Term::from_field_text(body_field, word))
                .collect();
            
            if !terms.is_empty() {
                let phrase_query = PhraseQuery::new(terms);
                sub_queries.push(Box::new(phrase_query));
            }
        }
        
        if sub_queries.is_empty() {
            return Ok(Box::new(AllQuery));
        }
        
        Ok(Box::new(BooleanQuery::union(sub_queries)))
    }
}
```

## Faceted Search

```rust
/// Facet collector for aggregations
pub struct FacetCollector {
    schema: Arc<SearchSchema>,
    facet_fields: Vec<FacetField>,
}

impl FacetCollector {
    /// Collect facets from search results
    pub async fn collect(
        &self,
        searcher: &Searcher,
        query: &dyn Query,
        facet_config: &FacetConfig,
    ) -> Result<FacetResults> {
        let mut results = FacetResults::new();
        
        for facet_field in &facet_config.fields {
            match facet_field.facet_type {
                FacetType::Terms => {
                    let terms_facet = self.collect_terms_facet(
                        searcher,
                        query,
                        &facet_field,
                    ).await?;
                    results.add_facet(facet_field.name.clone(), terms_facet);
                }
                FacetType::Range => {
                    let range_facet = self.collect_range_facet(
                        searcher,
                        query,
                        &facet_field,
                    ).await?;
                    results.add_facet(facet_field.name.clone(), range_facet);
                }
                FacetType::Date => {
                    let date_facet = self.collect_date_facet(
                        searcher,
                        query,
                        &facet_field,
                    ).await?;
                    results.add_facet(facet_field.name.clone(), date_facet);
                }
            }
        }
        
        Ok(results)
    }
    
    /// Collect terms facet
    async fn collect_terms_facet(
        &self,
        searcher: &Searcher,
        query: &dyn Query,
        facet_field: &FacetField,
    ) -> Result<Facet> {
        let field = self.schema.get_field(&facet_field.field)
            .ok_or_else(|| ReedError::FieldNotFound(facet_field.field.clone()))?;
        
        // Create facet collector
        let mut facet_counts: HashMap<String, u64> = HashMap::new();
        
        // Collect top docs
        let collector = TopDocs::with_limit(10000);
        let top_docs = searcher.search(query, &collector)?;
        
        // Count facet values
        for (_score, doc_address) in top_docs {
            let doc = searcher.doc(doc_address)?;
            
            if let Some(values) = doc.get_all(field) {
                for value in values {
                    if let Some(text) = value.as_text() {
                        *facet_counts.entry(text.to_string()).or_insert(0) += 1;
                    }
                }
            }
        }
        
        // Sort by count
        let mut facet_values: Vec<_> = facet_counts
            .into_iter()
            .map(|(value, count)| FacetValue {
                value,
                count,
                query: None,
            })
            .collect();
        
        facet_values.sort_by(|a, b| b.count.cmp(&a.count));
        
        // Limit to top N
        facet_values.truncate(facet_field.limit.unwrap_or(10));
        
        Ok(Facet::Terms(TermsFacet {
            field: facet_field.field.clone(),
            values: facet_values,
        }))
    }
}

#[derive(Debug, Clone)]
pub struct FacetResults {
    facets: HashMap<String, Facet>,
}

#[derive(Debug, Clone)]
pub enum Facet {
    Terms(TermsFacet),
    Range(RangeFacet),
    Date(DateFacet),
}

#[derive(Debug, Clone)]
pub struct TermsFacet {
    pub field: String,
    pub values: Vec<FacetValue>,
}

#[derive(Debug, Clone)]
pub struct FacetValue {
    pub value: String,
    pub count: u64,
    pub query: Option<String>,
}
```

## Highlighting

```rust
/// Search result highlighting
pub struct Highlighter {
    pre_tag: String,
    post_tag: String,
    fragment_size: usize,
    max_fragments: usize,
}

impl Highlighter {
    pub fn new(config: HighlightConfig) -> Self {
        Self {
            pre_tag: config.pre_tag.unwrap_or_else(|| "<mark>".to_string()),
            post_tag: config.post_tag.unwrap_or_else(|| "</mark>".to_string()),
            fragment_size: config.fragment_size.unwrap_or(150),
            max_fragments: config.max_fragments.unwrap_or(3),
        }
    }
    
    /// Highlight search results
    pub async fn highlight(
        &self,
        results: &mut [SearchResult],
        query: &str,
    ) -> Result<()> {
        let query_terms = self.extract_query_terms(query);
        
        for result in results {
            // Highlight title
            if let Some(ref mut title) = result.title {
                *title = self.highlight_text(title, &query_terms);
            }
            
            // Create highlighted fragments from body
            if let Some(ref body) = result.body {
                result.highlights = self.create_fragments(body, &query_terms);
            }
        }
        
        Ok(())
    }
    
    /// Highlight text with query terms
    fn highlight_text(&self, text: &str, terms: &[String]) -> String {
        let mut highlighted = text.to_string();
        
        for term in terms {
            let regex = regex::RegexBuilder::new(&format!(r"\b{}\b", regex::escape(term)))
                .case_insensitive(true)
                .build()
                .unwrap();
            
            highlighted = regex.replace_all(
                &highlighted,
                format!("{}{}{}", self.pre_tag, term, self.post_tag),
            ).to_string();
        }
        
        highlighted
    }
    
    /// Create highlighted fragments
    fn create_fragments(&self, text: &str, terms: &[String]) -> Vec<String> {
        let mut fragments = Vec::new();
        let text_lower = text.to_lowercase();
        
        // Find positions of all terms
        let mut positions = Vec::new();
        for term in terms {
            let term_lower = term.to_lowercase();
            let mut start = 0;
            
            while let Some(pos) = text_lower[start..].find(&term_lower) {
                let absolute_pos = start + pos;
                positions.push((absolute_pos, term));
                start = absolute_pos + term.len();
            }
        }
        
        // Sort positions
        positions.sort_by_key(|(pos, _)| *pos);
        
        // Create fragments around positions
        for (pos, term) in positions.iter().take(self.max_fragments) {
            let fragment = self.extract_fragment(text, *pos, term.len());
            let highlighted = self.highlight_text(&fragment, terms);
            
            if !fragments.contains(&highlighted) {
                fragments.push(highlighted);
            }
        }
        
        fragments
    }
    
    /// Extract fragment around position
    fn extract_fragment(&self, text: &str, position: usize, term_len: usize) -> String {
        let start = position.saturating_sub(self.fragment_size / 2);
        let end = (position + term_len + self.fragment_size / 2).min(text.len());
        
        let mut fragment = String::new();
        
        // Add ellipsis if not at start
        if start > 0 {
            fragment.push_str("...");
        }
        
        // Extract fragment
        fragment.push_str(&text[start..end]);
        
        // Add ellipsis if not at end
        if end < text.len() {
            fragment.push_str("...");
        }
        
        fragment
    }
}
```

## Multi-language Support

```rust
/// Language analyzer registry
pub struct AnalyzerRegistry {
    analyzers: HashMap<String, Box<dyn TextAnalyzer>>,
    language_detector: LanguageDetector,
}

impl AnalyzerRegistry {
    pub fn new(config: &SearchConfig) -> Self {
        let mut analyzers = HashMap::new();
        
        // Register language-specific analyzers
        analyzers.insert("en".to_string(), Box::new(EnglishAnalyzer::new()));
        analyzers.insert("de".to_string(), Box::new(GermanAnalyzer::new()));
        analyzers.insert("fr".to_string(), Box::new(FrenchAnalyzer::new()));
        analyzers.insert("es".to_string(), Box::new(SpanishAnalyzer::new()));
        
        // Default analyzer
        analyzers.insert("default".to_string(), Box::new(StandardAnalyzer::new()));
        
        Self {
            analyzers,
            language_detector: LanguageDetector::new(),
        }
    }
    
    /// Get analyzer for language
    pub fn get_analyzer(&self, language: &str) -> &dyn TextAnalyzer {
        self.analyzers
            .get(language)
            .or_else(|| self.analyzers.get("default"))
            .map(|a| a.as_ref())
            .unwrap()
    }
}

/// Text analyzer trait
trait TextAnalyzer: Send + Sync {
    fn tokenize(&self, text: &str) -> Vec<Token>;
    fn normalize(&self, token: &str) -> String;
    fn stem(&self, token: &str) -> String;
}

/// English analyzer with stemming
struct EnglishAnalyzer {
    stemmer: rust_stemmers::Stemmer,
    stop_words: HashSet<String>,
}

impl EnglishAnalyzer {
    fn new() -> Self {
        Self {
            stemmer: rust_stemmers::Stemmer::create(rust_stemmers::Algorithm::English),
            stop_words: Self::load_stop_words(),
        }
    }
    
    fn load_stop_words() -> HashSet<String> {
        let words = vec![
            "a", "an", "and", "are", "as", "at", "be", "by", "for",
            "from", "has", "he", "in", "is", "it", "its", "of", "on",
            "that", "the", "to", "was", "will", "with",
        ];
        
        words.into_iter().map(String::from).collect()
    }
}

impl TextAnalyzer for EnglishAnalyzer {
    fn tokenize(&self, text: &str) -> Vec<Token> {
        let mut tokens = Vec::new();
        let mut current_token = String::new();
        let mut position = 0;
        
        for ch in text.chars() {
            if ch.is_alphanumeric() {
                current_token.push(ch.to_lowercase().next().unwrap());
            } else if !current_token.is_empty() {
                if !self.stop_words.contains(&current_token) {
                    tokens.push(Token {
                        text: current_token.clone(),
                        position,
                        offset: position - current_token.len(),
                    });
                }
                current_token.clear();
            }
            position += 1;
        }
        
        if !current_token.is_empty() && !self.stop_words.contains(&current_token) {
            tokens.push(Token {
                text: current_token,
                position,
                offset: position - current_token.len(),
            });
        }
        
        tokens
    }
    
    fn normalize(&self, token: &str) -> String {
        token.to_lowercase()
    }
    
    fn stem(&self, token: &str) -> String {
        self.stemmer.stem(token).to_string()
    }
}

#[derive(Debug, Clone)]
struct Token {
    text: String,
    position: usize,
    offset: usize,
}
```

## Search Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SearchConfig {
    pub index_path: PathBuf,
    pub writer_memory_mb: usize,
    pub auto_commit: bool,
    pub commit_interval_secs: u64,
    pub default_limit: usize,
    pub max_limit: usize,
    pub enable_fuzzy: bool,
    pub fuzzy_distance: u8,
    pub enable_highlighting: bool,
    pub enable_facets: bool,
    pub languages: Vec<String>,
}

impl Default for SearchConfig {
    fn default() -> Self {
        Self {
            index_path: PathBuf::from("data/search_index"),
            writer_memory_mb: 50,
            auto_commit: true,
            commit_interval_secs: 60,
            default_limit: 20,
            max_limit: 100,
            enable_fuzzy: true,
            fuzzy_distance: 2,
            enable_highlighting: true,
            enable_facets: true,
            languages: vec!["en".to_string(), "de".to_string()],
        }
    }
}

#[derive(Debug, Clone)]
pub struct SearchQuery {
    pub query: String,
    pub filters: SearchFilters,
    pub options: SearchOptions,
    pub facets: FacetConfig,
}

#[derive(Debug, Clone)]
pub struct SearchOptions {
    pub limit: usize,
    pub offset: usize,
    pub sort_by: SortField,
    pub sort_order: SortOrder,
    pub query_mode: QueryMode,
    pub fuzzy_distance: u8,
    pub field_boosts: HashMap<String, f32>,
    pub highlight: bool,
    pub include_facets: bool,
}

#[derive(Debug, Clone)]
pub enum QueryMode {
    Simple,
    Advanced,
    Fuzzy,
    Phrase,
}

#[derive(Debug, Clone)]
pub struct SearchResults {
    pub total: usize,
    pub results: Vec<SearchResult>,
    pub facets: Option<FacetResults>,
    pub search_time_ms: u64,
    pub query: String,
}

#[derive(Debug, Clone)]
pub struct SearchResult {
    pub id: String,
    pub content_type: String,
    pub title: Option<String>,
    pub body: Option<String>,
    pub highlights: Vec<String>,
    pub score: f32,
    pub metadata: HashMap<String, String>,
}
```

## Summary

Diese Search Engine bietet:
- **Full-text Search** - Tantivy-based High-performance Search
- **Fuzzy Matching** - Typo-tolerant Search
- **Faceted Search** - Category Aggregations
- **Multi-language** - Language-specific Analyzers
- **Highlighting** - Search Result Highlighting
- **Advanced Queries** - Boolean, Phrase, Range Queries

Powerful Search Capabilities für ReedCMS.