# Async Processing

## Overview

Async Processing Implementation für ReedCMS. Background Jobs, Task Queues und Event-Driven Processing.

## Job Queue System

```rust
use std::sync::Arc;
use tokio::sync::{RwLock, mpsc};
use redis::aio::ConnectionManager;

/// Main job queue service
pub struct JobQueue {
    backend: Arc<dyn QueueBackend>,
    workers: Arc<WorkerPool>,
    scheduler: Arc<JobScheduler>,
    monitor: Arc<JobMonitor>,
    config: JobQueueConfig,
}

impl JobQueue {
    pub fn new(redis: ConnectionManager, config: JobQueueConfig) -> Self {
        let backend = match config.backend {
            BackendType::Redis => {
                Arc::new(RedisBackend::new(redis)) as Arc<dyn QueueBackend>
            }
            BackendType::Postgres => {
                Arc::new(PostgresBackend::new(config.db_pool.clone())) as Arc<dyn QueueBackend>
            }
        };
        
        let workers = Arc::new(WorkerPool::new(config.worker_count));
        let scheduler = Arc::new(JobScheduler::new(backend.clone()));
        let monitor = Arc::new(JobMonitor::new());
        
        Self {
            backend,
            workers,
            scheduler,
            monitor,
            config,
        }
    }
    
    /// Start job processing
    pub async fn start(&self) -> Result<()> {
        // Start workers
        self.workers.start(
            self.backend.clone(),
            self.monitor.clone(),
        ).await?;
        
        // Start scheduler
        self.scheduler.start().await?;
        
        // Start monitor
        self.monitor.start().await?;
        
        info!("Job queue started with {} workers", self.config.worker_count);
        
        Ok(())
    }
    
    /// Enqueue job
    pub async fn enqueue<J>(&self, job: J) -> Result<JobId>
    where
        J: Job + 'static,
    {
        let job_data = JobData {
            id: JobId::new(),
            job_type: J::JOB_TYPE.to_string(),
            payload: serde_json::to_value(&job)?,
            priority: job.priority(),
            max_retries: job.max_retries(),
            timeout: job.timeout(),
            created_at: chrono::Utc::now(),
            scheduled_for: chrono::Utc::now(),
            attempts: 0,
            status: JobStatus::Pending,
        };
        
        // Add to queue
        self.backend.enqueue(job_data.clone()).await?;
        
        // Track metrics
        self.monitor.job_enqueued(&job_data.job_type).await;
        
        Ok(job_data.id)
    }
    
    /// Schedule job for later
    pub async fn schedule<J>(&self, job: J, run_at: chrono::DateTime<chrono::Utc>) -> Result<JobId>
    where
        J: Job + 'static,
    {
        let mut job_data = JobData {
            id: JobId::new(),
            job_type: J::JOB_TYPE.to_string(),
            payload: serde_json::to_value(&job)?,
            priority: job.priority(),
            max_retries: job.max_retries(),
            timeout: job.timeout(),
            created_at: chrono::Utc::now(),
            scheduled_for: run_at,
            attempts: 0,
            status: JobStatus::Scheduled,
        };
        
        // Add to scheduled set
        self.backend.schedule(job_data.clone()).await?;
        
        Ok(job_data.id)
    }
    
    /// Get job status
    pub async fn get_status(&self, job_id: &JobId) -> Result<Option<JobStatus>> {
        self.backend.get_job_status(job_id).await
    }
    
    /// Cancel job
    pub async fn cancel(&self, job_id: &JobId) -> Result<bool> {
        let cancelled = self.backend.cancel_job(job_id).await?;
        
        if cancelled {
            self.monitor.job_cancelled(job_id).await;
        }
        
        Ok(cancelled)
    }
}

/// Job trait
#[async_trait]
pub trait Job: Send + Sync + Serialize + DeserializeOwned {
    const JOB_TYPE: &'static str;
    
    async fn execute(&self, context: JobContext) -> JobResult;
    
    fn priority(&self) -> Priority {
        Priority::Normal
    }
    
    fn max_retries(&self) -> u32 {
        3
    }
    
    fn timeout(&self) -> Duration {
        Duration::from_secs(300) // 5 minutes default
    }
    
    fn retry_delay(&self, attempt: u32) -> Duration {
        // Exponential backoff
        Duration::from_secs(2u64.pow(attempt))
    }
}

/// Job result
pub type JobResult = Result<JobOutput, JobError>;

#[derive(Debug, Serialize, Deserialize)]
pub struct JobOutput {
    pub data: Option<serde_json::Value>,
    pub metrics: HashMap<String, f64>,
}

#[derive(Debug, thiserror::Error)]
pub enum JobError {
    #[error("Job failed: {0}")]
    Failed(String),
    
    #[error("Job timed out")]
    Timeout,
    
    #[error("Retry later: {0}")]
    RetryLater(String),
    
    #[error("Fatal error: {0}")]
    Fatal(String),
}

/// Job data
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JobData {
    pub id: JobId,
    pub job_type: String,
    pub payload: serde_json::Value,
    pub priority: Priority,
    pub max_retries: u32,
    pub timeout: Duration,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub scheduled_for: chrono::DateTime<chrono::Utc>,
    pub attempts: u32,
    pub status: JobStatus,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct JobId(Uuid);

impl JobId {
    pub fn new() -> Self {
        Self(Uuid::new_v4())
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum JobStatus {
    Pending,
    Scheduled,
    Running,
    Completed,
    Failed,
    Cancelled,
    Retrying,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub enum Priority {
    Low = 0,
    Normal = 1,
    High = 2,
    Critical = 3,
}
```

## Worker Pool

```rust
/// Worker pool for job processing
pub struct WorkerPool {
    workers: Vec<Worker>,
    shutdown: Arc<tokio::sync::Notify>,
}

impl WorkerPool {
    pub fn new(size: usize) -> Self {
        let shutdown = Arc::new(tokio::sync::Notify::new());
        let mut workers = Vec::with_capacity(size);
        
        for id in 0..size {
            workers.push(Worker {
                id,
                shutdown: shutdown.clone(),
            });
        }
        
        Self { workers, shutdown }
    }
    
    /// Start workers
    pub async fn start(
        &self,
        backend: Arc<dyn QueueBackend>,
        monitor: Arc<JobMonitor>,
    ) -> Result<()> {
        for worker in &self.workers {
            let backend = backend.clone();
            let monitor = monitor.clone();
            let shutdown = worker.shutdown.clone();
            let worker_id = worker.id;
            
            tokio::spawn(async move {
                worker_loop(worker_id, backend, monitor, shutdown).await;
            });
        }
        
        Ok(())
    }
    
    /// Shutdown workers
    pub async fn shutdown(&self) {
        self.shutdown.notify_waiters();
    }
}

/// Worker
struct Worker {
    id: usize,
    shutdown: Arc<tokio::sync::Notify>,
}

/// Worker loop
async fn worker_loop(
    worker_id: usize,
    backend: Arc<dyn QueueBackend>,
    monitor: Arc<JobMonitor>,
    shutdown: Arc<tokio::sync::Notify>,
) {
    info!("Worker {} started", worker_id);
    
    loop {
        tokio::select! {
            _ = shutdown.notified() => {
                info!("Worker {} shutting down", worker_id);
                break;
            }
            job = backend.dequeue() => {
                match job {
                    Ok(Some(job_data)) => {
                        if let Err(e) = process_job(job_data, &backend, &monitor).await {
                            error!("Worker {} error processing job: {}", worker_id, e);
                        }
                    }
                    Ok(None) => {
                        // No job available, wait a bit
                        tokio::time::sleep(Duration::from_millis(100)).await;
                    }
                    Err(e) => {
                        error!("Worker {} error dequeuing job: {}", worker_id, e);
                        tokio::time::sleep(Duration::from_secs(1)).await;
                    }
                }
            }
        }
    }
}

/// Process single job
async fn process_job(
    mut job_data: JobData,
    backend: &Arc<dyn QueueBackend>,
    monitor: &Arc<JobMonitor>,
) -> Result<()> {
    let start = std::time::Instant::now();
    
    // Update status
    job_data.status = JobStatus::Running;
    job_data.attempts += 1;
    backend.update_job(&job_data).await?;
    monitor.job_started(&job_data.id).await;
    
    // Create job context
    let context = JobContext {
        job_id: job_data.id,
        attempt: job_data.attempts,
        created_at: job_data.created_at,
    };
    
    // Execute job with timeout
    let result = tokio::time::timeout(
        job_data.timeout,
        execute_job(&job_data.job_type, &job_data.payload, context),
    ).await;
    
    match result {
        Ok(Ok(output)) => {
            // Job succeeded
            job_data.status = JobStatus::Completed;
            backend.complete_job(&job_data, output).await?;
            monitor.job_completed(&job_data.id, start.elapsed()).await;
        }
        Ok(Err(JobError::RetryLater(reason))) => {
            // Retry job
            if job_data.attempts < job_data.max_retries {
                job_data.status = JobStatus::Retrying;
                let delay = calculate_retry_delay(&job_data);
                job_data.scheduled_for = chrono::Utc::now() + delay;
                backend.retry_job(&job_data).await?;
                monitor.job_retrying(&job_data.id, &reason).await;
            } else {
                // Max retries exceeded
                job_data.status = JobStatus::Failed;
                backend.fail_job(&job_data, &reason).await?;
                monitor.job_failed(&job_data.id, &reason).await;
            }
        }
        Ok(Err(e)) => {
            // Job failed
            job_data.status = JobStatus::Failed;
            let reason = e.to_string();
            backend.fail_job(&job_data, &reason).await?;
            monitor.job_failed(&job_data.id, &reason).await;
        }
        Err(_) => {
            // Job timed out
            job_data.status = JobStatus::Failed;
            let reason = "Job timed out";
            backend.fail_job(&job_data, reason).await?;
            monitor.job_failed(&job_data.id, reason).await;
        }
    }
    
    Ok(())
}

/// Execute job dynamically
async fn execute_job(
    job_type: &str,
    payload: &serde_json::Value,
    context: JobContext,
) -> JobResult {
    // Get job handler from registry
    let handler = JOB_REGISTRY
        .get(job_type)
        .ok_or_else(|| JobError::Fatal(format!("Unknown job type: {}", job_type)))?;
    
    handler(payload, context).await
}

/// Job context
#[derive(Debug, Clone)]
pub struct JobContext {
    pub job_id: JobId,
    pub attempt: u32,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

## Queue Backends

```rust
/// Queue backend trait
#[async_trait]
pub trait QueueBackend: Send + Sync {
    async fn enqueue(&self, job: JobData) -> Result<()>;
    async fn dequeue(&self) -> Result<Option<JobData>>;
    async fn schedule(&self, job: JobData) -> Result<()>;
    async fn update_job(&self, job: &JobData) -> Result<()>;
    async fn complete_job(&self, job: &JobData, output: JobOutput) -> Result<()>;
    async fn fail_job(&self, job: &JobData, reason: &str) -> Result<()>;
    async fn retry_job(&self, job: &JobData) -> Result<()>;
    async fn cancel_job(&self, job_id: &JobId) -> Result<bool>;
    async fn get_job_status(&self, job_id: &JobId) -> Result<Option<JobStatus>>;
}

/// Redis queue backend
pub struct RedisBackend {
    conn: ConnectionManager,
    queue_key: String,
    processing_key: String,
    scheduled_key: String,
}

impl RedisBackend {
    pub fn new(conn: ConnectionManager) -> Self {
        Self {
            conn,
            queue_key: "jobs:queue".to_string(),
            processing_key: "jobs:processing".to_string(),
            scheduled_key: "jobs:scheduled".to_string(),
        }
    }
}

#[async_trait]
impl QueueBackend for RedisBackend {
    async fn enqueue(&self, job: JobData) -> Result<()> {
        let mut conn = self.conn.clone();
        
        // Serialize job
        let job_json = serde_json::to_string(&job)?;
        
        // Add to priority queue
        let score = match job.priority {
            Priority::Low => 0,
            Priority::Normal => 1,
            Priority::High => 2,
            Priority::Critical => 3,
        };
        
        conn.zadd(&self.queue_key, &job_json, score).await?;
        
        // Store job data
        let job_key = format!("job:{}", job.id.0);
        conn.set_ex(&job_key, &job_json, 86400).await?; // 24 hour TTL
        
        Ok(())
    }
    
    async fn dequeue(&self) -> Result<Option<JobData>> {
        let mut conn = self.conn.clone();
        
        // Get highest priority job
        let result: Option<Vec<String>> = conn
            .zrevrange(&self.queue_key, 0, 0)
            .await?;
        
        if let Some(jobs) = result {
            if let Some(job_json) = jobs.first() {
                // Remove from queue
                conn.zrem(&self.queue_key, job_json).await?;
                
                // Add to processing set
                let timestamp = chrono::Utc::now().timestamp();
                conn.zadd(&self.processing_key, job_json, timestamp).await?;
                
                // Deserialize job
                let job: JobData = serde_json::from_str(job_json)?;
                
                return Ok(Some(job));
            }
        }
        
        // Check scheduled jobs
        self.process_scheduled_jobs().await?;
        
        Ok(None)
    }
    
    async fn schedule(&self, job: JobData) -> Result<()> {
        let mut conn = self.conn.clone();
        
        // Serialize job
        let job_json = serde_json::to_string(&job)?;
        
        // Add to scheduled set with timestamp as score
        let score = job.scheduled_for.timestamp();
        conn.zadd(&self.scheduled_key, &job_json, score).await?;
        
        // Store job data
        let job_key = format!("job:{}", job.id.0);
        conn.set_ex(&job_key, &job_json, 86400 * 7).await?; // 7 day TTL
        
        Ok(())
    }
    
    /// Process scheduled jobs that are ready
    async fn process_scheduled_jobs(&self) -> Result<()> {
        let mut conn = self.conn.clone();
        let now = chrono::Utc::now().timestamp();
        
        // Get jobs scheduled for now or earlier
        let jobs: Vec<String> = conn
            .zrangebyscore(&self.scheduled_key, "-inf", now)
            .await?;
        
        for job_json in jobs {
            // Remove from scheduled
            conn.zrem(&self.scheduled_key, &job_json).await?;
            
            // Add to main queue
            if let Ok(mut job) = serde_json::from_str::<JobData>(&job_json) {
                job.status = JobStatus::Pending;
                self.enqueue(job).await?;
            }
        }
        
        Ok(())
    }
}
```

## Event-Driven Processing

```rust
/// Event processor for async event handling
pub struct EventProcessor {
    handlers: Arc<RwLock<HashMap<String, Vec<Box<dyn EventHandler>>>>>,
    queue: Arc<JobQueue>,
    config: EventProcessorConfig,
}

impl EventProcessor {
    pub fn new(queue: Arc<JobQueue>, config: EventProcessorConfig) -> Self {
        Self {
            handlers: Arc::new(RwLock::new(HashMap::new())),
            queue,
            config,
        }
    }
    
    /// Register event handler
    pub async fn register_handler<H>(&self, event_type: &str, handler: H)
    where
        H: EventHandler + 'static,
    {
        let mut handlers = self.handlers.write().await;
        handlers
            .entry(event_type.to_string())
            .or_insert_with(Vec::new)
            .push(Box::new(handler));
    }
    
    /// Process event
    pub async fn process_event(&self, event: Event) -> Result<()> {
        let handlers = self.handlers.read().await;
        
        if let Some(event_handlers) = handlers.get(&event.event_type) {
            for handler in event_handlers {
                match self.config.processing_mode {
                    ProcessingMode::Sync => {
                        // Process synchronously
                        handler.handle(&event).await?;
                    }
                    ProcessingMode::Async => {
                        // Queue for async processing
                        let job = EventProcessingJob {
                            event: event.clone(),
                            handler_id: handler.id(),
                        };
                        self.queue.enqueue(job).await?;
                    }
                    ProcessingMode::Parallel => {
                        // Spawn parallel task
                        let event = event.clone();
                        let handler = handler.clone();
                        tokio::spawn(async move {
                            if let Err(e) = handler.handle(&event).await {
                                error!("Event handler error: {}", e);
                            }
                        });
                    }
                }
            }
        }
        
        Ok(())
    }
}

/// Event handler trait
#[async_trait]
pub trait EventHandler: Send + Sync {
    fn id(&self) -> String;
    async fn handle(&self, event: &Event) -> Result<()>;
    fn clone(&self) -> Box<dyn EventHandler>;
}

/// Event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Event {
    pub id: Uuid,
    pub event_type: String,
    pub source: String,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub data: serde_json::Value,
    pub metadata: HashMap<String, String>,
}

/// Event processing job
#[derive(Debug, Serialize, Deserialize)]
struct EventProcessingJob {
    event: Event,
    handler_id: String,
}

#[async_trait]
impl Job for EventProcessingJob {
    const JOB_TYPE: &'static str = "event_processing";
    
    async fn execute(&self, _context: JobContext) -> JobResult {
        // Get handler from registry
        let handler = EVENT_HANDLER_REGISTRY
            .get(&self.handler_id)
            .ok_or_else(|| JobError::Fatal(format!("Unknown handler: {}", self.handler_id)))?;
        
        // Process event
        handler.handle(&self.event).await
            .map_err(|e| JobError::Failed(e.to_string()))?;
        
        Ok(JobOutput {
            data: None,
            metrics: HashMap::new(),
        })
    }
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
enum ProcessingMode {
    Sync,
    Async,
    Parallel,
}
```

## Batch Processing

```rust
/// Batch processor for bulk operations
pub struct BatchProcessor {
    queue: Arc<JobQueue>,
    config: BatchConfig,
}

impl BatchProcessor {
    /// Process items in batches
    pub async fn process_batch<T, F, Fut>(
        &self,
        items: Vec<T>,
        processor: F,
    ) -> Result<BatchResult>
    where
        T: Send + Sync + Serialize + 'static,
        F: Fn(Vec<T>) -> Fut + Send + Sync + Clone + 'static,
        Fut: Future<Output = Result<Vec<ProcessedItem>>> + Send,
    {
        let total_items = items.len();
        let batch_size = self.config.batch_size;
        let mut batch_jobs = Vec::new();
        
        // Create batch jobs
        for chunk in items.chunks(batch_size) {
            let job = BatchJob {
                items: chunk.to_vec(),
                processor_id: "batch_processor".to_string(),
            };
            
            let job_id = self.queue.enqueue(job).await?;
            batch_jobs.push(job_id);
        }
        
        // Monitor batch progress
        let result = self.monitor_batch(batch_jobs, total_items).await?;
        
        Ok(result)
    }
    
    /// Monitor batch completion
    async fn monitor_batch(
        &self,
        job_ids: Vec<JobId>,
        total_items: usize,
    ) -> Result<BatchResult> {
        let start = std::time::Instant::now();
        let mut completed = 0;
        let mut failed = 0;
        let mut results = Vec::new();
        
        // Poll for completion
        loop {
            let mut all_done = true;
            
            for job_id in &job_ids {
                if let Some(status) = self.queue.get_status(job_id).await? {
                    match status {
                        JobStatus::Completed => {
                            completed += 1;
                        }
                        JobStatus::Failed => {
                            failed += 1;
                        }
                        JobStatus::Pending | JobStatus::Running | JobStatus::Retrying => {
                            all_done = false;
                        }
                        _ => {}
                    }
                }
            }
            
            if all_done {
                break;
            }
            
            // Check timeout
            if start.elapsed() > self.config.timeout {
                return Err(BatchError::Timeout);
            }
            
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
        
        Ok(BatchResult {
            total_items,
            processed: completed,
            failed,
            duration: start.elapsed(),
        })
    }
}

/// Batch job
#[derive(Debug, Serialize, Deserialize)]
struct BatchJob<T> {
    items: Vec<T>,
    processor_id: String,
}

#[async_trait]
impl<T> Job for BatchJob<T>
where
    T: Send + Sync + Serialize + DeserializeOwned + 'static,
{
    const JOB_TYPE: &'static str = "batch_processing";
    
    async fn execute(&self, _context: JobContext) -> JobResult {
        // Process batch items
        let mut processed = 0;
        let mut errors = 0;
        
        for item in &self.items {
            // Process item
            match process_item(item).await {
                Ok(_) => processed += 1,
                Err(_) => errors += 1,
            }
        }
        
        Ok(JobOutput {
            data: Some(json!({
                "processed": processed,
                "errors": errors,
            })),
            metrics: hashmap! {
                "items_processed" => processed as f64,
                "items_failed" => errors as f64,
            },
        })
    }
}

#[derive(Debug)]
pub struct BatchResult {
    pub total_items: usize,
    pub processed: usize,
    pub failed: usize,
    pub duration: Duration,
}
```

## Job Monitoring

```rust
/// Job monitoring and metrics
pub struct JobMonitor {
    metrics: Arc<JobMetrics>,
    alerts: Arc<AlertManager>,
}

impl JobMonitor {
    /// Start monitoring
    pub async fn start(&self) -> Result<()> {
        // Start metrics collection
        self.start_metrics_collection().await?;
        
        // Start health checks
        self.start_health_checks().await?;
        
        Ok(())
    }
    
    /// Get job statistics
    pub async fn get_stats(&self) -> JobStats {
        self.metrics.get_stats().await
    }
    
    /// Get job history
    pub async fn get_history(
        &self,
        filter: JobHistoryFilter,
    ) -> Result<Vec<JobHistoryEntry>> {
        self.metrics.get_history(filter).await
    }
}

/// Job metrics
pub struct JobMetrics {
    counters: Arc<RwLock<HashMap<String, AtomicU64>>>,
    timings: Arc<RwLock<HashMap<String, Vec<Duration>>>>,
}

impl JobMetrics {
    /// Record job metrics
    pub async fn record(&self, job_type: &str, status: JobStatus, duration: Duration) {
        // Update counters
        let counter_key = format!("{}:{:?}", job_type, status);
        let mut counters = self.counters.write().await;
        counters
            .entry(counter_key)
            .or_insert_with(|| AtomicU64::new(0))
            .fetch_add(1, Ordering::Relaxed);
        
        // Update timings
        if status == JobStatus::Completed {
            let mut timings = self.timings.write().await;
            timings
                .entry(job_type.to_string())
                .or_insert_with(Vec::new)
                .push(duration);
        }
    }
    
    /// Get statistics
    pub async fn get_stats(&self) -> JobStats {
        let counters = self.counters.read().await;
        let timings = self.timings.read().await;
        
        let mut stats = JobStats::default();
        
        // Aggregate counters
        for (key, counter) in counters.iter() {
            let count = counter.load(Ordering::Relaxed);
            let parts: Vec<&str> = key.split(':').collect();
            
            if parts.len() == 2 {
                let job_type = parts[0];
                let status = parts[1];
                
                match status {
                    "Completed" => stats.completed += count,
                    "Failed" => stats.failed += count,
                    "Running" => stats.running += count,
                    _ => {}
                }
                
                stats.by_type
                    .entry(job_type.to_string())
                    .or_insert_with(TypeStats::default)
                    .total += count;
            }
        }
        
        // Calculate average durations
        for (job_type, durations) in timings.iter() {
            if !durations.is_empty() {
                let avg = durations.iter().sum::<Duration>() / durations.len() as u32;
                stats.by_type
                    .get_mut(job_type)
                    .map(|s| s.avg_duration = Some(avg));
            }
        }
        
        stats
    }
}

#[derive(Debug, Default)]
pub struct JobStats {
    pub total: u64,
    pub completed: u64,
    pub failed: u64,
    pub running: u64,
    pub by_type: HashMap<String, TypeStats>,
}

#[derive(Debug, Default)]
pub struct TypeStats {
    pub total: u64,
    pub avg_duration: Option<Duration>,
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JobQueueConfig {
    pub backend: BackendType,
    pub worker_count: usize,
    pub max_retries: u32,
    pub default_timeout: Duration,
    pub db_pool: Option<PgPool>,
    pub redis_url: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum BackendType {
    Redis,
    Postgres,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventProcessorConfig {
    pub processing_mode: ProcessingMode,
    pub max_concurrent: usize,
    pub retry_policy: RetryPolicy,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BatchConfig {
    pub batch_size: usize,
    pub parallel_batches: usize,
    pub timeout: Duration,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RetryPolicy {
    pub max_attempts: u32,
    pub initial_delay: Duration,
    pub max_delay: Duration,
    pub exponential_base: f64,
}

impl Default for JobQueueConfig {
    fn default() -> Self {
        Self {
            backend: BackendType::Redis,
            worker_count: 4,
            max_retries: 3,
            default_timeout: Duration::from_secs(300),
            db_pool: None,
            redis_url: "redis://localhost".to_string(),
        }
    }
}
```

## Summary

Diese Async Processing Implementation bietet:
- **Job Queue** - Priority-based Job Queue
- **Worker Pool** - Scalable Worker Processing
- **Event Processing** - Event-Driven Architecture
- **Batch Processing** - Bulk Operations
- **Multiple Backends** - Redis, PostgreSQL
- **Monitoring** - Job Metrics und History

Robust Async Processing für Background Tasks.