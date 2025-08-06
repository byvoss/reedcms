# Webhooks

## Overview

Webhooks Implementation für ReedCMS. Event-Driven Integration mit External Systems.

## Webhook Manager

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use reqwest::Client;

/// Main webhook service
pub struct WebhookManager {
    registry: Arc<WebhookRegistry>,
    dispatcher: Arc<WebhookDispatcher>,
    processor: Arc<EventProcessor>,
    retry_manager: Arc<RetryManager>,
    config: WebhookConfig,
}

impl WebhookManager {
    pub fn new(config: WebhookConfig) -> Self {
        let client = Client::builder()
            .timeout(config.request_timeout)
            .build()
            .expect("Failed to create HTTP client");
        
        let registry = Arc::new(WebhookRegistry::new());
        let dispatcher = Arc::new(WebhookDispatcher::new(client));
        let processor = Arc::new(EventProcessor::new());
        let retry_manager = Arc::new(RetryManager::new(config.retry.clone()));
        
        Self {
            registry,
            dispatcher,
            processor,
            retry_manager,
            config,
        }
    }
    
    /// Register webhook
    pub async fn register_webhook(&self, request: RegisterWebhookRequest) -> Result<Webhook> {
        // Validate webhook
        self.validate_webhook(&request)?;
        
        // Create webhook
        let webhook = Webhook {
            id: WebhookId::new(),
            name: request.name,
            url: request.url,
            events: request.events,
            headers: request.headers.unwrap_or_default(),
            secret: request.secret.or_else(|| Some(generate_secret())),
            active: true,
            created_at: chrono::Utc::now(),
            metadata: request.metadata,
        };
        
        // Test webhook if requested
        if request.test_on_create {
            self.test_webhook(&webhook).await?;
        }
        
        // Register
        self.registry.register(webhook.clone()).await?;
        
        Ok(webhook)
    }
    
    /// Trigger webhook for event
    pub async fn trigger(&self, event: Event) -> Result<()> {
        // Get webhooks for event
        let webhooks = self.registry.get_webhooks_for_event(&event.event_type).await?;
        
        if webhooks.is_empty() {
            return Ok(());
        }
        
        // Process event
        let payload = self.processor.process_event(&event).await?;
        
        // Dispatch to webhooks
        for webhook in webhooks {
            if webhook.active {
                let delivery = WebhookDelivery {
                    id: DeliveryId::new(),
                    webhook_id: webhook.id.clone(),
                    event_id: event.id.clone(),
                    payload: payload.clone(),
                    created_at: chrono::Utc::now(),
                };
                
                // Dispatch asynchronously
                let dispatcher = self.dispatcher.clone();
                let retry_manager = self.retry_manager.clone();
                let webhook = webhook.clone();
                
                tokio::spawn(async move {
                    if let Err(e) = dispatcher.dispatch(&webhook, &delivery).await {
                        error!("Webhook dispatch failed: {}", e);
                        // Queue for retry
                        let _ = retry_manager.queue_retry(&webhook, &delivery).await;
                    }
                });
            }
        }
        
        Ok(())
    }
    
    /// Get webhook by ID
    pub async fn get_webhook(&self, webhook_id: &WebhookId) -> Result<Option<Webhook>> {
        self.registry.get(webhook_id).await
    }
    
    /// Update webhook
    pub async fn update_webhook(
        &self,
        webhook_id: &WebhookId,
        update: UpdateWebhookRequest,
    ) -> Result<Webhook> {
        let mut webhook = self.get_webhook(webhook_id)
            .await?
            .ok_or_else(|| WebhookError::NotFound(webhook_id.clone()))?;
        
        // Apply updates
        if let Some(name) = update.name {
            webhook.name = name;
        }
        
        if let Some(url) = update.url {
            webhook.url = url;
        }
        
        if let Some(events) = update.events {
            webhook.events = events;
        }
        
        if let Some(headers) = update.headers {
            webhook.headers = headers;
        }
        
        if let Some(active) = update.active {
            webhook.active = active;
        }
        
        // Save changes
        self.registry.update(webhook.clone()).await?;
        
        Ok(webhook)
    }
    
    /// Delete webhook
    pub async fn delete_webhook(&self, webhook_id: &WebhookId) -> Result<()> {
        self.registry.delete(webhook_id).await
    }
    
    /// Test webhook
    pub async fn test_webhook(&self, webhook: &Webhook) -> Result<TestResult> {
        let test_event = Event {
            id: EventId::new(),
            event_type: "webhook.test".to_string(),
            source: "system".to_string(),
            timestamp: chrono::Utc::now(),
            data: json!({
                "test": true,
                "webhook_id": webhook.id,
                "timestamp": chrono::Utc::now(),
            }),
        };
        
        let payload = self.processor.process_event(&test_event).await?;
        
        let delivery = WebhookDelivery {
            id: DeliveryId::new(),
            webhook_id: webhook.id.clone(),
            event_id: test_event.id,
            payload,
            created_at: chrono::Utc::now(),
        };
        
        let result = self.dispatcher.dispatch(webhook, &delivery).await?;
        
        Ok(TestResult {
            success: result.success,
            status_code: result.status_code,
            response_time: result.response_time,
            error: result.error,
        })
    }
}

/// Webhook model
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Webhook {
    pub id: WebhookId,
    pub name: String,
    pub url: String,
    pub events: Vec<String>,
    pub headers: HashMap<String, String>,
    pub secret: Option<String>,
    pub active: bool,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub metadata: HashMap<String, String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct WebhookId(Uuid);

impl WebhookId {
    pub fn new() -> Self {
        Self(Uuid::new_v4())
    }
}
```

## Webhook Dispatcher

```rust
/// Webhook delivery dispatcher
pub struct WebhookDispatcher {
    client: Client,
    signer: Arc<WebhookSigner>,
}

impl WebhookDispatcher {
    pub fn new(client: Client) -> Self {
        Self {
            client,
            signer: Arc::new(WebhookSigner::new()),
        }
    }
    
    /// Dispatch webhook
    pub async fn dispatch(
        &self,
        webhook: &Webhook,
        delivery: &WebhookDelivery,
    ) -> Result<DeliveryResult> {
        let start = std::time::Instant::now();
        
        // Build request
        let mut request = self.client
            .post(&webhook.url)
            .json(&delivery.payload)
            .timeout(Duration::from_secs(30));
        
        // Add headers
        request = request.header("X-Webhook-ID", webhook.id.0.to_string());
        request = request.header("X-Delivery-ID", delivery.id.0.to_string());
        request = request.header("X-Event-Type", &delivery.payload.event_type);
        request = request.header("X-Timestamp", delivery.created_at.to_rfc3339());
        
        // Add custom headers
        for (key, value) in &webhook.headers {
            request = request.header(key, value);
        }
        
        // Add signature if secret configured
        if let Some(secret) = &webhook.secret {
            let signature = self.signer.sign(&delivery.payload, secret)?;
            request = request.header("X-Webhook-Signature", signature);
        }
        
        // Send request
        let response = match request.send().await {
            Ok(resp) => resp,
            Err(e) => {
                return Ok(DeliveryResult {
                    success: false,
                    status_code: None,
                    response_time: start.elapsed(),
                    error: Some(e.to_string()),
                    response_body: None,
                });
            }
        };
        
        let status_code = response.status().as_u16();
        let response_body = response.text().await.ok();
        
        Ok(DeliveryResult {
            success: status_code >= 200 && status_code < 300,
            status_code: Some(status_code),
            response_time: start.elapsed(),
            error: None,
            response_body,
        })
    }
}

/// Webhook signature generator
struct WebhookSigner;

impl WebhookSigner {
    fn new() -> Self {
        Self
    }
    
    /// Sign payload
    fn sign(&self, payload: &WebhookPayload, secret: &str) -> Result<String> {
        use hmac::{Hmac, Mac};
        use sha2::Sha256;
        
        let payload_json = serde_json::to_string(payload)?;
        
        let mut mac = Hmac::<Sha256>::new_from_slice(secret.as_bytes())
            .map_err(|_| WebhookError::InvalidSecret)?;
        
        mac.update(payload_json.as_bytes());
        
        let result = mac.finalize();
        let signature = hex::encode(result.into_bytes());
        
        Ok(format!("sha256={}", signature))
    }
    
    /// Verify signature
    pub fn verify(&self, payload: &[u8], signature: &str, secret: &str) -> bool {
        use hmac::{Hmac, Mac};
        use sha2::Sha256;
        
        if !signature.starts_with("sha256=") {
            return false;
        }
        
        let provided_sig = &signature[7..];
        
        let mut mac = match Hmac::<Sha256>::new_from_slice(secret.as_bytes()) {
            Ok(m) => m,
            Err(_) => return false,
        };
        
        mac.update(payload);
        let expected = hex::encode(mac.finalize().into_bytes());
        
        // Constant time comparison
        provided_sig.len() == expected.len() &&
        provided_sig.bytes()
            .zip(expected.bytes())
            .all(|(a, b)| a == b)
    }
}

/// Delivery result
#[derive(Debug)]
pub struct DeliveryResult {
    pub success: bool,
    pub status_code: Option<u16>,
    pub response_time: Duration,
    pub error: Option<String>,
    pub response_body: Option<String>,
}
```

## Event Processing

```rust
/// Event processor for webhook payloads
pub struct EventProcessor {
    transformers: HashMap<String, Box<dyn PayloadTransformer>>,
    enrichers: Vec<Box<dyn PayloadEnricher>>,
}

impl EventProcessor {
    pub fn new() -> Self {
        let mut processor = Self {
            transformers: HashMap::new(),
            enrichers: Vec::new(),
        };
        
        // Register default transformers
        processor.register_defaults();
        
        processor
    }
    
    /// Process event into webhook payload
    pub async fn process_event(&self, event: &Event) -> Result<WebhookPayload> {
        // Create base payload
        let mut payload = WebhookPayload {
            event_id: event.id.0.to_string(),
            event_type: event.event_type.clone(),
            timestamp: event.timestamp,
            data: event.data.clone(),
            metadata: HashMap::new(),
        };
        
        // Apply transformer if available
        if let Some(transformer) = self.transformers.get(&event.event_type) {
            payload = transformer.transform(payload).await?;
        }
        
        // Apply enrichers
        for enricher in &self.enrichers {
            payload = enricher.enrich(payload).await?;
        }
        
        Ok(payload)
    }
    
    /// Register transformer
    pub fn register_transformer(
        &mut self,
        event_type: String,
        transformer: Box<dyn PayloadTransformer>,
    ) {
        self.transformers.insert(event_type, transformer);
    }
    
    /// Register enricher
    pub fn register_enricher(&mut self, enricher: Box<dyn PayloadEnricher>) {
        self.enrichers.push(enricher);
    }
    
    /// Register default transformers
    fn register_defaults(&mut self) {
        // Content events
        self.transformers.insert(
            "content.created".to_string(),
            Box::new(ContentTransformer::new()),
        );
        
        self.transformers.insert(
            "content.updated".to_string(),
            Box::new(ContentTransformer::new()),
        );
        
        // User events
        self.transformers.insert(
            "user.created".to_string(),
            Box::new(UserTransformer::new()),
        );
        
        // System events
        self.transformers.insert(
            "system.health".to_string(),
            Box::new(SystemTransformer::new()),
        );
    }
}

/// Webhook payload
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WebhookPayload {
    pub event_id: String,
    pub event_type: String,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub data: serde_json::Value,
    pub metadata: HashMap<String, String>,
}

/// Payload transformer trait
#[async_trait]
trait PayloadTransformer: Send + Sync {
    async fn transform(&self, payload: WebhookPayload) -> Result<WebhookPayload>;
}

/// Payload enricher trait
#[async_trait]
trait PayloadEnricher: Send + Sync {
    async fn enrich(&self, payload: WebhookPayload) -> Result<WebhookPayload>;
}

/// Content event transformer
struct ContentTransformer;

impl ContentTransformer {
    fn new() -> Self {
        Self
    }
}

#[async_trait]
impl PayloadTransformer for ContentTransformer {
    async fn transform(&self, mut payload: WebhookPayload) -> Result<WebhookPayload> {
        // Extract content data
        if let Some(content) = payload.data.get("content") {
            // Add computed fields
            let mut enriched = content.clone();
            
            if let Some(obj) = enriched.as_object_mut() {
                // Add URL
                if let Some(slug) = obj.get("slug").and_then(|s| s.as_str()) {
                    obj.insert(
                        "url".to_string(),
                        json!(format!("https://example.com/{}", slug)),
                    );
                }
                
                // Add preview URL
                if let Some(id) = obj.get("id").and_then(|s| s.as_str()) {
                    obj.insert(
                        "preview_url".to_string(),
                        json!(format!("https://example.com/preview/{}", id)),
                    );
                }
            }
            
            payload.data["content"] = enriched;
        }
        
        Ok(payload)
    }
}
```

## Retry Management

```rust
/// Webhook retry manager
pub struct RetryManager {
    queue: Arc<RetryQueue>,
    config: RetryConfig,
}

impl RetryManager {
    pub fn new(config: RetryConfig) -> Self {
        Self {
            queue: Arc::new(RetryQueue::new()),
            config,
        }
    }
    
    /// Start retry processor
    pub fn start(self: Arc<Self>) {
        tokio::spawn(async move {
            let mut interval = tokio::time::interval(self.config.check_interval);
            
            loop {
                interval.tick().await;
                
                if let Err(e) = self.process_retries().await {
                    error!("Retry processing error: {}", e);
                }
            }
        });
    }
    
    /// Queue webhook for retry
    pub async fn queue_retry(
        &self,
        webhook: &Webhook,
        delivery: &WebhookDelivery,
    ) -> Result<()> {
        let retry_count = self.get_retry_count(&delivery.id).await?;
        
        if retry_count >= self.config.max_retries {
            warn!("Max retries exceeded for delivery {}", delivery.id.0);
            return Ok(());
        }
        
        let delay = self.calculate_delay(retry_count);
        let retry_at = chrono::Utc::now() + delay;
        
        let retry_item = RetryItem {
            webhook: webhook.clone(),
            delivery: delivery.clone(),
            retry_count: retry_count + 1,
            retry_at,
        };
        
        self.queue.enqueue(retry_item).await?;
        
        Ok(())
    }
    
    /// Process pending retries
    async fn process_retries(&self) -> Result<()> {
        let items = self.queue.get_ready_items().await?;
        
        for item in items {
            // Dispatch webhook
            let dispatcher = get_webhook_manager().dispatcher.clone();
            let result = dispatcher.dispatch(&item.webhook, &item.delivery).await?;
            
            if !result.success {
                // Queue for another retry if not at max
                if item.retry_count < self.config.max_retries {
                    self.queue_retry(&item.webhook, &item.delivery).await?;
                } else {
                    // Mark as failed
                    self.mark_failed(&item.delivery.id).await?;
                }
            }
        }
        
        Ok(())
    }
    
    /// Calculate retry delay
    fn calculate_delay(&self, retry_count: u32) -> chrono::Duration {
        let seconds = match self.config.strategy {
            RetryStrategy::Fixed => self.config.base_delay_seconds,
            RetryStrategy::Linear => self.config.base_delay_seconds * retry_count,
            RetryStrategy::Exponential => {
                self.config.base_delay_seconds * 2u64.pow(retry_count.min(10))
            }
        };
        
        let seconds = seconds.min(self.config.max_delay_seconds);
        chrono::Duration::seconds(seconds as i64)
    }
}

/// Retry configuration
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RetryConfig {
    pub max_retries: u32,
    pub strategy: RetryStrategy,
    pub base_delay_seconds: u64,
    pub max_delay_seconds: u64,
    pub check_interval: Duration,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum RetryStrategy {
    Fixed,
    Linear,
    Exponential,
}
```

## Webhook Registry

```rust
/// Webhook storage and lookup
pub struct WebhookRegistry {
    webhooks: Arc<RwLock<HashMap<WebhookId, Webhook>>>,
    event_index: Arc<RwLock<HashMap<String, HashSet<WebhookId>>>>,
}

impl WebhookRegistry {
    pub fn new() -> Self {
        Self {
            webhooks: Arc::new(RwLock::new(HashMap::new())),
            event_index: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    /// Register webhook
    pub async fn register(&self, webhook: Webhook) -> Result<()> {
        let webhook_id = webhook.id.clone();
        
        // Store webhook
        self.webhooks.write().await.insert(webhook_id.clone(), webhook.clone());
        
        // Update event index
        let mut event_index = self.event_index.write().await;
        for event in &webhook.events {
            event_index
                .entry(event.clone())
                .or_insert_with(HashSet::new)
                .insert(webhook_id.clone());
        }
        
        Ok(())
    }
    
    /// Get webhooks for event
    pub async fn get_webhooks_for_event(&self, event_type: &str) -> Result<Vec<Webhook>> {
        let event_index = self.event_index.read().await;
        let webhooks = self.webhooks.read().await;
        
        let mut result = Vec::new();
        
        // Get exact matches
        if let Some(webhook_ids) = event_index.get(event_type) {
            for webhook_id in webhook_ids {
                if let Some(webhook) = webhooks.get(webhook_id) {
                    result.push(webhook.clone());
                }
            }
        }
        
        // Get wildcard matches
        if let Some(webhook_ids) = event_index.get("*") {
            for webhook_id in webhook_ids {
                if let Some(webhook) = webhooks.get(webhook_id) {
                    if !result.iter().any(|w| w.id == webhook.id) {
                        result.push(webhook.clone());
                    }
                }
            }
        }
        
        // Get pattern matches (e.g., "content.*")
        let event_prefix = event_type.split('.').next().unwrap_or("");
        let pattern = format!("{}.*", event_prefix);
        
        if let Some(webhook_ids) = event_index.get(&pattern) {
            for webhook_id in webhook_ids {
                if let Some(webhook) = webhooks.get(webhook_id) {
                    if !result.iter().any(|w| w.id == webhook.id) {
                        result.push(webhook.clone());
                    }
                }
            }
        }
        
        Ok(result)
    }
}
```

## Webhook Security

```rust
/// Webhook security validator
pub struct WebhookSecurity {
    allowed_ips: Option<Vec<IpNetwork>>,
    rate_limiter: Arc<RateLimiter>,
    secret_validator: Arc<SecretValidator>,
}

impl WebhookSecurity {
    /// Validate incoming webhook
    pub async fn validate_incoming(
        &self,
        request: &Request,
        secret: &str,
    ) -> Result<()> {
        // Check IP allowlist
        if let Some(allowed_ips) = &self.allowed_ips {
            let client_ip = extract_client_ip(request)?;
            
            if !allowed_ips.iter().any(|net| net.contains(client_ip)) {
                return Err(WebhookError::UnauthorizedIP(client_ip));
            }
        }
        
        // Check rate limit
        let client_id = extract_client_id(request)?;
        if !self.rate_limiter.check(&client_id).await {
            return Err(WebhookError::RateLimitExceeded);
        }
        
        // Validate signature
        let signature = request
            .headers()
            .get("X-Webhook-Signature")
            .and_then(|h| h.to_str().ok())
            .ok_or(WebhookError::MissingSignature)?;
        
        let body = request.body_bytes().await?;
        
        if !self.secret_validator.verify(&body, signature, secret) {
            return Err(WebhookError::InvalidSignature);
        }
        
        Ok(())
    }
    
    /// Validate outgoing webhook URL
    pub fn validate_url(&self, url: &str) -> Result<()> {
        let parsed = url::Url::parse(url)
            .map_err(|_| WebhookError::InvalidUrl)?;
        
        // Check protocol
        if parsed.scheme() != "https" && parsed.scheme() != "http" {
            return Err(WebhookError::InvalidProtocol);
        }
        
        // Disallow local addresses in production
        if cfg!(not(debug_assertions)) {
            if let Some(host) = parsed.host() {
                if is_local_address(host) {
                    return Err(WebhookError::LocalAddressNotAllowed);
                }
            }
        }
        
        Ok(())
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WebhookConfig {
    pub max_webhooks_per_tenant: usize,
    pub max_events_per_webhook: usize,
    pub request_timeout: Duration,
    pub max_payload_size: usize,
    pub retry: RetryConfig,
    pub security: SecurityConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SecurityConfig {
    pub require_https: bool,
    pub allow_local_urls: bool,
    pub ip_allowlist: Option<Vec<String>>,
    pub rate_limit_per_minute: u32,
}

impl Default for WebhookConfig {
    fn default() -> Self {
        Self {
            max_webhooks_per_tenant: 100,
            max_events_per_webhook: 50,
            request_timeout: Duration::from_secs(30),
            max_payload_size: 1024 * 1024, // 1MB
            retry: RetryConfig {
                max_retries: 3,
                strategy: RetryStrategy::Exponential,
                base_delay_seconds: 60,
                max_delay_seconds: 3600,
                check_interval: Duration::from_secs(60),
            },
            security: SecurityConfig {
                require_https: true,
                allow_local_urls: false,
                ip_allowlist: None,
                rate_limit_per_minute: 1000,
            },
        }
    }
}
```

## Summary

Diese Webhooks Implementation bietet:
- **Event-Driven Integration** - Flexible Event Routing
- **Reliable Delivery** - Retry Management, Error Handling
- **Security** - HMAC Signatures, IP Allowlisting
- **Payload Processing** - Transform and Enrich Events
- **Performance** - Async Dispatch, Rate Limiting
- **Monitoring** - Delivery Tracking, Test Mode

Complete Webhook System für External Integrations.