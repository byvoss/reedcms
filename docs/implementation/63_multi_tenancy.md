# Multi-Tenancy

## Overview

Multi-Tenancy Implementation für ReedCMS. Isolated Tenant Management mit Shared-Nothing Architecture.

## Tenant Manager

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use sqlx::PgPool;

/// Main multi-tenancy service
pub struct TenantManager {
    registry: Arc<TenantRegistry>,
    isolation: Arc<IsolationManager>,
    router: Arc<TenantRouter>,
    provisioner: Arc<TenantProvisioner>,
    config: TenantConfig,
}

impl TenantManager {
    pub fn new(pool: Arc<PgPool>, config: TenantConfig) -> Self {
        let registry = Arc::new(TenantRegistry::new());
        let isolation = Arc::new(IsolationManager::new(config.isolation.clone()));
        let router = Arc::new(TenantRouter::new());
        let provisioner = Arc::new(TenantProvisioner::new(pool));
        
        Self {
            registry,
            isolation,
            router,
            provisioner,
            config,
        }
    }
    
    /// Create new tenant
    pub async fn create_tenant(&self, request: CreateTenantRequest) -> Result<Tenant> {
        // Validate tenant
        self.validate_tenant_request(&request)?;
        
        // Generate tenant ID
        let tenant_id = TenantId::new();
        
        // Create tenant
        let tenant = Tenant {
            id: tenant_id.clone(),
            name: request.name,
            domain: request.domain,
            status: TenantStatus::Provisioning,
            isolation_mode: request.isolation_mode.unwrap_or(self.config.default_isolation),
            created_at: chrono::Utc::now(),
            metadata: request.metadata,
        };
        
        // Provision resources
        self.provisioner.provision(&tenant).await?;
        
        // Setup isolation
        self.isolation.setup_tenant(&tenant).await?;
        
        // Register tenant
        self.registry.register(tenant.clone()).await?;
        
        // Update status
        self.update_tenant_status(&tenant_id, TenantStatus::Active).await?;
        
        Ok(tenant)
    }
    
    /// Get tenant by ID
    pub async fn get_tenant(&self, tenant_id: &TenantId) -> Result<Option<Tenant>> {
        self.registry.get(tenant_id).await
    }
    
    /// Get tenant by domain
    pub async fn get_tenant_by_domain(&self, domain: &str) -> Result<Option<Tenant>> {
        self.registry.get_by_domain(domain).await
    }
    
    /// List all tenants
    pub async fn list_tenants(&self, filter: TenantFilter) -> Result<Vec<Tenant>> {
        self.registry.list(filter).await
    }
    
    /// Update tenant
    pub async fn update_tenant(
        &self,
        tenant_id: &TenantId,
        update: UpdateTenantRequest,
    ) -> Result<Tenant> {
        let mut tenant = self.get_tenant(tenant_id)
            .await?
            .ok_or_else(|| TenantError::NotFound(tenant_id.clone()))?;
        
        // Apply updates
        if let Some(name) = update.name {
            tenant.name = name;
        }
        
        if let Some(domain) = update.domain {
            // Validate domain change
            self.validate_domain_change(&tenant, &domain)?;
            tenant.domain = domain;
        }
        
        if let Some(metadata) = update.metadata {
            tenant.metadata.extend(metadata);
        }
        
        // Save changes
        self.registry.update(tenant.clone()).await?;
        
        Ok(tenant)
    }
    
    /// Delete tenant
    pub async fn delete_tenant(&self, tenant_id: &TenantId) -> Result<()> {
        let tenant = self.get_tenant(tenant_id)
            .await?
            .ok_or_else(|| TenantError::NotFound(tenant_id.clone()))?;
        
        // Update status
        self.update_tenant_status(tenant_id, TenantStatus::Deleting).await?;
        
        // Cleanup resources
        self.provisioner.deprovision(&tenant).await?;
        
        // Remove isolation
        self.isolation.cleanup_tenant(&tenant).await?;
        
        // Unregister
        self.registry.unregister(tenant_id).await?;
        
        Ok(())
    }
    
    /// Get tenant context for request
    pub async fn get_tenant_context(&self, request: &Request) -> Result<TenantContext> {
        let tenant = self.router.resolve_tenant(request).await?;
        
        Ok(TenantContext {
            tenant,
            isolation: self.isolation.get_isolation(&tenant.id).await?,
        })
    }
}

/// Tenant model
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Tenant {
    pub id: TenantId,
    pub name: String,
    pub domain: String,
    pub status: TenantStatus,
    pub isolation_mode: IsolationMode,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub metadata: HashMap<String, String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct TenantId(Uuid);

impl TenantId {
    pub fn new() -> Self {
        Self(Uuid::new_v4())
    }
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum TenantStatus {
    Provisioning,
    Active,
    Suspended,
    Deleting,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum IsolationMode {
    Shared,
    Schema,
    Database,
    Physical,
}
```

## Tenant Isolation

```rust
/// Isolation manager for tenant separation
pub struct IsolationManager {
    strategies: HashMap<IsolationMode, Box<dyn IsolationStrategy>>,
    config: IsolationConfig,
}

impl IsolationManager {
    pub fn new(config: IsolationConfig) -> Self {
        let mut strategies: HashMap<IsolationMode, Box<dyn IsolationStrategy>> = HashMap::new();
        
        strategies.insert(
            IsolationMode::Shared,
            Box::new(SharedIsolation::new()),
        );
        
        strategies.insert(
            IsolationMode::Schema,
            Box::new(SchemaIsolation::new()),
        );
        
        strategies.insert(
            IsolationMode::Database,
            Box::new(DatabaseIsolation::new()),
        );
        
        strategies.insert(
            IsolationMode::Physical,
            Box::new(PhysicalIsolation::new()),
        );
        
        Self { strategies, config }
    }
    
    /// Setup tenant isolation
    pub async fn setup_tenant(&self, tenant: &Tenant) -> Result<()> {
        let strategy = self.strategies
            .get(&tenant.isolation_mode)
            .ok_or_else(|| IsolationError::UnsupportedMode(tenant.isolation_mode))?;
        
        strategy.setup(tenant).await
    }
    
    /// Get isolation context
    pub async fn get_isolation(&self, tenant_id: &TenantId) -> Result<IsolationContext> {
        // Load tenant isolation info
        let info = self.load_isolation_info(tenant_id).await?;
        
        Ok(IsolationContext {
            tenant_id: tenant_id.clone(),
            mode: info.mode,
            connection_string: info.connection_string,
            schema_name: info.schema_name,
            resource_limits: info.resource_limits,
        })
    }
}

/// Isolation strategy trait
#[async_trait]
trait IsolationStrategy: Send + Sync {
    async fn setup(&self, tenant: &Tenant) -> Result<()>;
    async fn cleanup(&self, tenant: &Tenant) -> Result<()>;
    async fn get_connection(&self, tenant_id: &TenantId) -> Result<DatabaseConnection>;
}

/// Schema-based isolation
struct SchemaIsolation;

impl SchemaIsolation {
    fn new() -> Self {
        Self
    }
}

#[async_trait]
impl IsolationStrategy for SchemaIsolation {
    async fn setup(&self, tenant: &Tenant) -> Result<()> {
        let schema_name = format!("tenant_{}", tenant.id.0);
        
        // Create schema
        sqlx::query(&format!("CREATE SCHEMA IF NOT EXISTS {}", schema_name))
            .execute(&*get_master_pool())
            .await?;
        
        // Set permissions
        sqlx::query(&format!(
            "GRANT ALL ON SCHEMA {} TO tenant_role",
            schema_name
        ))
        .execute(&*get_master_pool())
        .await?;
        
        // Create tables in schema
        self.create_tenant_tables(&schema_name).await?;
        
        Ok(())
    }
    
    async fn cleanup(&self, tenant: &Tenant) -> Result<()> {
        let schema_name = format!("tenant_{}", tenant.id.0);
        
        // Drop schema cascade
        sqlx::query(&format!("DROP SCHEMA IF EXISTS {} CASCADE", schema_name))
            .execute(&*get_master_pool())
            .await?;
        
        Ok(())
    }
    
    async fn get_connection(&self, tenant_id: &TenantId) -> Result<DatabaseConnection> {
        let schema_name = format!("tenant_{}", tenant_id.0);
        
        // Return connection with schema set
        Ok(DatabaseConnection {
            pool: get_master_pool(),
            search_path: Some(schema_name),
        })
    }
}

/// Database-based isolation
struct DatabaseIsolation;

#[async_trait]
impl IsolationStrategy for DatabaseIsolation {
    async fn setup(&self, tenant: &Tenant) -> Result<()> {
        let db_name = format!("tenant_{}", tenant.id.0);
        
        // Create database
        sqlx::query(&format!("CREATE DATABASE {}", db_name))
            .execute(&*get_master_pool())
            .await?;
        
        // Create user
        let username = format!("tenant_{}_user", tenant.id.0);
        let password = generate_secure_password();
        
        sqlx::query(&format!(
            "CREATE USER {} WITH PASSWORD '{}'",
            username, password
        ))
        .execute(&*get_master_pool())
        .await?;
        
        // Grant privileges
        sqlx::query(&format!(
            "GRANT ALL PRIVILEGES ON DATABASE {} TO {}",
            db_name, username
        ))
        .execute(&*get_master_pool())
        .await?;
        
        // Store credentials
        self.store_tenant_credentials(&tenant.id, &username, &password).await?;
        
        // Initialize database
        self.initialize_tenant_database(&db_name).await?;
        
        Ok(())
    }
    
    async fn cleanup(&self, tenant: &Tenant) -> Result<()> {
        let db_name = format!("tenant_{}", tenant.id.0);
        let username = format!("tenant_{}_user", tenant.id.0);
        
        // Terminate connections
        sqlx::query(&format!(
            "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '{}'",
            db_name
        ))
        .execute(&*get_master_pool())
        .await?;
        
        // Drop database
        sqlx::query(&format!("DROP DATABASE IF EXISTS {}", db_name))
            .execute(&*get_master_pool())
            .await?;
        
        // Drop user
        sqlx::query(&format!("DROP USER IF EXISTS {}", username))
            .execute(&*get_master_pool())
            .await?;
        
        Ok(())
    }
    
    async fn get_connection(&self, tenant_id: &TenantId) -> Result<DatabaseConnection> {
        let credentials = self.get_tenant_credentials(tenant_id).await?;
        
        // Create tenant-specific connection pool
        let pool = PgPoolOptions::new()
            .max_connections(5)
            .connect(&credentials.connection_string)
            .await?;
        
        Ok(DatabaseConnection {
            pool: Arc::new(pool),
            search_path: None,
        })
    }
}
```

## Tenant Routing

```rust
/// Tenant routing for request handling
pub struct TenantRouter {
    strategies: Vec<Box<dyn RoutingStrategy>>,
    cache: Arc<RwLock<HashMap<String, TenantId>>>,
}

impl TenantRouter {
    pub fn new() -> Self {
        let strategies: Vec<Box<dyn RoutingStrategy>> = vec![
            Box::new(DomainRoutingStrategy::new()),
            Box::new(SubdomainRoutingStrategy::new()),
            Box::new(HeaderRoutingStrategy::new()),
            Box::new(PathRoutingStrategy::new()),
        ];
        
        Self {
            strategies,
            cache: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    /// Resolve tenant from request
    pub async fn resolve_tenant(&self, request: &Request) -> Result<Tenant> {
        // Try cache first
        let cache_key = self.build_cache_key(request);
        if let Some(tenant_id) = self.cache.read().await.get(&cache_key) {
            if let Some(tenant) = get_tenant_manager().get_tenant(tenant_id).await? {
                return Ok(tenant);
            }
        }
        
        // Try each strategy
        for strategy in &self.strategies {
            if let Some(tenant_id) = strategy.extract_tenant_id(request)? {
                if let Some(tenant) = get_tenant_manager().get_tenant(&tenant_id).await? {
                    // Cache result
                    self.cache.write().await.insert(cache_key, tenant_id);
                    return Ok(tenant);
                }
            }
        }
        
        Err(TenantError::NotResolved)
    }
}

/// Routing strategy trait
trait RoutingStrategy: Send + Sync {
    fn extract_tenant_id(&self, request: &Request) -> Result<Option<TenantId>>;
}

/// Domain-based routing
struct DomainRoutingStrategy;

impl DomainRoutingStrategy {
    fn new() -> Self {
        Self
    }
}

impl RoutingStrategy for DomainRoutingStrategy {
    fn extract_tenant_id(&self, request: &Request) -> Result<Option<TenantId>> {
        let host = request.headers()
            .get("Host")
            .and_then(|h| h.to_str().ok())?;
        
        if let Some(tenant) = get_tenant_manager()
            .get_tenant_by_domain(host)
            .await? {
            return Ok(Some(tenant.id));
        }
        
        Ok(None)
    }
}

/// Subdomain-based routing
struct SubdomainRoutingStrategy;

impl RoutingStrategy for SubdomainRoutingStrategy {
    fn extract_tenant_id(&self, request: &Request) -> Result<Option<TenantId>> {
        let host = request.headers()
            .get("Host")
            .and_then(|h| h.to_str().ok())?;
        
        // Extract subdomain
        if let Some(subdomain) = extract_subdomain(host) {
            if let Ok(uuid) = Uuid::parse_str(subdomain) {
                return Ok(Some(TenantId(uuid)));
            }
        }
        
        Ok(None)
    }
}

/// Header-based routing
struct HeaderRoutingStrategy;

impl RoutingStrategy for HeaderRoutingStrategy {
    fn extract_tenant_id(&self, request: &Request) -> Result<Option<TenantId>> {
        if let Some(tenant_header) = request.headers().get("X-Tenant-ID") {
            if let Ok(tenant_str) = tenant_header.to_str() {
                if let Ok(uuid) = Uuid::parse_str(tenant_str) {
                    return Ok(Some(TenantId(uuid)));
                }
            }
        }
        
        Ok(None)
    }
}
```

## Tenant Provisioning

```rust
/// Tenant provisioning service
pub struct TenantProvisioner {
    pool: Arc<PgPool>,
    provisioners: Vec<Box<dyn ResourceProvisioner>>,
}

impl TenantProvisioner {
    pub fn new(pool: Arc<PgPool>) -> Self {
        let provisioners: Vec<Box<dyn ResourceProvisioner>> = vec![
            Box::new(DatabaseProvisioner::new(pool.clone())),
            Box::new(StorageProvisioner::new()),
            Box::new(CacheProvisioner::new()),
            Box::new(SearchProvisioner::new()),
        ];
        
        Self { pool, provisioners }
    }
    
    /// Provision tenant resources
    pub async fn provision(&self, tenant: &Tenant) -> Result<()> {
        for provisioner in &self.provisioners {
            provisioner.provision(tenant).await?;
        }
        
        Ok(())
    }
    
    /// Deprovision tenant resources
    pub async fn deprovision(&self, tenant: &Tenant) -> Result<()> {
        for provisioner in &self.provisioners {
            provisioner.deprovision(tenant).await?;
        }
        
        Ok(())
    }
}

/// Resource provisioner trait
#[async_trait]
trait ResourceProvisioner: Send + Sync {
    async fn provision(&self, tenant: &Tenant) -> Result<()>;
    async fn deprovision(&self, tenant: &Tenant) -> Result<()>;
}

/// Storage provisioning
struct StorageProvisioner;

#[async_trait]
impl ResourceProvisioner for StorageProvisioner {
    async fn provision(&self, tenant: &Tenant) -> Result<()> {
        // Create tenant storage directory
        let storage_path = format!("data/tenants/{}", tenant.id.0);
        tokio::fs::create_dir_all(&storage_path).await?;
        
        // Set permissions
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            let permissions = std::fs::Permissions::from_mode(0o700);
            tokio::fs::set_permissions(&storage_path, permissions).await?;
        }
        
        // Create subdirectories
        for subdir in &["uploads", "cache", "temp"] {
            let path = format!("{}/{}", storage_path, subdir);
            tokio::fs::create_dir_all(&path).await?;
        }
        
        Ok(())
    }
    
    async fn deprovision(&self, tenant: &Tenant) -> Result<()> {
        let storage_path = format!("data/tenants/{}", tenant.id.0);
        
        if tokio::fs::metadata(&storage_path).await.is_ok() {
            tokio::fs::remove_dir_all(&storage_path).await?;
        }
        
        Ok(())
    }
}
```

## Tenant Middleware

```rust
/// Axum middleware for tenant context
pub async fn tenant_middleware<B>(
    State(manager): State<Arc<TenantManager>>,
    mut req: Request<B>,
    next: Next<B>,
) -> Result<Response, TenantError> {
    // Resolve tenant
    let tenant_context = manager.get_tenant_context(&req).await?;
    
    // Inject tenant context
    req.extensions_mut().insert(tenant_context);
    
    // Continue processing
    Ok(next.run(req).await)
}

/// Tenant context extractor
#[derive(Clone)]
pub struct TenantContext {
    pub tenant: Tenant,
    pub isolation: IsolationContext,
}

#[async_trait]
impl<S> FromRequestParts<S> for TenantContext
where
    S: Send + Sync,
{
    type Rejection = TenantError;
    
    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        parts
            .extensions
            .get::<TenantContext>()
            .cloned()
            .ok_or(TenantError::ContextNotFound)
    }
}

/// Database connection with tenant context
pub struct TenantDatabase {
    connection: DatabaseConnection,
    tenant_id: TenantId,
}

impl TenantDatabase {
    /// Execute query in tenant context
    pub async fn query<T>(&self, query: &str) -> Result<Vec<T>>
    where
        T: for<'r> sqlx::FromRow<'r, sqlx::postgres::PgRow> + Send + Unpin,
    {
        let query = self.prepare_query(query)?;
        
        let rows = sqlx::query_as::<_, T>(&query)
            .fetch_all(&*self.connection.pool)
            .await?;
        
        Ok(rows)
    }
    
    /// Prepare query with tenant context
    fn prepare_query(&self, query: &str) -> Result<String> {
        if let Some(ref search_path) = self.connection.search_path {
            // Prepend SET search_path
            Ok(format!("SET search_path TO {}; {}", search_path, query))
        } else {
            Ok(query.to_string())
        }
    }
}
```

## Tenant Limits

```rust
/// Resource limits per tenant
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TenantLimits {
    pub max_storage_bytes: u64,
    pub max_bandwidth_bytes: u64,
    pub max_requests_per_minute: u32,
    pub max_database_connections: u32,
    pub max_cache_memory_bytes: u64,
    pub allowed_features: HashSet<String>,
}

/// Tenant usage tracker
pub struct TenantUsageTracker {
    metrics: Arc<TenantMetrics>,
    limiter: Arc<TenantRateLimiter>,
}

impl TenantUsageTracker {
    /// Check if operation is allowed
    pub async fn check_limit(
        &self,
        tenant_id: &TenantId,
        resource: ResourceType,
        amount: u64,
    ) -> Result<()> {
        let limits = self.get_tenant_limits(tenant_id).await?;
        let usage = self.metrics.get_usage(tenant_id, resource).await?;
        
        match resource {
            ResourceType::Storage => {
                if usage + amount > limits.max_storage_bytes {
                    return Err(TenantError::StorageLimitExceeded);
                }
            }
            ResourceType::Bandwidth => {
                if usage + amount > limits.max_bandwidth_bytes {
                    return Err(TenantError::BandwidthLimitExceeded);
                }
            }
            ResourceType::Requests => {
                if !self.limiter.check_rate(tenant_id, limits.max_requests_per_minute).await {
                    return Err(TenantError::RateLimitExceeded);
                }
            }
        }
        
        Ok(())
    }
    
    /// Record usage
    pub async fn record_usage(
        &self,
        tenant_id: &TenantId,
        resource: ResourceType,
        amount: u64,
    ) -> Result<()> {
        self.metrics.increment_usage(tenant_id, resource, amount).await
    }
}

#[derive(Debug, Clone, Copy)]
pub enum ResourceType {
    Storage,
    Bandwidth,
    Requests,
    DatabaseConnections,
    CacheMemory,
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TenantConfig {
    pub default_isolation: IsolationMode,
    pub isolation: IsolationConfig,
    pub routing: RoutingConfig,
    pub limits: TenantLimitConfig,
    pub provisioning: ProvisioningConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IsolationConfig {
    pub schema_prefix: String,
    pub database_prefix: String,
    pub connection_pool_size: u32,
    pub resource_isolation: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RoutingConfig {
    pub strategies: Vec<String>,
    pub cache_ttl: Duration,
    pub default_tenant_header: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TenantLimitConfig {
    pub default_storage_gb: u64,
    pub default_bandwidth_gb: u64,
    pub default_requests_per_minute: u32,
    pub enforcement_mode: EnforcementMode,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum EnforcementMode {
    Soft,  // Log warnings
    Hard,  // Block requests
}

impl Default for TenantConfig {
    fn default() -> Self {
        Self {
            default_isolation: IsolationMode::Schema,
            isolation: IsolationConfig {
                schema_prefix: "tenant_".to_string(),
                database_prefix: "tenant_".to_string(),
                connection_pool_size: 5,
                resource_isolation: true,
            },
            routing: RoutingConfig {
                strategies: vec![
                    "domain".to_string(),
                    "subdomain".to_string(),
                    "header".to_string(),
                ],
                cache_ttl: Duration::from_secs(300),
                default_tenant_header: "X-Tenant-ID".to_string(),
            },
            limits: TenantLimitConfig {
                default_storage_gb: 10,
                default_bandwidth_gb: 100,
                default_requests_per_minute: 1000,
                enforcement_mode: EnforcementMode::Hard,
            },
            provisioning: ProvisioningConfig {
                auto_provision: true,
                cleanup_on_delete: true,
                backup_before_delete: true,
            },
        }
    }
}
```

## Summary

Diese Multi-Tenancy Implementation bietet:
- **Flexible Isolation** - Shared, Schema, Database, Physical
- **Automatic Routing** - Domain, Subdomain, Header, Path
- **Resource Management** - Provisioning, Limits, Usage Tracking
- **Tenant Lifecycle** - Create, Update, Delete, Suspend
- **Security Isolation** - Complete Tenant Separation
- **Performance** - Efficient Resource Sharing

Complete Multi-Tenant Architecture für SaaS Applications.