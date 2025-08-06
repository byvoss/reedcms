# Configuration Types

## Overview

Konfigurationsstrukturen für ReedCMS. Alle Konfigurationen folgen dem KISS-Prinzip mit klaren Defaults.

## Main Configuration

```rust
use serde::{Deserialize, Serialize};
use std::path::PathBuf;

/// Root configuration loaded from config/reed.toml
#[derive(Debug, Clone, Deserialize)]
pub struct ReedConfig {
    pub site: SiteConfig,
    pub database: DatabaseConfig,
    pub redis: RedisConfig,
    pub server: ServerConfig,
    pub paths: PathConfig,
    pub security: SecurityConfig,
    pub performance: PerformanceConfig,
    pub locale: LocaleConfig,
    pub plugins: PluginConfig,
    pub firewall: FirewallConfig,
}

impl Default for ReedConfig {
    fn default() -> Self {
        Self {
            site: SiteConfig::default(),
            database: DatabaseConfig::default(),
            redis: RedisConfig::default(),
            server: ServerConfig::default(),
            paths: PathConfig::default(),
            security: SecurityConfig::default(),
            performance: PerformanceConfig::default(),
            locale: LocaleConfig::default(),
            plugins: PluginConfig::default(),
            firewall: FirewallConfig::default(),
        }
    }
}
```

## Site Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct SiteConfig {
    pub name: String,
    pub domain: String,
    pub default_theme: String,
    pub admin_email: String,
}

impl Default for SiteConfig {
    fn default() -> Self {
        Self {
            name: "ReedCMS Site".to_string(),
            domain: "localhost".to_string(),
            default_theme: "base".to_string(),
            admin_email: "admin@localhost".to_string(),
        }
    }
}
```

## Database Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct DatabaseConfig {
    pub main_url: String,
    pub ucg_url: String,
    pub max_connections: u32,
    pub min_connections: u32,
    pub connection_timeout_seconds: u64,
    pub idle_timeout_seconds: u64,
    pub statement_cache_capacity: usize,
}

impl Default for DatabaseConfig {
    fn default() -> Self {
        Self {
            main_url: "postgresql://reed:reed@localhost/reed_main".to_string(),
            ucg_url: "postgresql://reed:reed@localhost/reed_ucg".to_string(),
            max_connections: 20,
            min_connections: 5,
            connection_timeout_seconds: 30,
            idle_timeout_seconds: 600,
            statement_cache_capacity: 100,
        }
    }
}
```

## Redis Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct RedisConfig {
    pub url: String,
    pub database: u8,
    pub max_memory: String,
    pub max_memory_policy: EvictionPolicy,
    pub key_prefix: String,
}

#[derive(Debug, Clone, Deserialize)]
#[serde(rename_all = "kebab-case")]
pub enum EvictionPolicy {
    VolatileLru,
    AllkeysLru,
    NoEviction,
}

impl Default for RedisConfig {
    fn default() -> Self {
        Self {
            url: "redis://localhost:6379".to_string(),
            database: 0,
            max_memory: "512MB".to_string(),
            max_memory_policy: EvictionPolicy::VolatileLru,
            key_prefix: "reed:".to_string(),
        }
    }
}
```

## Server Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
    pub workers: Option<usize>,  // None = CPU count
    pub request_timeout_seconds: u64,
    pub body_limit: String,
    pub upload_limit: String,
}

impl Default for ServerConfig {
    fn default() -> Self {
        Self {
            host: "0.0.0.0".to_string(),
            port: 3000,
            workers: None,
            request_timeout_seconds: 30,
            body_limit: "1MB".to_string(),
            upload_limit: "10MB".to_string(),
        }
    }
}
```

## Path Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct PathConfig {
    pub base_dir: PathBuf,
    pub config_dir: PathBuf,
    pub snippets_dir: PathBuf,
    pub themes_dir: PathBuf,
    pub plugins_dir: PathBuf,
    pub assets_dir: PathBuf,
    pub build_dir: PathBuf,
    pub socket_dir: PathBuf,
}

impl Default for PathConfig {
    fn default() -> Self {
        let base = PathBuf::from(".");
        Self {
            base_dir: base.clone(),
            config_dir: base.join("config"),
            snippets_dir: base.join("snippets"),
            themes_dir: base.join("themes"),
            plugins_dir: base.join("plugins"),
            assets_dir: base.join("assets"),
            build_dir: base.join("build"),
            socket_dir: PathBuf::from("/var/run/reed-cms"),
        }
    }
}
```

## Security Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct SecurityConfig {
    pub session_duration_hours: u32,
    pub password_min_length: usize,
    pub require_2fa: bool,
    pub secure_cookies: bool,
    pub csrf_protection: bool,
    pub rate_limiting: RateLimitConfig,
    pub headers: SecurityHeaders,
}

#[derive(Debug, Clone, Deserialize)]
pub struct RateLimitConfig {
    pub login_attempts_per_15min: u32,
    pub api_requests_per_minute: u32,
    pub admin_requests_per_minute: u32,
}

#[derive(Debug, Clone, Deserialize)]
pub struct SecurityHeaders {
    pub enable_hsts: bool,
    pub enable_csp: bool,
    pub frame_options: String,
}

impl Default for SecurityConfig {
    fn default() -> Self {
        Self {
            session_duration_hours: 24,
            password_min_length: 12,
            require_2fa: false,
            secure_cookies: true,
            csrf_protection: true,
            rate_limiting: RateLimitConfig {
                login_attempts_per_15min: 5,
                api_requests_per_minute: 60,
                admin_requests_per_minute: 30,
            },
            headers: SecurityHeaders {
                enable_hsts: true,
                enable_csp: true,
                frame_options: "DENY".to_string(),
            },
        }
    }
}
```

## Performance Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct PerformanceConfig {
    pub cache_ttl_seconds: u64,
    pub template_cache_size: usize,
    pub query_cache_enabled: bool,
    pub hot_reload_enabled: bool,
    pub benchmark_enabled: bool,
    pub memory_monitoring: MemoryMonitorConfig,
    pub intelligence: IntelligenceConfig,
}

#[derive(Debug, Clone, Deserialize)]
pub struct MemoryMonitorConfig {
    pub enabled: bool,
    pub check_interval_seconds: u64,
    pub warning_threshold_percent: u8,
    pub critical_threshold_percent: u8,
}

#[derive(Debug, Clone, Deserialize)]
pub struct IntelligenceConfig {
    pub error_pattern_detection: bool,
    pub anomaly_detection: bool,
    pub security_pattern_matching: bool,
    pub provider_plugin: Option<String>,
}

impl Default for PerformanceConfig {
    fn default() -> Self {
        Self {
            cache_ttl_seconds: 300,
            template_cache_size: 100,
            query_cache_enabled: true,
            hot_reload_enabled: cfg!(debug_assertions),
            benchmark_enabled: false,
            memory_monitoring: MemoryMonitorConfig {
                enabled: true,
                check_interval_seconds: 60,
                warning_threshold_percent: 85,
                critical_threshold_percent: 95,
            },
            intelligence: IntelligenceConfig {
                error_pattern_detection: true,
                anomaly_detection: true,
                security_pattern_matching: true,
                provider_plugin: None,
            },
        }
    }
}
```

## Locale Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct LocaleConfig {
    pub detection_method: LocaleMethod,
    pub default_locale: String,
    pub supported_locales: Vec<String>,
    pub fallback_chain: Vec<String>,
}

#[derive(Debug, Clone, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum LocaleMethod {
    Header,     // Accept-Language header
    Path,       // /en/page format
    Subdomain,  // en.site.com format
}

impl Default for LocaleConfig {
    fn default() -> Self {
        Self {
            detection_method: LocaleMethod::Path,
            default_locale: "en_US".to_string(),
            supported_locales: vec!["en_US".to_string()],
            fallback_chain: vec!["en_US".to_string()],
        }
    }
}
```

## Plugin Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct PluginConfig {
    pub enabled: bool,
    pub auto_load: bool,
    pub socket_timeout_ms: u64,
    pub max_message_size: usize,
    pub rust_plugins: RustPluginConfig,
    pub lua_plugins: LuaPluginConfig,
}

#[derive(Debug, Clone, Deserialize)]
pub struct RustPluginConfig {
    pub enabled: bool,
    pub process_limit: usize,
    pub memory_limit_mb: u64,
}

#[derive(Debug, Clone, Deserialize)]
pub struct LuaPluginConfig {
    pub enabled: bool,
    pub sandbox_enabled: bool,
    pub execution_timeout_ms: u64,
}

impl Default for PluginConfig {
    fn default() -> Self {
        Self {
            enabled: true,
            auto_load: true,
            socket_timeout_ms: 5000,
            max_message_size: 1_048_576, // 1MB
            rust_plugins: RustPluginConfig {
                enabled: true,
                process_limit: 10,
                memory_limit_mb: 512,
            },
            lua_plugins: LuaPluginConfig {
                enabled: true,
                sandbox_enabled: true,
                execution_timeout_ms: 200,
            },
        }
    }
}
```

## Firewall Configuration

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct FirewallConfig {
    pub enabled: bool,
    pub default_rules: HashMap<String, String>,
    pub http_rules: HttpRuleConfig,
}

#[derive(Debug, Clone, Deserialize)]
pub struct HttpRuleConfig {
    pub max_fields: u32,
    pub max_field_size: String,
    pub require_csrf: bool,
    pub rate_limit: String,
}

impl Default for FirewallConfig {
    fn default() -> Self {
        let mut default_rules = HashMap::new();
        default_rules.insert("text_field".to_string(), 
            "strip_html,max_length:10000,no_script_tags".to_string());
        default_rules.insert("email_field".to_string(), 
            "validate_email,strip_html".to_string());
        
        Self {
            enabled: true,
            default_rules,
            http_rules: HttpRuleConfig {
                max_fields: 50,
                max_field_size: "1MB".to_string(),
                require_csrf: true,
                rate_limit: "10/minute".to_string(),
            },
        }
    }
}
```

## Configuration Loading

```rust
use std::fs;
use toml;

impl ReedConfig {
    /// Load configuration from file with defaults
    pub fn from_file(path: &Path) -> Result<Self> {
        let contents = fs::read_to_string(path)
            .map_err(|e| ReedError::Configuration(format!("Cannot read config: {}", e)))?;
        
        let config: ReedConfig = toml::from_str(&contents)
            .map_err(|e| ReedError::Configuration(format!("Invalid TOML: {}", e)))?;
        
        config.validate()?;
        Ok(config)
    }
    
    /// Validate configuration consistency
    pub fn validate(&self) -> Result<()> {
        // Validate paths exist
        if !self.paths.config_dir.exists() {
            return Err(ReedError::Configuration(
                "Config directory does not exist".to_string()
            ));
        }
        
        // Validate locale configuration
        if !self.locale.supported_locales.contains(&self.locale.default_locale) {
            return Err(ReedError::Configuration(
                "Default locale not in supported locales".to_string()
            ));
        }
        
        // Validate memory thresholds
        if self.performance.memory_monitoring.warning_threshold_percent >= 
           self.performance.memory_monitoring.critical_threshold_percent {
            return Err(ReedError::Configuration(
                "Warning threshold must be less than critical".to_string()
            ));
        }
        
        Ok(())
    }
}
```

## Summary

Diese Configuration Types bieten:
- **Klare Struktur** mit logischen Gruppierungen
- **Sinnvolle Defaults** für alle Werte
- **Validierung** der Konfiguration
- **Intelligence Integration** vorbereitet aber optional
- **KISS-Prinzip** ohne unnötige Komplexität

Alle Konfigurationen sind TOML-kompatibel und selbstdokumentierend.