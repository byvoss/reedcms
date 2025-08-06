# Search Indexing

## Overview

Search Indexing Implementation für ReedCMS. Automatic Content Indexing mit Delta Updates und Background Processing.

## Indexing Manager

```rust
use std::sync::Arc;
use tokio::sync::{RwLock, mpsc};
use tokio::time::{interval, Duration};

/// Search indexing manager
pub struct IndexingManager {
    search_engine: Arc<SearchEngine>,
    content_manager: Arc<ContentManager>,
    indexing_queue: Arc<IndexingQueue>,
    change_tracker: Arc<ChangeTracker>,
    config: IndexingConfig,
}

impl IndexingManager {
    pub fn new(
        search_engine: Arc<SearchEngine>,
        content_manager: Arc<ContentManager>,
        config: IndexingConfig,
    ) -> Self {
        let indexing_queue = Arc::new(IndexingQueue::new(config.queue_size));
        let change_tracker = Arc::new(ChangeTracker::new());
        
        let manager = Self {
            search_engine,
            content_manager,
            indexing_queue,
            change_tracker,
            config,
        };
        
        // Start background workers
        manager.start_workers();
        
        manager
    }
    
    /// Start background indexing workers
    fn start_workers(&self) {
        // Delta indexing worker
        self.start_delta_worker();
        
        // Full reindex scheduler
        if self.config.enable_scheduled_reindex {
            self.start_reindex_scheduler();
        }
        
        // Queue processor
        self.start_queue_processor();
    }
    
    /// Index single content item
    pub async fn index_content(&self, content_id: Uuid) -> Result<()> {
        // Add to queue for processing
        self.indexing_queue.enqueue(IndexingTask {
            task_type: TaskType::Index,
            content_id,
            priority: Priority::Normal,
            retry_count: 0,
        }).await?;
        
        Ok(())
    }
    
    /// Remove content from index
    pub async fn remove_content(&self, content_id: Uuid) -> Result<()> {
        self.indexing_queue.enqueue(IndexingTask {
            task_type: TaskType::Remove,
            content_id,
            priority: Priority::High,
            retry_count: 0,
        }).await?;
        
        Ok(())
    }
    
    /// Perform full reindex
    pub async fn full_reindex(&self) -> Result<ReindexStatus> {
        info!("Starting full search index rebuild");
        
        let mut status = ReindexStatus {
            total_items: 0,
            indexed_items: 0,
            failed_items: 0,
            start_time: chrono::Utc::now(),
            end_time: None,
            errors: Vec::new(),
        };
        
        // Get all content types
        let content_types = self.content_manager.get_content_types().await?;
        
        for content_type in content_types {
            // Index each content type
            match self.reindex_content_type(&content_type, &mut status).await {
                Ok(_) => {},
                Err(e) => {
                    error!("Failed to reindex content type {}: {}", content_type, e);
                    status.errors.push(format!("Content type {}: {}", content_type, e));
                }
            }
        }
        
        status.end_time = Some(chrono::Utc::now());
        
        info!(
            "Full reindex completed: {} indexed, {} failed out of {} total",
            status.indexed_items, status.failed_items, status.total_items
        );
        
        Ok(status)
    }
    
    /// Reindex specific content type
    async fn reindex_content_type(
        &self,
        content_type: &str,
        status: &mut ReindexStatus,
    ) -> Result<()> {
        let mut offset = 0;
        let batch_size = self.config.batch_size;
        
        loop {
            // Get batch of content
            let items = self.content_manager
                .list_by_type(content_type, offset, batch_size)
                .await?;
            
            if items.is_empty() {
                break;
            }
            
            status.total_items += items.len();
            
            // Index batch
            for item in items {
                match self.index_item(&item).await {
                    Ok(_) => status.indexed_items += 1,
                    Err(e) => {
                        error!("Failed to index item {}: {}", item.id, e);
                        status.failed_items += 1;
                    }
                }
            }
            
            offset += batch_size;
            
            // Rate limiting
            tokio::time::sleep(Duration::from_millis(self.config.batch_delay_ms)).await;
        }
        
        Ok(())
    }
    
    /// Index single item
    async fn index_item(&self, item: &ContentItem) -> Result<()> {
        // Convert to searchable content
        let searchable = self.convert_to_searchable(item).await?;
        
        // Index in search engine
        self.search_engine.index_content(&searchable).await?;
        
        // Update change tracker
        self.change_tracker.mark_indexed(item.id, item.version).await;
        
        Ok(())
    }
}

/// Indexing task
#[derive(Debug, Clone)]
struct IndexingTask {
    task_type: TaskType,
    content_id: Uuid,
    priority: Priority,
    retry_count: u32,
}

#[derive(Debug, Clone)]
enum TaskType {
    Index,
    Remove,
    Update,
}

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
enum Priority {
    Low = 0,
    Normal = 1,
    High = 2,
}
```

## Change Tracking

```rust
/// Change tracking for delta indexing
pub struct ChangeTracker {
    changes: Arc<RwLock<HashMap<Uuid, ChangeRecord>>>,
    persistence: Arc<ChangePersistence>,
}

impl ChangeTracker {
    /// Track content change
    pub async fn track_change(
        &self,
        content_id: Uuid,
        change_type: ChangeType,
        version: u32,
    ) -> Result<()> {
        let record = ChangeRecord {
            content_id,
            change_type,
            version,
            timestamp: chrono::Utc::now(),
            indexed: false,
        };
        
        // Store in memory
        self.changes.write().await.insert(content_id, record.clone());
        
        // Persist to storage
        self.persistence.save_change(&record).await?;
        
        Ok(())
    }
    
    /// Get pending changes
    pub async fn get_pending_changes(&self, limit: usize) -> Result<Vec<ChangeRecord>> {
        let changes = self.changes.read().await;
        
        let pending: Vec<_> = changes
            .values()
            .filter(|r| !r.indexed)
            .take(limit)
            .cloned()
            .collect();
        
        Ok(pending)
    }
    
    /// Mark change as indexed
    pub async fn mark_indexed(&self, content_id: Uuid, version: u32) {
        if let Some(record) = self.changes.write().await.get_mut(&content_id) {
            if record.version <= version {
                record.indexed = true;
            }
        }
    }
    
    /// Clean old changes
    pub async fn cleanup_old_changes(&self, older_than: chrono::Duration) -> Result<usize> {
        let cutoff = chrono::Utc::now() - older_than;
        let mut changes = self.changes.write().await;
        
        let old_changes: Vec<_> = changes
            .iter()
            .filter(|(_, r)| r.indexed && r.timestamp < cutoff)
            .map(|(id, _)| *id)
            .collect();
        
        let count = old_changes.len();
        
        for id in old_changes {
            changes.remove(&id);
        }
        
        // Clean from persistence
        self.persistence.cleanup_before(cutoff).await?;
        
        Ok(count)
    }
}

#[derive(Debug, Clone)]
struct ChangeRecord {
    content_id: Uuid,
    change_type: ChangeType,
    version: u32,
    timestamp: chrono::DateTime<chrono::Utc>,
    indexed: bool,
}

#[derive(Debug, Clone)]
enum ChangeType {
    Created,
    Updated,
    Deleted,
}
```

## Indexing Queue

```rust
use tokio::sync::Semaphore;
use priority_queue::PriorityQueue;

/// Priority queue for indexing tasks
pub struct IndexingQueue {
    queue: Arc<RwLock<PriorityQueue<IndexingTask, Priority>>>,
    semaphore: Arc<Semaphore>,
    max_size: usize,
}

impl IndexingQueue {
    pub fn new(max_size: usize) -> Self {
        Self {
            queue: Arc::new(RwLock::new(PriorityQueue::new())),
            semaphore: Arc::new(Semaphore::new(max_size)),
            max_size,
        }
    }
    
    /// Enqueue indexing task
    pub async fn enqueue(&self, task: IndexingTask) -> Result<()> {
        // Acquire permit
        let _permit = self.semaphore
            .acquire()
            .await
            .map_err(|_| ReedError::QueueFull)?;
        
        let priority = task.priority.clone();
        
        self.queue.write().await.push(task, priority);
        
        Ok(())
    }
    
    /// Dequeue next task
    pub async fn dequeue(&self) -> Option<IndexingTask> {
        let mut queue = self.queue.write().await;
        
        if let Some((task, _priority)) = queue.pop() {
            // Release permit when task is dequeued
            drop(queue);
            Some(task)
        } else {
            None
        }
    }
    
    /// Get queue size
    pub async fn size(&self) -> usize {
        self.queue.read().await.len()
    }
    
    /// Clear queue
    pub async fn clear(&self) {
        self.queue.write().await.clear();
    }
}
```

## Delta Indexing

```rust
/// Delta indexing processor
pub struct DeltaIndexer {
    indexing_manager: Arc<IndexingManager>,
    change_tracker: Arc<ChangeTracker>,
    config: DeltaIndexingConfig,
}

impl DeltaIndexer {
    /// Process delta changes
    pub async fn process_deltas(&self) -> Result<DeltaStats> {
        let mut stats = DeltaStats::default();
        
        // Get pending changes
        let changes = self.change_tracker
            .get_pending_changes(self.config.batch_size)
            .await?;
        
        stats.total_changes = changes.len();
        
        for change in changes {
            match self.process_change(&change).await {
                Ok(_) => {
                    stats.processed += 1;
                    self.change_tracker
                        .mark_indexed(change.content_id, change.version)
                        .await;
                }
                Err(e) => {
                    error!("Failed to process change {}: {}", change.content_id, e);
                    stats.failed += 1;
                }
            }
        }
        
        Ok(stats)
    }
    
    /// Process single change
    async fn process_change(&self, change: &ChangeRecord) -> Result<()> {
        match change.change_type {
            ChangeType::Created | ChangeType::Updated => {
                self.indexing_manager.index_content(change.content_id).await?;
            }
            ChangeType::Deleted => {
                self.indexing_manager.remove_content(change.content_id).await?;
            }
        }
        
        Ok(())
    }
    
    /// Start delta processing loop
    pub fn start_processing_loop(self: Arc<Self>) {
        tokio::spawn(async move {
            let mut ticker = interval(Duration::from_secs(self.config.interval_secs));
            
            loop {
                ticker.tick().await;
                
                match self.process_deltas().await {
                    Ok(stats) => {
                        if stats.total_changes > 0 {
                            debug!(
                                "Delta indexing: processed {} out of {} changes",
                                stats.processed, stats.total_changes
                            );
                        }
                    }
                    Err(e) => {
                        error!("Delta indexing error: {}", e);
                    }
                }
            }
        });
    }
}

#[derive(Debug, Default)]
struct DeltaStats {
    total_changes: usize,
    processed: usize,
    failed: usize,
}
```

## Content Conversion

```rust
/// Convert content to searchable format
pub struct ContentConverter {
    field_extractors: HashMap<String, Box<dyn FieldExtractor>>,
    text_processors: Vec<Box<dyn TextProcessor>>,
}

impl ContentConverter {
    pub fn new() -> Self {
        let mut converter = Self {
            field_extractors: HashMap::new(),
            text_processors: Vec::new(),
        };
        
        // Register default extractors
        converter.register_default_extractors();
        converter.register_default_processors();
        
        converter
    }
    
    /// Convert content item to searchable
    pub async fn convert(
        &self,
        item: &ContentItem,
    ) -> Result<SearchableContent> {
        let mut searchable = SearchableContent {
            id: item.id.to_string(),
            content_type: item.content_type.clone(),
            title: None,
            body: String::new(),
            metadata: HashMap::new(),
            tags: Vec::new(),
            updated_at: item.updated_at,
        };
        
        // Extract fields
        for (field_name, field_value) in &item.fields {
            if let Some(extractor) = self.field_extractors.get(field_name) {
                extractor.extract(field_value, &mut searchable)?;
            } else {
                // Default extraction
                self.default_extract(field_name, field_value, &mut searchable)?;
            }
        }
        
        // Process text
        for processor in &self.text_processors {
            searchable.body = processor.process(&searchable.body)?;
        }
        
        Ok(searchable)
    }
    
    /// Register field extractor
    pub fn register_extractor(
        &mut self,
        field_name: &str,
        extractor: Box<dyn FieldExtractor>,
    ) {
        self.field_extractors.insert(field_name.to_string(), extractor);
    }
    
    /// Default field extraction
    fn default_extract(
        &self,
        field_name: &str,
        field_value: &serde_json::Value,
        searchable: &mut SearchableContent,
    ) -> Result<()> {
        match field_value {
            serde_json::Value::String(s) => {
                // Add to body for full-text search
                searchable.body.push_str(&format!(" {} ", s));
                
                // Special fields
                match field_name {
                    "title" => searchable.title = Some(s.clone()),
                    "tags" => searchable.tags.push(s.clone()),
                    _ => {
                        searchable.metadata.insert(
                            field_name.to_string(),
                            MetadataValue::Text(s.clone()),
                        );
                    }
                }
            }
            serde_json::Value::Array(arr) => {
                for item in arr {
                    self.default_extract(field_name, item, searchable)?;
                }
            }
            serde_json::Value::Object(obj) => {
                // Flatten nested objects
                for (key, value) in obj {
                    let nested_name = format!("{}.{}", field_name, key);
                    self.default_extract(&nested_name, value, searchable)?;
                }
            }
            _ => {
                // Store as metadata
                if let Ok(text) = serde_json::to_string(field_value) {
                    searchable.metadata.insert(
                        field_name.to_string(),
                        MetadataValue::Text(text),
                    );
                }
            }
        }
        
        Ok(())
    }
}

/// Field extractor trait
trait FieldExtractor: Send + Sync {
    fn extract(
        &self,
        value: &serde_json::Value,
        searchable: &mut SearchableContent,
    ) -> Result<()>;
}

/// Text processor trait
trait TextProcessor: Send + Sync {
    fn process(&self, text: &str) -> Result<String>;
}

/// HTML strip processor
struct HtmlStripProcessor;

impl TextProcessor for HtmlStripProcessor {
    fn process(&self, text: &str) -> Result<String> {
        // Remove HTML tags
        let re = regex::Regex::new(r"<[^>]+>").unwrap();
        Ok(re.replace_all(text, " ").to_string())
    }
}

/// Markdown processor
struct MarkdownProcessor;

impl TextProcessor for MarkdownProcessor {
    fn process(&self, text: &str) -> Result<String> {
        // Convert markdown to plain text
        let parser = pulldown_cmark::Parser::new(text);
        let mut plain_text = String::new();
        
        for event in parser {
            match event {
                pulldown_cmark::Event::Text(text) => {
                    plain_text.push_str(&text);
                    plain_text.push(' ');
                }
                pulldown_cmark::Event::Code(code) => {
                    plain_text.push_str(&code);
                    plain_text.push(' ');
                }
                _ => {}
            }
        }
        
        Ok(plain_text)
    }
}
```

## Indexing Hooks

```rust
/// Indexing hooks for customization
pub struct IndexingHooks {
    pre_index: Vec<Box<dyn PreIndexHook>>,
    post_index: Vec<Box<dyn PostIndexHook>>,
}

impl IndexingHooks {
    /// Register pre-index hook
    pub fn register_pre_index(&mut self, hook: Box<dyn PreIndexHook>) {
        self.pre_index.push(hook);
    }
    
    /// Register post-index hook
    pub fn register_post_index(&mut self, hook: Box<dyn PostIndexHook>) {
        self.post_index.push(hook);
    }
    
    /// Execute pre-index hooks
    pub async fn execute_pre_index(
        &self,
        content: &mut SearchableContent,
    ) -> Result<()> {
        for hook in &self.pre_index {
            hook.before_index(content).await?;
        }
        Ok(())
    }
    
    /// Execute post-index hooks
    pub async fn execute_post_index(
        &self,
        content_id: &str,
        success: bool,
    ) -> Result<()> {
        for hook in &self.post_index {
            hook.after_index(content_id, success).await?;
        }
        Ok(())
    }
}

#[async_trait]
trait PreIndexHook: Send + Sync {
    async fn before_index(&self, content: &mut SearchableContent) -> Result<()>;
}

#[async_trait]
trait PostIndexHook: Send + Sync {
    async fn after_index(&self, content_id: &str, success: bool) -> Result<()>;
}

/// Content enrichment hook
struct ContentEnrichmentHook {
    enricher: Arc<ContentEnricher>,
}

#[async_trait]
impl PreIndexHook for ContentEnrichmentHook {
    async fn before_index(&self, content: &mut SearchableContent) -> Result<()> {
        // Enrich content with additional metadata
        if let Some(enriched) = self.enricher.enrich(content).await? {
            content.metadata.extend(enriched);
        }
        Ok(())
    }
}
```

## Indexing Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IndexingConfig {
    pub queue_size: usize,
    pub batch_size: usize,
    pub batch_delay_ms: u64,
    pub enable_scheduled_reindex: bool,
    pub reindex_schedule: String, // Cron expression
    pub delta_indexing: DeltaIndexingConfig,
    pub content_conversion: ContentConversionConfig,
}

impl Default for IndexingConfig {
    fn default() -> Self {
        Self {
            queue_size: 10000,
            batch_size: 100,
            batch_delay_ms: 100,
            enable_scheduled_reindex: false,
            reindex_schedule: "0 0 2 * * *".to_string(), // 2 AM daily
            delta_indexing: DeltaIndexingConfig::default(),
            content_conversion: ContentConversionConfig::default(),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DeltaIndexingConfig {
    pub enabled: bool,
    pub interval_secs: u64,
    pub batch_size: usize,
    pub max_retries: u32,
}

impl Default for DeltaIndexingConfig {
    fn default() -> Self {
        Self {
            enabled: true,
            interval_secs: 5,
            batch_size: 50,
            max_retries: 3,
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContentConversionConfig {
    pub strip_html: bool,
    pub process_markdown: bool,
    pub max_field_length: usize,
    pub excluded_fields: Vec<String>,
}

impl Default for ContentConversionConfig {
    fn default() -> Self {
        Self {
            strip_html: true,
            process_markdown: true,
            max_field_length: 10000,
            excluded_fields: vec!["password".to_string(), "secret".to_string()],
        }
    }
}

/// Reindex status
#[derive(Debug, Clone, Serialize)]
pub struct ReindexStatus {
    pub total_items: usize,
    pub indexed_items: usize,
    pub failed_items: usize,
    pub start_time: chrono::DateTime<chrono::Utc>,
    pub end_time: Option<chrono::DateTime<chrono::Utc>>,
    pub errors: Vec<String>,
}
```

## Summary

Dieses Search Indexing System bietet:
- **Automatic Indexing** - Content Change Detection
- **Delta Updates** - Incremental Index Updates
- **Background Processing** - Queue-based Indexing
- **Content Conversion** - Field Extraction und Text Processing
- **Indexing Hooks** - Customization Points
- **Scheduled Reindexing** - Full Index Rebuilds

Efficient und scalable Search Indexing.