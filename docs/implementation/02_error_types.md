# Error Types and Result Handling

## Overview

Einheitliches Error Handling mit klaren Kategorien und deterministischen Recovery-Hinweisen.

## Core Error Type

```rust
use thiserror::Error;
use uuid::Uuid;

/// Main error type for ReedCMS
#[derive(Debug, Error)]
pub enum ReedError {
    // Database errors
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Redis error: {0}")]
    Redis(#[from] redis::RedisError),
    
    // Entity errors
    #[error("Entity not found: {id}")]
    EntityNotFound { id: Uuid },
    
    #[error("Entity already exists: {semantic_name}")]
    EntityExists { semantic_name: String },
    
    // UCG errors
    #[error("Invalid UCG path: {path}")]
    InvalidPath { path: String },
    
    #[error("UCG integrity error: {reason}")]
    UcgIntegrity { reason: String },
    
    // Template errors
    #[error("Template error: {0}")]
    Template(#[from] tera::Error),
    
    #[error("Template not found: {name} in theme chain")]
    TemplateNotFound { name: String },
    
    // Validation errors
    #[error("Validation failed: {0}")]
    Validation(#[from] ValidationError),
    
    #[error("Content rejected by firewall: {rule}")]
    ContentFirewall { rule: String, field: String },
    
    // Permission errors
    #[error("Permission denied: {action}")]
    PermissionDenied { action: String },
    
    #[error("Authentication required")]
    AuthenticationRequired,
    
    // Configuration errors
    #[error("Configuration error: {0}")]
    Configuration(String),
    
    #[error("Missing required configuration: {key}")]
    MissingConfig { key: String },
    
    // Plugin errors
    #[error("Plugin error: {plugin} - {message}")]
    Plugin { plugin: String, message: String },
    
    #[error("Plugin not found: {name}")]
    PluginNotFound { name: String },
    
    // System errors
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Internal error: {0}")]
    Internal(String),
}
```

## Result Type Alias

```rust
/// Standard Result type for ReedCMS operations
pub type Result<T> = std::result::Result<T, ReedError>;

/// Result with explicit error type
pub type ResultWith<T, E> = std::result::Result<T, E>;
```

## Error Context and Recovery

```rust
/// Extended error information for intelligent handling
#[derive(Debug, Clone)]
pub struct ErrorDetails {
    pub error: ReedError,
    pub context: ErrorContext,
    pub recovery_hint: Option<RecoveryHint>,
    pub occurred_at: DateTime<Utc>,
}

/// Recovery hints for deterministic error handling
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RecoveryHint {
    pub category: RecoveryCategory,
    pub steps: Vec<String>,
    pub can_retry: bool,
    pub retry_after: Option<Duration>,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum RecoveryCategory {
    RetryOperation,      // Temporary failure
    CheckConfiguration,  // Config issue
    FixData,            // Data integrity
    ContactSupport,     // Unrecoverable
    UpdatePermissions,  // Access issue
}

impl ReedError {
    /// Get deterministic recovery hint based on error type
    pub fn recovery_hint(&self) -> Option<RecoveryHint> {
        match self {
            ReedError::Redis(_) => Some(RecoveryHint {
                category: RecoveryCategory::RetryOperation,
                steps: vec![
                    "Check Redis connection".to_string(),
                    "Verify Redis is running: redis-cli ping".to_string(),
                    "System will use PostgreSQL fallback".to_string(),
                ],
                can_retry: true,
                retry_after: Some(Duration::from_secs(5)),
            }),
            
            ReedError::EntityNotFound { id } => Some(RecoveryHint {
                category: RecoveryCategory::FixData,
                steps: vec![
                    format!("Entity {} does not exist", id),
                    "Run: reed whois {} --debug".to_string(),
                    "Check for typos in semantic name".to_string(),
                ],
                can_retry: false,
                retry_after: None,
            }),
            
            ReedError::ContentFirewall { rule, field } => Some(RecoveryHint {
                category: RecoveryCategory::FixData,
                steps: vec![
                    format!("Content in field '{}' violates rule '{}'", field, rule),
                    "Check firewall rules: reed fw rules show".to_string(),
                    "Modify content to comply with security rules".to_string(),
                ],
                can_retry: true,
                retry_after: None,
            }),
            
            ReedError::MissingConfig { key } => Some(RecoveryHint {
                category: RecoveryCategory::CheckConfiguration,
                steps: vec![
                    format!("Add '{}' to configuration", key),
                    "Check config/reed.toml".to_string(),
                    "Use: reed config validate".to_string(),
                ],
                can_retry: false,
                retry_after: None,
            }),
            
            _ => None,
        }
    }
}
```

## Error Pattern Recognition

```rust
/// Pattern matcher for known error scenarios
pub struct ErrorPatternMatcher {
    patterns: Vec<ErrorPattern>,
}

#[derive(Debug, Clone)]
pub struct ErrorPattern {
    pub name: String,
    pub matches: Box<dyn Fn(&ReedError) -> bool + Send + Sync>,
    pub suggestion: ErrorSuggestion,
}

impl ErrorPatternMatcher {
    pub fn new() -> Self {
        let mut matcher = Self {
            patterns: Vec::new(),
        };
        
        // Register known patterns
        matcher.register_connection_pool_exhaustion();
        matcher.register_memory_pressure();
        matcher.register_template_loop();
        
        matcher
    }
    
    fn register_connection_pool_exhaustion(&mut self) {
        let pattern = ErrorPattern {
            name: "Connection Pool Exhausted".to_string(),
            matches: Box::new(|error| {
                matches!(error, ReedError::Database(e) 
                    if e.to_string().contains("connection pool"))
            }),
            suggestion: ErrorSuggestion {
                description: "Database connection pool is exhausted".to_string(),
                confidence: 0.95,
                recovery_steps: vec![
                    "Increase max_connections in config/database.toml".to_string(),
                    "Current setting may be too low for load".to_string(),
                    "Consider connection pooling with PgBouncer".to_string(),
                ],
                similar_errors: vec!["timeout", "too many connections"].into(),
            },
        };
        self.patterns.push(pattern);
    }
    
    pub fn analyze(&self, error: &ReedError) -> Option<&ErrorSuggestion> {
        self.patterns
            .iter()
            .find(|p| (p.matches)(error))
            .map(|p| &p.suggestion)
    }
}
```

## HTTP Error Responses

```rust
use axum::response::{IntoResponse, Response};
use axum::http::StatusCode;

impl IntoResponse for ReedError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            ReedError::EntityNotFound { .. } => {
                (StatusCode::NOT_FOUND, self.to_string())
            }
            ReedError::Validation(_) | ReedError::ContentFirewall { .. } => {
                (StatusCode::BAD_REQUEST, self.to_string())
            }
            ReedError::PermissionDenied { .. } => {
                (StatusCode::FORBIDDEN, "Access denied".to_string())
            }
            ReedError::AuthenticationRequired => {
                (StatusCode::UNAUTHORIZED, "Authentication required".to_string())
            }
            ReedError::Database(_) | ReedError::Redis(_) => {
                (StatusCode::SERVICE_UNAVAILABLE, "Service temporarily unavailable".to_string())
            }
            _ => {
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal error".to_string())
            }
        };
        
        (status, message).into_response()
    }
}
```

## Error Logging

```rust
/// Structured error logging with context
pub trait ErrorLogging {
    fn log_with_context(&self, context: &ErrorContext);
}

impl ErrorLogging for ReedError {
    fn log_with_context(&self, context: &ErrorContext) {
        let recovery = self.recovery_hint();
        
        match self {
            ReedError::Database(_) | ReedError::Redis(_) => {
                tracing::error!(
                    error = %self,
                    operation = %context.operation,
                    recovery = ?recovery,
                    "Infrastructure error"
                );
            }
            ReedError::ContentFirewall { .. } => {
                tracing::warn!(
                    error = %self,
                    user = %context.user_role,
                    "Content rejected by firewall"
                );
            }
            _ => {
                tracing::error!(
                    error = %self,
                    context = ?context,
                    "Operation failed"
                );
            }
        }
    }
}
```

## Summary

Dieses Error System bietet:
- **Klare Kategorisierung** aller Fehlertypen
- **Deterministische Recovery Hints** für jeden Fehler
- **Pattern Recognition** für bekannte Probleme
- **Strukturiertes Logging** mit Kontext
- **HTTP-kompatible** Fehlerantworten

Die IntelligenceProvider-Integration ist vorbereitet aber nicht implementiert.