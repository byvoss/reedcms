# Core Type Definitions

## Overview

Fundamentale Datentypen für ReedCMS. Jeder Typ hat eine klare, einzelne Verantwortung.

## Base Types

```rust
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Serialize, Deserialize};

/// Unique identifier for all entities in the system
pub type EntityId = Uuid;

/// Semantic name with $ prefix for human-readable references
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct SemanticName(String);

impl SemanticName {
    pub fn new(name: &str) -> Result<Self, ValidationError> {
        if !name.starts_with('$') {
            return Err(ValidationError::InvalidSemanticName);
        }
        
        let clean_name = &name[1..];
        if !clean_name.chars().all(|c| c.is_ascii_lowercase() || c == '_') {
            return Err(ValidationError::InvalidCharacters);
        }
        
        Ok(SemanticName(name.to_string()))
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

/// Language/Locale identifier following ISO standards
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct Locale(String);

impl Locale {
    pub fn new(code: &str) -> Result<Self, ValidationError> {
        // Support both "de_DE" and "eu" (Basque) formats
        let valid = match code.len() {
            2 => code.chars().all(|c| c.is_ascii_lowercase()),
            5 => code.chars().nth(2) == Some('_') &&
                 code[0..2].chars().all(|c| c.is_ascii_lowercase()) &&
                 code[3..5].chars().all(|c| c.is_ascii_uppercase()),
            _ => false,
        };
        
        if !valid {
            return Err(ValidationError::InvalidLocale);
        }
        
        Ok(Locale(code.to_string()))
    }
}
```

## Entity System

```rust
/// Core entity type - everything in UCG is an entity
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Entity {
    pub id: EntityId,
    pub entity_type: EntityType,
    pub semantic_name: Option<SemanticName>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub created_by: String,
    pub data: serde_json::Value,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum EntityType {
    Snippet,
    Theme,
    Page,
    User,
    Plugin,
    Custom(String),
}

/// Association between entities in the UCG
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Association {
    pub id: EntityId,
    pub parent_id: EntityId,
    pub child_id: EntityId,
    pub association_type: AssociationType,
    pub weight: i32,  // For ordering
    pub path: String, // UCG path like "content.1.1"
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum AssociationType {
    Contains,     // Parent contains child
    References,   // Soft reference
    Extends,      // Theme inheritance
    Custom(String),
}
```

## Content Types

```rust
/// Snippet instance with typed content
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Snippet {
    pub id: EntityId,
    pub snippet_type: String,  // References registry definition
    pub semantic_name: SemanticName,
    pub fields: HashMap<String, FieldValue>,
    pub metadata: SnippetMetadata,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SnippetMetadata {
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub created_by: String,
    pub updated_by: String,
    pub version: u32,
    pub status: ContentStatus,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum ContentStatus {
    Draft,
    Published,
    Archived,
    Deleted,
}

/// Field values with type safety
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", content = "value")]
pub enum FieldValue {
    Text(String),
    Number(f64),
    Boolean(bool),
    Date(DateTime<Utc>),
    Reference(EntityId),
    Json(serde_json::Value),
}
```

## Theme System Types

```rust
/// Theme with scope-based organization
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Theme {
    pub name: String,  // e.g., "corporate.berlin.christmas"
    pub parent: Option<String>,
    pub context: ThemeContext,
    pub active: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ThemeContext {
    pub context_type: String,  // "location", "season", "audience"
    pub context_value: String, // "berlin", "christmas", "b2b"
}

/// Theme chain for EPC resolution
pub type ThemeChain = Vec<String>;
```

## Intelligence Interface Types

```rust
/// Prepared interface for intelligent error handling
#[async_trait]
pub trait IntelligenceProvider: Send + Sync {
    /// Analyze error patterns and suggest solutions
    async fn analyze_error(&self, error: &ReedError, context: &ErrorContext) 
        -> Option<ErrorSuggestion>;
    
    /// Detect performance anomalies
    async fn detect_anomaly(&self, metrics: &PerformanceMetrics) 
        -> Option<AnomalyReport>;
    
    /// Pattern matching for security threats
    async fn analyze_security_pattern(&self, request: &SecurityContext) 
        -> Option<ThreatAssessment>;
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ErrorContext {
    pub operation: String,
    pub user_role: UserRole,
    pub recent_actions: Vec<String>,
    pub system_state: SystemState,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ErrorSuggestion {
    pub description: String,
    pub confidence: f32,  // 0.0 to 1.0
    pub recovery_steps: Vec<String>,
    pub similar_errors: Vec<String>,
}

/// Deterministic pattern matching results
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AnomalyReport {
    pub metric_name: String,
    pub expected_range: (f64, f64),
    pub actual_value: f64,
    pub severity: AnomalySeverity,
    pub suggested_action: String,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum AnomalySeverity {
    Low,
    Medium,
    High,
    Critical,
}
```

## Performance Types

```rust
/// Core performance metrics
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PerformanceMetrics {
    pub ucg_query_time: Duration,
    pub content_load_time: Duration,
    pub template_render_time: Duration,
    pub total_response_time: Duration,
    pub memory_usage: MemoryUsage,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MemoryUsage {
    pub redis_mb: f64,
    pub postgres_connections: u32,
    pub process_memory_mb: f64,
}
```

## Configuration Types

```rust
/// System-wide configuration
#[derive(Debug, Clone, Deserialize)]
pub struct ReedConfig {
    pub database: DatabaseConfig,
    pub redis: RedisConfig,
    pub server: ServerConfig,
    pub security: SecurityConfig,
    pub intelligence: IntelligenceConfig,
}

#[derive(Debug, Clone, Deserialize)]
pub struct IntelligenceConfig {
    pub provider: Option<String>,  // Plugin name, None for core
    pub error_patterns_enabled: bool,
    pub anomaly_detection_enabled: bool,
    pub security_patterns_enabled: bool,
}
```

## Validation Types

```rust
/// Validation errors with clear messages
#[derive(Debug, thiserror::Error)]
pub enum ValidationError {
    #[error("Semantic name must start with $")]
    InvalidSemanticName,
    
    #[error("Invalid characters in name")]
    InvalidCharacters,
    
    #[error("Invalid locale format")]
    InvalidLocale,
    
    #[error("Field validation failed: {field}")]
    FieldValidation { field: String, reason: String },
}
```

## Summary

Diese Core Types bilden das Fundament:
- **Entity/Association** für UCG
- **Snippet/Theme** für Content
- **IntelligenceProvider** für deterministische Patterns
- **Clear validation** für alle Inputs

Jeder Typ hat genau eine Verantwortung und fügt sich nahtlos ins Gesamtsystem.