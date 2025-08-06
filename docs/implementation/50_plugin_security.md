# Plugin Security

## Overview

Plugin Security Implementation für ReedCMS. Comprehensive Security Model mit Sandboxing, Permissions und Resource Limits.

## Security Architecture

```rust
use std::collections::HashSet;
use tokio::sync::RwLock;

/// Plugin security manager
pub struct PluginSecurityManager {
    policies: Arc<RwLock<SecurityPolicies>>,
    permission_checker: Arc<PermissionChecker>,
    resource_monitor: Arc<ResourceMonitor>,
    audit_logger: Arc<AuditLogger>,
}

impl PluginSecurityManager {
    pub fn new(config: SecurityConfig) -> Self {
        Self {
            policies: Arc::new(RwLock::new(SecurityPolicies::from_config(&config))),
            permission_checker: Arc::new(PermissionChecker::new()),
            resource_monitor: Arc::new(ResourceMonitor::new(config.resource_limits)),
            audit_logger: Arc::new(AuditLogger::new(config.audit)),
        }
    }
    
    /// Validate plugin security
    pub async fn validate_plugin(
        &self,
        plugin_id: &str,
        manifest: &PluginManifest,
    ) -> Result<SecurityValidation> {
        let mut validation = SecurityValidation::new();
        
        // Check plugin signature
        if let Err(e) = self.verify_plugin_signature(plugin_id, manifest).await {
            validation.add_error(SecurityError::InvalidSignature(e.to_string()));
        }
        
        // Validate requested permissions
        self.validate_permissions(&manifest.permissions, &mut validation).await?;
        
        // Check resource requirements
        self.validate_resource_requirements(&manifest.resources, &mut validation).await?;
        
        // Scan for security risks
        self.scan_security_risks(plugin_id, &mut validation).await?;
        
        // Log validation result
        self.audit_logger.log_validation(plugin_id, &validation).await;
        
        Ok(validation)
    }
    
    /// Create security context for plugin
    pub async fn create_security_context(
        &self,
        plugin_id: &str,
        permissions: &PluginPermissions,
    ) -> Result<SecurityContext> {
        // Create permission set
        let granted_permissions = self.filter_permissions(permissions).await?;
        
        // Initialize resource quotas
        let resource_quota = self.resource_monitor
            .create_quota(plugin_id)
            .await?;
        
        // Create isolation context
        let isolation = self.create_isolation_context(plugin_id).await?;
        
        Ok(SecurityContext {
            plugin_id: plugin_id.to_string(),
            permissions: granted_permissions,
            resource_quota,
            isolation,
            created_at: chrono::Utc::now(),
        })
    }
}

/// Security validation result
#[derive(Debug)]
pub struct SecurityValidation {
    pub valid: bool,
    pub errors: Vec<SecurityError>,
    pub warnings: Vec<SecurityWarning>,
    pub risk_level: RiskLevel,
}

impl SecurityValidation {
    fn new() -> Self {
        Self {
            valid: true,
            errors: Vec::new(),
            warnings: Vec::new(),
            risk_level: RiskLevel::Low,
        }
    }
    
    fn add_error(&mut self, error: SecurityError) {
        self.valid = false;
        self.errors.push(error);
        self.update_risk_level();
    }
    
    fn add_warning(&mut self, warning: SecurityWarning) {
        self.warnings.push(warning);
        self.update_risk_level();
    }
    
    fn update_risk_level(&mut self) {
        if !self.errors.is_empty() {
            self.risk_level = RiskLevel::Critical;
        } else if self.warnings.len() > 3 {
            self.risk_level = RiskLevel::High;
        } else if !self.warnings.is_empty() {
            self.risk_level = RiskLevel::Medium;
        }
    }
}

#[derive(Debug, Clone)]
pub enum RiskLevel {
    Low,
    Medium,
    High,
    Critical,
}
```

## Permission System

```rust
/// Plugin permission system
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginPermissions {
    pub required: Vec<Permission>,
    pub optional: Vec<Permission>,
    pub capabilities: Vec<Capability>,
}

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum Permission {
    // Content permissions
    ContentRead,
    ContentWrite,
    ContentDelete,
    ContentPublish,
    
    // Storage permissions
    StorageRead,
    StorageWrite,
    StorageDelete,
    
    // Network permissions
    NetworkAccess,
    NetworkAccessHost(String),
    
    // System permissions
    SystemInfo,
    SystemExecute,
    
    // File permissions
    FileRead(PathPattern),
    FileWrite(PathPattern),
    
    // Event permissions
    EventSubscribe(String),
    EventPublish(String),
    
    // Admin permissions
    AdminAccess,
    AdminUsers,
    AdminSettings,
    
    // Custom permission
    Custom(String),
}

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct PathPattern {
    pattern: String,
    recursive: bool,
}

/// Permission checker
pub struct PermissionChecker {
    rules: Arc<RwLock<PermissionRules>>,
}

impl PermissionChecker {
    /// Check if plugin has permission
    pub async fn check_permission(
        &self,
        plugin_id: &str,
        permission: &Permission,
        context: &SecurityContext,
    ) -> Result<bool> {
        // Check if permission is granted
        if !context.permissions.contains(permission) {
            // Log permission denial
            self.log_permission_denial(plugin_id, permission).await;
            return Ok(false);
        }
        
        // Apply additional rules
        let rules = self.rules.read().await;
        if let Some(rule) = rules.get_rule_for_permission(permission) {
            return rule.evaluate(plugin_id, context).await;
        }
        
        Ok(true)
    }
    
    /// Check multiple permissions
    pub async fn check_permissions(
        &self,
        plugin_id: &str,
        permissions: &[Permission],
        context: &SecurityContext,
        require_all: bool,
    ) -> Result<bool> {
        if require_all {
            // All permissions must be granted
            for permission in permissions {
                if !self.check_permission(plugin_id, permission, context).await? {
                    return Ok(false);
                }
            }
            Ok(true)
        } else {
            // At least one permission must be granted
            for permission in permissions {
                if self.check_permission(plugin_id, permission, context).await? {
                    return Ok(true);
                }
            }
            Ok(false)
        }
    }
}

/// Permission rules
struct PermissionRules {
    rules: HashMap<Permission, Box<dyn PermissionRule>>,
}

#[async_trait]
trait PermissionRule: Send + Sync {
    async fn evaluate(
        &self,
        plugin_id: &str,
        context: &SecurityContext,
    ) -> Result<bool>;
}
```

## Resource Limits

```rust
use tokio::time::{interval, Duration};

/// Resource monitoring and limits
pub struct ResourceMonitor {
    limits: ResourceLimits,
    usage: Arc<RwLock<HashMap<String, ResourceUsage>>>,
    enforcer: Arc<ResourceEnforcer>,
}

impl ResourceMonitor {
    pub fn new(limits: ResourceLimits) -> Self {
        let monitor = Self {
            limits: limits.clone(),
            usage: Arc::new(RwLock::new(HashMap::new())),
            enforcer: Arc::new(ResourceEnforcer::new(limits)),
        };
        
        // Start monitoring task
        monitor.start_monitoring();
        
        monitor
    }
    
    /// Create resource quota for plugin
    pub async fn create_quota(
        &self,
        plugin_id: &str,
    ) -> Result<ResourceQuota> {
        let quota = ResourceQuota {
            plugin_id: plugin_id.to_string(),
            memory_limit: self.limits.memory_per_plugin,
            cpu_limit: self.limits.cpu_per_plugin,
            disk_limit: self.limits.disk_per_plugin,
            network_limit: self.limits.network_per_plugin,
            created_at: chrono::Utc::now(),
        };
        
        // Initialize usage tracking
        self.usage.write().await.insert(
            plugin_id.to_string(),
            ResourceUsage::default(),
        );
        
        Ok(quota)
    }
    
    /// Track resource usage
    pub async fn track_usage(
        &self,
        plugin_id: &str,
        resource: ResourceType,
        amount: u64,
    ) -> Result<()> {
        let mut usage_map = self.usage.write().await;
        
        if let Some(usage) = usage_map.get_mut(plugin_id) {
            match resource {
                ResourceType::Memory => {
                    usage.memory_bytes = usage.memory_bytes.saturating_add(amount);
                    
                    // Check limit
                    if usage.memory_bytes > self.limits.memory_per_plugin {
                        return Err(ReedError::ResourceLimitExceeded(
                            format!("Memory limit exceeded for plugin {}", plugin_id)
                        ));
                    }
                }
                ResourceType::Cpu => {
                    usage.cpu_time_ms = usage.cpu_time_ms.saturating_add(amount);
                }
                ResourceType::Disk => {
                    usage.disk_bytes = usage.disk_bytes.saturating_add(amount);
                    
                    // Check limit
                    if usage.disk_bytes > self.limits.disk_per_plugin {
                        return Err(ReedError::ResourceLimitExceeded(
                            format!("Disk limit exceeded for plugin {}", plugin_id)
                        ));
                    }
                }
                ResourceType::Network => {
                    usage.network_bytes = usage.network_bytes.saturating_add(amount);
                    
                    // Check rate limit
                    self.enforcer.check_network_rate(plugin_id, amount).await?;
                }
            }
        }
        
        Ok(())
    }
    
    /// Start resource monitoring
    fn start_monitoring(&self) {
        let usage = self.usage.clone();
        let limits = self.limits.clone();
        
        tokio::spawn(async move {
            let mut ticker = interval(Duration::from_secs(5));
            
            loop {
                ticker.tick().await;
                
                // Check all plugins
                let usage_snapshot = usage.read().await.clone();
                
                for (plugin_id, plugin_usage) in usage_snapshot {
                    // Check for violations
                    if plugin_usage.memory_bytes > limits.memory_per_plugin {
                        warn!(
                            "Plugin {} exceeding memory limit: {} bytes",
                            plugin_id, plugin_usage.memory_bytes
                        );
                        
                        // Take action (e.g., kill plugin)
                        // ...
                    }
                }
            }
        });
    }
}

#[derive(Debug, Clone, Default)]
struct ResourceUsage {
    memory_bytes: u64,
    cpu_time_ms: u64,
    disk_bytes: u64,
    network_bytes: u64,
    file_handles: u32,
}

#[derive(Debug, Clone)]
pub enum ResourceType {
    Memory,
    Cpu,
    Disk,
    Network,
}

/// Resource quota for plugin
#[derive(Debug, Clone)]
pub struct ResourceQuota {
    pub plugin_id: String,
    pub memory_limit: u64,
    pub cpu_limit: u64,
    pub disk_limit: u64,
    pub network_limit: u64,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

## Sandbox Implementation

```rust
/// Plugin sandbox implementation
pub struct PluginSandbox {
    mode: SandboxMode,
    isolation: IsolationLevel,
    namespace: Option<String>,
}

impl PluginSandbox {
    /// Create sandboxed execution environment
    pub async fn create_sandbox(
        &self,
        plugin_id: &str,
        security_context: &SecurityContext,
    ) -> Result<SandboxedEnvironment> {
        match self.mode {
            SandboxMode::None => {
                Ok(SandboxedEnvironment::None)
            }
            SandboxMode::Process => {
                self.create_process_sandbox(plugin_id, security_context).await
            }
            SandboxMode::Container => {
                self.create_container_sandbox(plugin_id, security_context).await
            }
            SandboxMode::Wasm => {
                self.create_wasm_sandbox(plugin_id, security_context).await
            }
        }
    }
    
    /// Create process-based sandbox
    async fn create_process_sandbox(
        &self,
        plugin_id: &str,
        security_context: &SecurityContext,
    ) -> Result<SandboxedEnvironment> {
        // Create restricted process
        let mut command = tokio::process::Command::new("reed-plugin-host");
        
        // Set environment
        command.env_clear();
        command.env("PLUGIN_ID", plugin_id);
        command.env("PLUGIN_TOKEN", &security_context.create_token()?);
        
        // Apply system restrictions
        #[cfg(unix)]
        {
            use std::os::unix::process::CommandExt;
            
            // Set resource limits
            command.pre_exec(move || {
                // Set memory limit
                let rlimit = libc::rlimit {
                    rlim_cur: security_context.resource_quota.memory_limit,
                    rlim_max: security_context.resource_quota.memory_limit,
                };
                unsafe {
                    libc::setrlimit(libc::RLIMIT_AS, &rlimit);
                }
                
                // Drop privileges if running as root
                if unsafe { libc::getuid() } == 0 {
                    unsafe {
                        libc::setgid(65534); // nobody
                        libc::setuid(65534); // nobody
                    }
                }
                
                Ok(())
            });
        }
        
        let child = command.spawn()?;
        
        Ok(SandboxedEnvironment::Process {
            process: child,
            ipc_channel: self.create_ipc_channel(plugin_id).await?,
        })
    }
    
    /// Create container-based sandbox
    async fn create_container_sandbox(
        &self,
        plugin_id: &str,
        security_context: &SecurityContext,
    ) -> Result<SandboxedEnvironment> {
        // Use OCI runtime (e.g., runc)
        let container_config = ContainerConfig {
            image: "reed-plugin-runtime:latest",
            hostname: format!("plugin-{}", plugin_id),
            network_mode: if security_context.permissions.contains(&Permission::NetworkAccess) {
                NetworkMode::Bridge
            } else {
                NetworkMode::None
            },
            memory_limit: security_context.resource_quota.memory_limit,
            cpu_quota: security_context.resource_quota.cpu_limit,
            readonly_rootfs: true,
            security_opts: vec![
                "no-new-privileges".to_string(),
                "apparmor=reed-plugin".to_string(),
            ],
            cap_drop: vec!["ALL".to_string()],
            cap_add: vec![], // No capabilities by default
        };
        
        let container = self.create_container(plugin_id, container_config).await?;
        
        Ok(SandboxedEnvironment::Container {
            container_id: container.id,
            api_socket: container.api_socket,
        })
    }
}

#[derive(Debug)]
pub enum SandboxedEnvironment {
    None,
    Process {
        process: tokio::process::Child,
        ipc_channel: IpcChannel,
    },
    Container {
        container_id: String,
        api_socket: PathBuf,
    },
    Wasm {
        instance: WasmInstance,
    },
}
```

## Security Policies

```rust
/// Security policy engine
pub struct SecurityPolicies {
    policies: Vec<SecurityPolicy>,
    default_policy: DefaultPolicy,
}

impl SecurityPolicies {
    /// Evaluate policies for plugin
    pub async fn evaluate(
        &self,
        plugin_id: &str,
        action: &SecurityAction,
        context: &SecurityContext,
    ) -> PolicyDecision {
        // Check specific policies first
        for policy in &self.policies {
            if policy.applies_to(plugin_id, action) {
                let decision = policy.evaluate(context).await;
                
                if matches!(decision, PolicyDecision::Deny(_)) {
                    return decision;
                }
            }
        }
        
        // Apply default policy
        self.default_policy.evaluate(action)
    }
    
    /// Add custom policy
    pub fn add_policy(&mut self, policy: SecurityPolicy) {
        self.policies.push(policy);
        self.policies.sort_by_key(|p| p.priority);
    }
}

#[derive(Debug, Clone)]
pub struct SecurityPolicy {
    pub id: String,
    pub name: String,
    pub description: String,
    pub priority: i32,
    pub conditions: Vec<PolicyCondition>,
    pub actions: Vec<PolicyAction>,
    pub effect: PolicyEffect,
}

#[derive(Debug, Clone)]
pub enum PolicyCondition {
    PluginId(String),
    PluginType(PluginType),
    Permission(Permission),
    ResourceUsage { resource: ResourceType, threshold: u64 },
    TimeRange { start: chrono::NaiveTime, end: chrono::NaiveTime },
    Custom(Box<dyn Fn(&SecurityContext) -> bool + Send + Sync>),
}

#[derive(Debug, Clone)]
pub enum PolicyAction {
    Allow,
    Deny,
    RateLimit { requests: u32, window: Duration },
    RequireApproval,
    Log,
    Alert,
}

#[derive(Debug, Clone)]
pub enum PolicyEffect {
    Permit,
    Deny,
}

#[derive(Debug)]
pub enum PolicyDecision {
    Allow,
    Deny(String),
    RateLimit { remaining: u32, reset: chrono::DateTime<chrono::Utc> },
}
```

## Audit Logging

```rust
/// Security audit logger
pub struct AuditLogger {
    writer: Arc<Mutex<AuditWriter>>,
    config: AuditConfig,
}

impl AuditLogger {
    /// Log security event
    pub async fn log_event(&self, event: SecurityEvent) {
        let entry = AuditEntry {
            id: Uuid::new_v4(),
            timestamp: chrono::Utc::now(),
            event,
            context: self.capture_context(),
        };
        
        // Write to audit log
        let mut writer = self.writer.lock().await;
        if let Err(e) = writer.write_entry(&entry).await {
            error!("Failed to write audit log: {}", e);
        }
        
        // Alert on critical events
        if entry.event.severity() == Severity::Critical {
            self.send_alert(&entry).await;
        }
    }
    
    /// Log permission check
    pub async fn log_permission_check(
        &self,
        plugin_id: &str,
        permission: &Permission,
        granted: bool,
    ) {
        self.log_event(SecurityEvent::PermissionCheck {
            plugin_id: plugin_id.to_string(),
            permission: permission.clone(),
            granted,
        }).await;
    }
    
    /// Log resource violation
    pub async fn log_resource_violation(
        &self,
        plugin_id: &str,
        resource: ResourceType,
        limit: u64,
        usage: u64,
    ) {
        self.log_event(SecurityEvent::ResourceViolation {
            plugin_id: plugin_id.to_string(),
            resource,
            limit,
            usage,
        }).await;
    }
}

#[derive(Debug, Serialize)]
pub struct AuditEntry {
    pub id: Uuid,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub event: SecurityEvent,
    pub context: AuditContext,
}

#[derive(Debug, Serialize)]
pub enum SecurityEvent {
    PluginLoaded {
        plugin_id: String,
        version: String,
        risk_level: RiskLevel,
    },
    PluginUnloaded {
        plugin_id: String,
        reason: String,
    },
    PermissionCheck {
        plugin_id: String,
        permission: Permission,
        granted: bool,
    },
    PermissionDenied {
        plugin_id: String,
        permission: Permission,
        reason: String,
    },
    ResourceViolation {
        plugin_id: String,
        resource: ResourceType,
        limit: u64,
        usage: u64,
    },
    SecurityViolation {
        plugin_id: String,
        violation_type: ViolationType,
        details: String,
    },
    PolicyViolation {
        plugin_id: String,
        policy_id: String,
        action: String,
    },
}

#[derive(Debug, Serialize)]
pub enum ViolationType {
    UnauthorizedAccess,
    DataExfiltration,
    MaliciousCode,
    ResourceAbuse,
    PolicyViolation,
}
```

## Plugin Signature Verification

```rust
use ed25519_dalek::{PublicKey, Signature, Verifier};

/// Plugin signature verification
pub struct SignatureVerifier {
    trusted_keys: HashMap<String, PublicKey>,
    config: SignatureConfig,
}

impl SignatureVerifier {
    /// Verify plugin signature
    pub async fn verify_plugin(
        &self,
        plugin_path: &Path,
        manifest: &PluginManifest,
    ) -> Result<VerificationResult> {
        // Check if signature is required
        if !self.config.require_signatures && manifest.signature.is_none() {
            return Ok(VerificationResult::Unsigned);
        }
        
        let signature = manifest.signature.as_ref()
            .ok_or_else(|| ReedError::SecurityError(
                "Plugin signature required but not found".to_string()
            ))?;
        
        // Get public key for author
        let public_key = self.trusted_keys.get(&signature.signer)
            .ok_or_else(|| ReedError::SecurityError(
                format!("Unknown signer: {}", signature.signer)
            ))?;
        
        // Read plugin files
        let plugin_data = self.collect_plugin_files(plugin_path).await?;
        
        // Verify signature
        let sig = Signature::from_bytes(&signature.signature_bytes)?;
        
        match public_key.verify(&plugin_data, &sig) {
            Ok(()) => Ok(VerificationResult::Valid {
                signer: signature.signer.clone(),
                signed_at: signature.timestamp,
            }),
            Err(_) => Ok(VerificationResult::Invalid {
                reason: "Signature verification failed".to_string(),
            }),
        }
    }
    
    /// Collect plugin files for verification
    async fn collect_plugin_files(&self, plugin_path: &Path) -> Result<Vec<u8>> {
        let mut hasher = sha2::Sha256::new();
        
        // Hash all plugin files in deterministic order
        let mut files = Vec::new();
        for entry in walkdir::WalkDir::new(plugin_path) {
            let entry = entry?;
            if entry.file_type().is_file() {
                files.push(entry.path().to_path_buf());
            }
        }
        
        // Sort for deterministic hashing
        files.sort();
        
        // Hash each file
        for file_path in files {
            let content = tokio::fs::read(&file_path).await?;
            hasher.update(&content);
        }
        
        Ok(hasher.finalize().to_vec())
    }
}

#[derive(Debug)]
pub enum VerificationResult {
    Valid {
        signer: String,
        signed_at: chrono::DateTime<chrono::Utc>,
    },
    Invalid {
        reason: String,
    },
    Unsigned,
}
```

## Summary

Dieses Plugin Security System bietet:
- **Permission System** - Granular Permission Control
- **Resource Limits** - Memory, CPU, Disk, Network Quotas
- **Sandboxing** - Process, Container, WASM Isolation
- **Security Policies** - Flexible Policy Engine
- **Audit Logging** - Comprehensive Security Audit Trail
- **Signature Verification** - Plugin Authenticity

Comprehensive Security für sichere Plugin Execution.