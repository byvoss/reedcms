# ReedCMS Plugin Store Architecture

## Store Processor Philosophy

**"reed_store processor" - Centralized Plugin Distribution & Quality Control**

Single processor service handling plugin submissions, testing, certification, and distribution with automatic quality enforcement through user feedback and crash reporting.

## Integrated Store Module Architecture

### **ReedCMS Core Integration with Optional Store**
```rust
// Integrated into main ReedCMS binary - not separate service
pub struct ReedCMS {
    core_system: CoreSystem,
    plugin_manager: PluginManager,
    store_processor: Option<ReedStoreProcessor>,  // Optional - can be disabled
    license_validator: LicenseValidator,          // Always runs (minimal)
}

pub struct ReedStoreProcessor {
    registry: PluginRegistry,
    test_framework: PluginTestFramework,
    certification_manager: CertificationManager,
    ranking_system: UserRankingSystem,
    background_scheduler: BackgroundScheduler,
    store_enabled: bool,                         // Runtime toggle
}

pub struct LicenseValidator {
    commercial_plugins: HashMap<String, CommercialCertification>,
    validation_interval: Duration,               // 30 minutes default
}

### **Store Configuration & User Access Control**
```toml
# config/store.toml
[store]
enabled = true                    # Can be disabled completely
public_browsing = false          # Only admins can browse store
allow_plugin_installation = true # Can be restricted

[permissions]
store_browse = ["admin"]         # Who can see the store
store_install = ["admin"]        # Who can install plugins  
store_submit = ["developer"]     # Who can submit plugins
store_commercial = ["verified_developer"] # Commercial plugin access

[license_validation]
enabled = true                   # Always enabled (even if store disabled)
interval_minutes = 30           # Background validation frequency
strict_mode = false             # Block expired plugins vs warning only
```

**User Permission System:**
```rust
pub struct StorePermissions {
    pub can_browse_store: bool,      // See plugin listings
    pub can_install_plugins: bool,   // Install new plugins
    pub can_submit_plugins: bool,    // Submit to store
    pub can_manage_commercial: bool, // Commercial plugin management
}

impl ReedCMS {
    pub async fn get_store_permissions(&self, user: &User) -> StorePermissions {
        let config = &self.config.store;
        
        StorePermissions {
            can_browse_store: config.store_enabled && 
                             user.has_any_role(&config.permissions.store_browse),
            
            can_install_plugins: config.store_enabled && 
                                config.allow_plugin_installation &&
                                user.has_any_role(&config.permissions.store_install),
            
            can_submit_plugins: config.store_enabled &&
                               user.has_any_role(&config.permissions.store_submit),
            
            can_manage_commercial: config.store_enabled &&
                                  user.has_any_role(&config.permissions.store_commercial),
        }
    }
}
```
```rust
impl ReedStoreProcessor {
    pub async fn start_background_scheduler(&mut self) -> Result<(), Error> {
        let mut interval = tokio::time::interval(Duration::from_mins(30));
        
        tokio::spawn(async move {
            loop {
                interval.tick().await;
                
                // License validation only when system is idle
                if self.is_system_idle().await {
                    self.validate_all_licenses().await.ok();
                    self.check_plugin_updates().await.ok();
                    self.cleanup_expired_certificates().await.ok();
                }
            }
        });
        
        Ok(())
    }
    
    async fn validate_all_licenses(&self) -> Result<(), Error> {
        let commercial_plugins = self.registry.get_commercial_plugins().await?;
        
        for plugin in commercial_plugins {
            if let Some(cert) = &plugin.certification {
                // Check if certificate expired
                if cert.expires_at < Utc::now() {
                    self.suspend_expired_plugin(&plugin.name).await?;
                }
                
                // Validate payment setup still active
                if !self.payment_processor.validate_account(&cert.payment_setup.stripe_account_id).await? {
                    self.notify_maintainer_payment_issue(&plugin.name).await?;
                }
            }
        }
        
        Ok(())
    }
    
    async fn is_system_idle(&self) -> bool {
        // Check if no active store operations
        self.active_store_requests.load(Ordering::Relaxed) == 0
    }
}

### **Idle Resource Management**
```rust
impl ReedStoreProcessor {
    // Store operations only run when explicitly requested
    pub async fn handle_store_request(&self, request: StoreRequest) -> Result<StoreResponse, Error> {
        // Increment active request counter
        self.active_store_requests.fetch_add(1, Ordering::Relaxed);
        
        let result = match request {
            StoreRequest::ListPlugins { user } => {
                self.get_visible_plugins_for_user(&user).await
            },
            StoreRequest::SubmitPlugin { package } => {
                // Heavy operation - run immediately when requested
                self.process_plugin_submission(package).await
            },
            StoreRequest::InstallPlugin { name, user } => {
                self.handle_plugin_installation(&name, &user).await
            },
        };
        
        // Decrement active request counter
        self.active_store_requests.fetch_sub(1, Ordering::Relaxed);
        
        result
    }
    
    // Background tasks only run during idle periods
    async fn run_background_maintenance(&self) {
        if !self.is_system_idle().await {
            return; // Skip if system busy
        }
        
        // Lightweight maintenance tasks
        self.cleanup_old_crash_reports().await.ok();
        self.update_download_statistics().await.ok();
        self.refresh_plugin_rankings().await.ok();
    }
}

pub enum PluginStatus {
    Stable,          // All tests passed, good user ranking
    Beta,            // Poor ranking or test failures - developer/installed only
    Commercial,      // Certified for commercial distribution with payment
    Suspended,       // Maintainer blocked for abuse
}
```

pub struct PluginMetadata {
    pub name: String,
    pub version: String,
    pub status: PluginStatus,
    pub maintainer: PluginMaintainer,
    pub test_results: TestResults,
    pub user_ranking: UserRanking,
    pub certification: Option<CommercialCertification>,
    pub visibility: StoreVisibility,
}

### **Resource-Efficient Design**
```rust
// Store processor sleeps when not needed
impl ReedStoreProcessor {
    pub async fn new() -> Self {
        ReedStoreProcessor {
            registry: PluginRegistry::new(),
            test_framework: PluginTestFramework::new(), 
            certification_manager: CertificationManager::new(),
            ranking_system: UserRankingSystem::new(),
            background_scheduler: BackgroundScheduler::new(),
            active_store_requests: AtomicU32::new(0),
        }
    }
    
    // Main loop - mostly sleeping
    pub async fn run(&mut self) -> Result<(), Error> {
        // Start 30-minute background scheduler
        self.start_background_scheduler().await?;
        
        // Process store requests when they come in
        while let Some(request) = self.request_receiver.recv().await {
            self.handle_store_request(request).await?;
        }
        
        Ok(())
    }
}

### **All Plugins Must Pass**
```rust
pub struct TestResults {
    pub functional_tests_passed: bool,    // Basic functionality works
    pub error_free_tests_passed: bool,    // No crashes/panics under stress
    pub performance_tests_passed: bool,   // Memory/CPU within limits
    pub security_scan_passed: bool,       // No obvious vulnerabilities
    pub last_test_run: DateTime<Utc>,
}

impl ReedStoreProcessor {
    pub async fn process_plugin_submission(&self, plugin: PluginPackage) -> Result<SubmissionResult, Error> {
        // 1. Functional Tests (MANDATORY)
        let functional_result = self.test_framework.run_functional_tests(&plugin).await?;
        if !functional_result.passed {
            return Ok(SubmissionResult::Rejected {
                reason: "Functional tests failed".into(),
                details: functional_result.error_details,
            });
        }
        
        // 2. Error-Free Tests (MANDATORY)
        let error_free_result = self.test_framework.run_error_free_tests(&plugin).await?;
        if !error_free_result.passed {
            return Ok(SubmissionResult::Rejected {
                reason: "Plugin crashes under stress testing".into(),
                details: error_free_result.error_details,
            });
        }
        
        // 3. Performance & Security (MANDATORY)
        let perf_result = self.test_framework.run_performance_tests(&plugin).await?;
        let security_result = self.test_framework.run_security_scan(&plugin).await?;
        
        if !perf_result.passed || !security_result.passed {
            return Ok(SubmissionResult::Rejected {
                reason: "Performance or security requirements not met".into(),
                details: format!("Perf: {}, Security: {}", perf_result.summary, security_result.summary),
            });
        }
        
        // 4. All tests passed - accept plugin
        let test_results = TestResults {
            functional_tests_passed: true,
            error_free_tests_passed: true,
            performance_tests_passed: true,
            security_scan_passed: true,
            last_test_run: Utc::now(),
        };
        
        self.registry.register_plugin(plugin, test_results).await?;
        Ok(SubmissionResult::Accepted)
    }
}
```

## Commercial Plugin Certification

### **Payment & Identity Verification**
```rust
pub struct CommercialCertification {
    pub certificate_id: String,         // Unique certificate ID
    pub maintainer_verified: bool,       // KYC verification completed
    pub payment_setup: PaymentSetup,     // Billing information
    pub issued_at: DateTime<Utc>,
    pub expires_at: DateTime<Utc>,
    pub can_be_suspended: bool,          // Suspension capability
}

pub struct PaymentSetup {
    pub stripe_account_id: String,       // For revenue distribution
    pub tax_information: TaxInfo,        // Tax compliance data
    pub payout_method: PayoutMethod,     // Payment destination
    pub revenue_share: f64,              // e.g., 70% to maintainer, 30% to ReedCMS
}

impl CertificationManager {
    pub async fn issue_commercial_certificate(
        &self, 
        plugin_name: &str, 
        maintainer: &PluginMaintainer
    ) -> Result<CommercialCertification, CertificationError> {
        // 1. KYC Verification
        self.verify_maintainer_identity(maintainer).await?;
        
        // 2. Payment Setup Validation
        self.validate_stripe_account(&maintainer.payment_info).await?;
        
        // 3. Issue Certificate
        let cert = CommercialCertification {
            certificate_id: generate_certificate_id(),
            maintainer_verified: true,
            payment_setup: maintainer.payment_info.clone(),
            issued_at: Utc::now(),
            expires_at: Utc::now() + Duration::days(365), // 1 year validity
            can_be_suspended: true,
        };
        
        // 4. Store in registry
        self.registry.store_certificate(&plugin_name, &cert).await?;
        
        // 5. Enable commercial distribution
        self.registry.update_plugin_status(&plugin_name, PluginStatus::Commercial).await?;
        
        Ok(cert)
    }
    
    pub async fn suspend_maintainer(&self, maintainer_id: &str, reason: &str) -> Result<(), Error> {
        // Suspend all plugins by this maintainer
        let plugins = self.registry.get_plugins_by_maintainer(maintainer_id).await?;
        
        for plugin in plugins {
            self.registry.update_plugin_status(&plugin.name, PluginStatus::Suspended).await?;
        }
        
        // Stop revenue payments
        self.payment_processor.suspend_payments(maintainer_id).await?;
        
        // Notify maintainer
        self.notification_service.send_suspension_notice(maintainer_id, reason).await?;
        
        Ok(())
    }
}
```

## User Feedback & Ranking System

### **Crash Report Processing**
```rust
pub struct UserRanking {
    pub total_downloads: u64,
    pub crash_reports: u64,
    pub user_ratings: Vec<UserRating>,
    pub average_rating: f64,
    pub stability_score: f64,        // Based on crash reports
}

pub struct CrashReport {
    pub plugin_name: String,
    pub plugin_version: String,
    pub error_details: String,
    pub stack_trace: Option<String>,
    pub user_id: String,
    pub reedcms_version: String,
    pub reported_at: DateTime<Utc>,
}

impl UserRankingSystem {
    pub async fn process_crash_report(&mut self, report: CrashReport) -> Result<(), Error> {
        // 1. Store crash report
        self.store_crash_report(&report).await?;
        
        // 2. Recalculate plugin stability score
        let stability_score = self.calculate_stability_score(&report.plugin_name).await?;
        
        // 3. Check for beta degradation
        if stability_score < 0.7 {  // Beta degradation threshold
            self.degrade_to_beta_status(&report.plugin_name, stability_score).await?;
        }
        
        // 4. Notify maintainer if significant stability drop
        if stability_score < 0.5 {
            self.notify_maintainer_critical_stability(&report.plugin_name, stability_score).await?;
        }
        
        Ok(())
    }
    
    async fn calculate_stability_score(&self, plugin_name: &str) -> Result<f64, Error> {
        let downloads = self.get_total_downloads_last_30_days(plugin_name).await?;
        let crashes = self.get_crash_count_last_30_days(plugin_name).await?;
        
        if downloads == 0 {
            return Ok(1.0); // No downloads = no crashes possible
        }
        
        // Stability Score: 1.0 - (crashes / downloads)
        let score = 1.0 - (crashes as f64 / downloads as f64);
        Ok(score.max(0.0).min(1.0))
    }
    
    async fn degrade_to_beta_status(&self, plugin_name: &str, stability_score: f64) -> Result<(), Error> {
        // Update plugin status to beta
        self.registry.update_plugin_status(plugin_name, PluginStatus::Beta).await?;
        
        // Restrict store visibility
        self.registry.update_store_visibility(plugin_name, StoreVisibility::DeveloperAndInstalled).await?;
        
        // Notify maintainer
        self.notification_service.send_beta_degradation_notice(
            plugin_name, 
            stability_score,
            "Your plugin has been moved to beta status due to stability issues"
        ).await?;
        
        Ok(())
    }
}
```

## Store Visibility System

### **Visibility Rules Based on Plugin Status**
```rust
pub enum StoreVisibility {
    Public,              // Visible to all users (Stable, Commercial)
    DeveloperAndInstalled, // Only developers and users who have it installed (Beta)
    InstalledOnly,       // Only users who already have it installed (Suspended)
    Hidden,              // Completely hidden (Removed)
}

impl ReedStoreProcessor {
    pub async fn get_visible_plugins_for_user(&self, user: &User) -> Result<Vec<PluginMetadata>, Error> {
        let all_plugins = self.registry.get_all_plugins().await?;
        let mut visible_plugins = Vec::new();
        
        for plugin in all_plugins {
            let is_visible = match plugin.status {
                PluginStatus::Stable | PluginStatus::Commercial => {
                    // Always visible for stable and commercial plugins
                    true
                },
                
                PluginStatus::Beta => {
                    // Beta: Only visible to developers or users who have it installed
                    user.is_developer() || user.has_plugin_installed(&plugin.name)
                },
                
                PluginStatus::Suspended => {
                    // Suspended: Only visible to users who already have it installed
                    // (Allows them to uninstall but not install)
                    user.has_plugin_installed(&plugin.name)
                }
            };
            
            if is_visible {
                visible_plugins.push(plugin);
            }
        }
        
        Ok(visible_plugins)
    }
}
```

## Automated Testing Framework

### **Docker-Based Isolated Testing**
```rust
pub struct PluginTestFramework {
    docker_runner: DockerTestRunner,
    security_scanner: SecurityScanner,
    performance_monitor: PerformanceMonitor,
}

impl PluginTestFramework {
    pub async fn run_functional_tests(&self, plugin: &PluginPackage) -> Result<TestResult, Error> {
        // 1. Start plugin in isolated Docker container
        let container = self.docker_runner.start_plugin_container(plugin).await?;
        
        // 2. Test basic functionality
        let basic_commands = vec![
            "health_check",
            "process_sample_content", 
            "validate_sample_input",
            "handle_error_input"
        ];
        
        for command in basic_commands {
            let result = container.execute_command(command, Duration::from_secs(10)).await?;
            if !result.success {
                return Ok(TestResult::failed(format!("Command '{}' failed: {}", command, result.error)));
            }
        }
        
        // 3. Test with ReedCMS integration
        let integration_result = self.test_reedcms_integration(&container).await?;
        if !integration_result.success {
            return Ok(TestResult::failed(format!("ReedCMS integration failed: {}", integration_result.error)));
        }
        
        Ok(TestResult::passed())
    }
    
    pub async fn run_error_free_tests(&self, plugin: &PluginPackage) -> Result<TestResult, Error> {
        let container = self.docker_runner.start_plugin_container(plugin).await?;
        
        // Stress tests designed to find crashes
        let stress_tests = vec![
            ("invalid_json_input", r#"{"invalid": json malformed"#),
            ("extremely_large_content", &"x".repeat(10_000_000)), // 10MB string
            ("malformed_unicode", "\xFF\xFE invalid unicode"),
            ("null_byte_injection", "content\0with\0nulls"),
            ("concurrent_requests", ""), // Special test case
        ];
        
        for (test_name, test_input) in stress_tests {
            if test_name == "concurrent_requests" {
                // Test concurrent access
                let concurrent_result = self.test_concurrent_access(&container).await?;
                if concurrent_result.has_crashes() {
                    return Ok(TestResult::failed(format!("Crash during concurrent access test")));
                }
            } else {
                let result = container.execute_command_with_input("process_content", test_input).await?;
                if result.crashed {
                    return Ok(TestResult::failed(format!("Crash in test '{}': {}", test_name, result.crash_details)));
                }
            }
        }
        
        Ok(TestResult::passed())
    }
    
    pub async fn run_performance_tests(&self, plugin: &PluginPackage) -> Result<TestResult, Error> {
        let container = self.docker_runner.start_plugin_container(plugin).await?;
        
        // Monitor resource usage during testing
        let monitor = self.performance_monitor.start_monitoring(&container).await?;
        
        // Performance test scenarios
        let perf_tests = vec![
            ("small_content", "Small content test", 100),      // 100 iterations
            ("medium_content", &"Content ".repeat(1000), 50),  // 50 iterations
            ("large_content", &"Large ".repeat(10000), 10),    // 10 iterations
        ];
        
        for (test_name, content, iterations) in perf_tests {
            for _ in 0..iterations {
                container.execute_command_with_input("process_content", content).await?;
            }
        }
        
        let usage_stats = monitor.get_final_stats().await?;
        
        // Check performance limits
        if usage_stats.max_memory_mb > 100 {  // 100MB limit
            return Ok(TestResult::failed(format!("Memory usage too high: {}MB", usage_stats.max_memory_mb)));
        }
        
        if usage_stats.avg_cpu_percent > 80.0 {  // 80% CPU limit
            return Ok(TestResult::failed(format!("CPU usage too high: {:.1}%", usage_stats.avg_cpu_percent)));
        }
        
        Ok(TestResult::passed())
    }
}
```

## CLI Integration

### **Store Management Commands**
```bash
# Plugin Store Browsing
reed store list                           # All visible plugins for current user
reed store search "seo"                   # Search plugins
reed store info ai-core                   # Detailed plugin information
reed store ratings ai-core                # User ratings and reviews

# Plugin Installation
reed store install ai-core                # Install latest stable version
reed store install ai-core --version 1.2.0 # Install specific version
reed store uninstall ai-core              # Remove plugin

# Developer Commands
reed store submit ./my-plugin.tar.gz      # Submit plugin for review
reed store test ./my-plugin/              # Run local tests before submission
reed store status my-plugin               # Check submission status

# Developer Beta Access
reed store list --show-beta               # Show beta plugins (developers only)
reed store install my-plugin --beta       # Install beta version

# Commercial Plugin Management
reed store certify my-plugin --commercial # Apply for commercial certification
reed store earnings                       # View earnings from commercial plugins
reed store analytics my-plugin            # Download and usage statistics

# Maintainer Commands
reed store crash-reports my-plugin        # View crash reports for plugin
reed store user-feedback my-plugin        # View user ratings and comments
```

### **Memory Footprint by Configuration**
```rust
// Store disabled - minimal memory usage
impl ReedCMS {
    pub fn memory_usage_store_disabled() -> MemoryUsage {
        MemoryUsage {
            license_validator: 512_KB,  // Only commercial plugin certificates
            plugin_registry: 0_KB,      // Not loaded
            test_framework: 0_KB,       // Not loaded
            total: 512_KB               // Minimal footprint
        }
    }
    
    pub fn memory_usage_store_enabled() -> MemoryUsage {
        MemoryUsage {
            license_validator: 512_KB,
            plugin_registry: 2_MB,      // Plugin metadata cache
            test_framework: 1_MB,       // Docker test runner
            ranking_system: 512_KB,     // User ratings cache
            total: 4_MB                 // Full store functionality
        }
    }
}

### **Resource Usage Patterns**
- **Store Disabled:** ~512KB RAM, license checks every 30min (1-2 seconds CPU)
- **Store Enabled + Idle:** ~4MB RAM, background tasks every 30min (5-10 seconds CPU)  
- **Store Active (browsing):** ~8MB RAM, immediate response
- **Store Active (testing):** ~50MB RAM (temporary Docker containers)
```
```rust
// Built into ReedCMS core - automatic crash report submission
impl PluginManager {
    async fn handle_plugin_crash(&self, plugin_name: &str, error: &Error) -> Result<(), Error> {
        // Create crash report
        let crash_report = CrashReport {
            plugin_name: plugin_name.to_string(),
            plugin_version: self.get_plugin_version(plugin_name).await?,
            error_details: error.to_string(),
            stack_trace: error.backtrace().map(|bt| bt.to_string()),
            user_id: self.get_anonymous_user_id(), // Anonymous but trackable
            reedcms_version: env!("CARGO_PKG_VERSION").to_string(),
            reported_at: Utc::now(),
        };
        
        // Submit to reed_store processor
        self.store_client.submit_crash_report(crash_report).await?;
        
        Ok(())
    }
}
```

## Revenue Distribution

### **Commercial Plugin Economics**
```rust
pub struct RevenueDistribution {
    pub plugin_price: Decimal,           // Set by maintainer
    pub reedcms_fee: Decimal,           // 30% platform fee
    pub maintainer_revenue: Decimal,     // 70% to maintainer
    pub transaction_fees: Decimal,       // Stripe/PayPal fees
}

impl PaymentProcessor {
    pub async fn process_plugin_purchase(
        &self, 
        plugin_name: &str, 
        buyer_id: &str
    ) -> Result<PurchaseResult, Error> {
        let plugin = self.registry.get_plugin(plugin_name).await?;
        let certification = plugin.certification
            .ok_or(PaymentError::NotCommercialPlugin)?;
        
        // Calculate revenue distribution
        let revenue = RevenueDistribution {
            plugin_price: plugin.price,
            reedcms_fee: plugin.price * Decimal::from_str("0.30")?, // 30%
            maintainer_revenue: plugin.price * Decimal::from_str("0.70")?, // 70%
            transaction_fees: plugin.price * Decimal::from_str("0.029")?, // ~3% Stripe
        };
        
        // Process payment through Stripe
        let payment_result = self.stripe_client.create_payment_intent(
            &revenue.plugin_price,
            &buyer_id,
            &format!("ReedCMS Plugin: {}", plugin_name)
        ).await?;
        
        if payment_result.succeeded {
            // Distribute revenue
            self.distribute_revenue(&certification.payment_setup, &revenue).await?;
            
            // Grant plugin access to buyer
            self.grant_plugin_access(plugin_name, buyer_id).await?;
            
            Ok(PurchaseResult::Success)
        } else {
            Ok(PurchaseResult::Failed(payment_result.error))
        }
    }
}
```

## Data Storage

### **Plugin Registry Database Schema**
```sql
-- Plugin metadata storage
CREATE TABLE plugins (
    id UUID PRIMARY KEY,
    name VARCHAR(255) UNIQUE NOT NULL,
    version VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL, -- 'stable', 'beta', 'commercial', 'suspended'
    maintainer_id UUID NOT NULL,
    description TEXT,
    price DECIMAL(10,2), -- NULL for free plugins
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Test results tracking
CREATE TABLE plugin_test_results (
    id UUID PRIMARY KEY,
    plugin_id UUID REFERENCES plugins(id),
    test_type VARCHAR(50) NOT NULL, -- 'functional', 'error_free', 'performance', 'security'
    passed BOOLEAN NOT NULL,
    details TEXT,
    tested_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Crash reports
CREATE TABLE crash_reports (
    id UUID PRIMARY KEY,
    plugin_name VARCHAR(255) NOT NULL,
    plugin_version VARCHAR(50) NOT NULL,
    error_details TEXT NOT NULL,
    stack_trace TEXT,
    user_id UUID, -- Anonymous but trackable
    reedcms_version VARCHAR(50),
    reported_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- User ratings
CREATE TABLE plugin_ratings (
    id UUID PRIMARY KEY,
    plugin_id UUID REFERENCES plugins(id),
    user_id UUID NOT NULL,
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Commercial certifications
CREATE TABLE commercial_certifications (
    id UUID PRIMARY KEY,
    plugin_id UUID REFERENCES plugins(id),
    certificate_id VARCHAR(255) UNIQUE NOT NULL,
    maintainer_verified BOOLEAN DEFAULT FALSE,
    stripe_account_id VARCHAR(255),
    issued_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL
);
```

## Security & Quality Assurance

### **Multi-Layer Security**
1. **Submission Security:** All plugins run in isolated Docker containers during testing
2. **Code Analysis:** Automated security scanning for common vulnerabilities
3. **Permission Validation:** Plugin permission requirements verified against actual usage
4. **Sandboxed Execution:** Plugins run with limited system access in production
5. **Crash Monitoring:** Automatic crash detection and reporting
6. **User Feedback:** Community-driven quality control through ratings and crash reports

### **Quality Metrics**
```rust
pub struct QualityMetrics {
    pub stability_score: f64,        // Based on crash reports vs downloads
    pub user_satisfaction: f64,      // Average user rating
    pub performance_score: f64,      // Memory/CPU efficiency
    pub security_score: f64,         // Security scan results
    pub maintenance_score: f64,      // Update frequency and responsiveness
}
```

This system ensures high-quality plugins through automated testing, user feedback integration, and financial incentives for maintainers to keep their plugins stable and well-maintained.