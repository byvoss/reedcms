# Input Validation

## Overview

Input Validation Implementation für ReedCMS. Comprehensive Validation Framework mit Type Safety und Security Focus.

## Validation Core

```rust
use std::sync::Arc;
use serde::{Serialize, Deserialize};
use validator::{Validate, ValidationError};

/// Main validation service
pub struct ValidationService {
    validators: Arc<ValidatorRegistry>,
    sanitizers: Arc<SanitizerRegistry>,
    rules: Arc<ValidationRules>,
    config: ValidationConfig,
}

impl ValidationService {
    pub fn new(config: ValidationConfig) -> Self {
        let validators = Arc::new(ValidatorRegistry::new());
        let sanitizers = Arc::new(SanitizerRegistry::new());
        let rules = Arc::new(ValidationRules::default());
        
        Self {
            validators,
            sanitizers,
            rules,
            config,
        }
    }
    
    /// Validate input data
    pub async fn validate<T>(&self, input: &T) -> Result<ValidationResult>
    where
        T: Validate + Serialize,
    {
        let mut result = ValidationResult::new();
        
        // Structural validation
        if let Err(errors) = input.validate() {
            for (field, field_errors) in errors.field_errors() {
                for error in field_errors {
                    result.add_error(field, error.code.as_ref().unwrap_or(&"invalid"));
                }
            }
        }
        
        // Custom validation rules
        let json_value = serde_json::to_value(input)?;
        self.apply_custom_rules(&json_value, &mut result).await?;
        
        // Security validation
        if self.config.enable_security_checks {
            self.apply_security_rules(&json_value, &mut result).await?;
        }
        
        Ok(result)
    }
    
    /// Validate and sanitize
    pub async fn validate_and_sanitize<T>(&self, input: T) -> Result<T>
    where
        T: Validate + Serialize + for<'de> Deserialize<'de>,
    {
        // Validate first
        self.validate(&input).await?.ensure_valid()?;
        
        // Sanitize
        let sanitized = self.sanitize(input).await?;
        
        // Validate again after sanitization
        self.validate(&sanitized).await?.ensure_valid()?;
        
        Ok(sanitized)
    }
    
    /// Sanitize input
    pub async fn sanitize<T>(&self, input: T) -> Result<T>
    where
        T: Serialize + for<'de> Deserialize<'de>,
    {
        let mut value = serde_json::to_value(input)?;
        
        // Apply sanitizers
        self.sanitizers.apply_all(&mut value).await?;
        
        // Convert back
        let sanitized = serde_json::from_value(value)?;
        
        Ok(sanitized)
    }
    
    /// Apply custom validation rules
    async fn apply_custom_rules(
        &self,
        value: &serde_json::Value,
        result: &mut ValidationResult,
    ) -> Result<()> {
        for (path, rules) in &self.rules.field_rules {
            if let Some(field_value) = self.get_field_value(value, path) {
                for rule in rules {
                    if !rule.validate(field_value).await? {
                        result.add_error(path, &rule.error_code());
                    }
                }
            }
        }
        
        Ok(())
    }
    
    /// Apply security validation rules
    async fn apply_security_rules(
        &self,
        value: &serde_json::Value,
        result: &mut ValidationResult,
    ) -> Result<()> {
        // Check for SQL injection patterns
        if let Some(sql_fields) = self.find_sql_injection_attempts(value) {
            for field in sql_fields {
                result.add_error(&field, "potential_sql_injection");
            }
        }
        
        // Check for XSS patterns
        if let Some(xss_fields) = self.find_xss_attempts(value) {
            for field in xss_fields {
                result.add_error(&field, "potential_xss");
            }
        }
        
        // Check for path traversal
        if let Some(path_fields) = self.find_path_traversal_attempts(value) {
            for field in path_fields {
                result.add_error(&field, "potential_path_traversal");
            }
        }
        
        Ok(())
    }
}

/// Validation result
#[derive(Debug, Default)]
pub struct ValidationResult {
    errors: HashMap<String, Vec<String>>,
    warnings: HashMap<String, Vec<String>>,
}

impl ValidationResult {
    pub fn new() -> Self {
        Self::default()
    }
    
    pub fn is_valid(&self) -> bool {
        self.errors.is_empty()
    }
    
    pub fn add_error(&mut self, field: &str, code: &str) {
        self.errors
            .entry(field.to_string())
            .or_default()
            .push(code.to_string());
    }
    
    pub fn add_warning(&mut self, field: &str, code: &str) {
        self.warnings
            .entry(field.to_string())
            .or_default()
            .push(code.to_string());
    }
    
    pub fn ensure_valid(self) -> Result<()> {
        if !self.is_valid() {
            return Err(ValidationError::Invalid(self));
        }
        Ok(())
    }
}
```

## Validator Registry

```rust
/// Registry of custom validators
pub struct ValidatorRegistry {
    validators: RwLock<HashMap<String, Box<dyn Validator>>>,
}

impl ValidatorRegistry {
    pub fn new() -> Self {
        let mut registry = Self {
            validators: RwLock::new(HashMap::new()),
        };
        
        // Register default validators
        registry.register_defaults();
        
        registry
    }
    
    /// Register validator
    pub fn register(&self, name: String, validator: Box<dyn Validator>) {
        self.validators.write().unwrap().insert(name, validator);
    }
    
    /// Get validator
    pub fn get(&self, name: &str) -> Option<Box<dyn Validator>> {
        self.validators.read().unwrap().get(name).cloned()
    }
    
    /// Register default validators
    fn register_defaults(&mut self) {
        // Email validator
        self.register(
            "email".to_string(),
            Box::new(EmailValidator::new()),
        );
        
        // URL validator
        self.register(
            "url".to_string(),
            Box::new(UrlValidator::new()),
        );
        
        // Phone validator
        self.register(
            "phone".to_string(),
            Box::new(PhoneValidator::new()),
        );
        
        // Username validator
        self.register(
            "username".to_string(),
            Box::new(UsernameValidator::new()),
        );
        
        // Password strength validator
        self.register(
            "password_strength".to_string(),
            Box::new(PasswordStrengthValidator::new()),
        );
        
        // Credit card validator
        self.register(
            "credit_card".to_string(),
            Box::new(CreditCardValidator::new()),
        );
    }
}

/// Validator trait
#[async_trait]
pub trait Validator: Send + Sync {
    async fn validate(&self, value: &serde_json::Value) -> Result<bool>;
    fn error_message(&self) -> String;
}

/// Email validator
struct EmailValidator {
    regex: regex::Regex,
}

impl EmailValidator {
    fn new() -> Self {
        Self {
            regex: regex::Regex::new(
                r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
            ).unwrap(),
        }
    }
}

#[async_trait]
impl Validator for EmailValidator {
    async fn validate(&self, value: &serde_json::Value) -> Result<bool> {
        if let Some(email) = value.as_str() {
            Ok(self.regex.is_match(email))
        } else {
            Ok(false)
        }
    }
    
    fn error_message(&self) -> String {
        "Invalid email address".to_string()
    }
}

/// Password strength validator
struct PasswordStrengthValidator {
    min_length: usize,
    require_uppercase: bool,
    require_lowercase: bool,
    require_numbers: bool,
    require_special: bool,
}

impl PasswordStrengthValidator {
    fn new() -> Self {
        Self {
            min_length: 8,
            require_uppercase: true,
            require_lowercase: true,
            require_numbers: true,
            require_special: true,
        }
    }
}

#[async_trait]
impl Validator for PasswordStrengthValidator {
    async fn validate(&self, value: &serde_json::Value) -> Result<bool> {
        if let Some(password) = value.as_str() {
            if password.len() < self.min_length {
                return Ok(false);
            }
            
            if self.require_uppercase && !password.chars().any(|c| c.is_uppercase()) {
                return Ok(false);
            }
            
            if self.require_lowercase && !password.chars().any(|c| c.is_lowercase()) {
                return Ok(false);
            }
            
            if self.require_numbers && !password.chars().any(|c| c.is_numeric()) {
                return Ok(false);
            }
            
            if self.require_special && !password.chars().any(|c| "!@#$%^&*()_+-=[]{}|;:,.<>?".contains(c)) {
                return Ok(false);
            }
            
            Ok(true)
        } else {
            Ok(false)
        }
    }
    
    fn error_message(&self) -> String {
        "Password does not meet strength requirements".to_string()
    }
}
```

## Sanitizer Registry

```rust
/// Registry of sanitizers
pub struct SanitizerRegistry {
    sanitizers: RwLock<HashMap<String, Box<dyn Sanitizer>>>,
}

impl SanitizerRegistry {
    pub fn new() -> Self {
        let mut registry = Self {
            sanitizers: RwLock::new(HashMap::new()),
        };
        
        registry.register_defaults();
        
        registry
    }
    
    /// Apply all sanitizers
    pub async fn apply_all(&self, value: &mut serde_json::Value) -> Result<()> {
        self.apply_recursive(value).await
    }
    
    /// Apply sanitizers recursively
    async fn apply_recursive(&self, value: &mut serde_json::Value) -> Result<()> {
        match value {
            serde_json::Value::String(s) => {
                *s = self.sanitize_string(s).await?;
            }
            serde_json::Value::Object(map) => {
                for (_, v) in map.iter_mut() {
                    self.apply_recursive(v).await?;
                }
            }
            serde_json::Value::Array(arr) => {
                for v in arr.iter_mut() {
                    self.apply_recursive(v).await?;
                }
            }
            _ => {}
        }
        
        Ok(())
    }
    
    /// Sanitize string value
    async fn sanitize_string(&self, value: &str) -> Result<String> {
        let mut result = value.to_string();
        
        // Apply sanitizers
        let sanitizers = self.sanitizers.read().unwrap();
        for sanitizer in sanitizers.values() {
            result = sanitizer.sanitize(&result).await?;
        }
        
        Ok(result)
    }
    
    /// Register default sanitizers
    fn register_defaults(&mut self) {
        // HTML sanitizer
        self.sanitizers.write().unwrap().insert(
            "html".to_string(),
            Box::new(HtmlSanitizer::new()),
        );
        
        // SQL sanitizer
        self.sanitizers.write().unwrap().insert(
            "sql".to_string(),
            Box::new(SqlSanitizer::new()),
        );
        
        // Script sanitizer
        self.sanitizers.write().unwrap().insert(
            "script".to_string(),
            Box::new(ScriptSanitizer::new()),
        );
        
        // Trim whitespace
        self.sanitizers.write().unwrap().insert(
            "trim".to_string(),
            Box::new(TrimSanitizer::new()),
        );
    }
}

/// Sanitizer trait
#[async_trait]
pub trait Sanitizer: Send + Sync {
    async fn sanitize(&self, value: &str) -> Result<String>;
}

/// HTML sanitizer
struct HtmlSanitizer {
    allowed_tags: HashSet<String>,
    allowed_attrs: HashMap<String, HashSet<String>>,
}

impl HtmlSanitizer {
    fn new() -> Self {
        let mut allowed_tags = HashSet::new();
        allowed_tags.insert("p".to_string());
        allowed_tags.insert("br".to_string());
        allowed_tags.insert("strong".to_string());
        allowed_tags.insert("em".to_string());
        allowed_tags.insert("u".to_string());
        allowed_tags.insert("a".to_string());
        allowed_tags.insert("ul".to_string());
        allowed_tags.insert("ol".to_string());
        allowed_tags.insert("li".to_string());
        
        let mut allowed_attrs = HashMap::new();
        let mut a_attrs = HashSet::new();
        a_attrs.insert("href".to_string());
        a_attrs.insert("title".to_string());
        allowed_attrs.insert("a".to_string(), a_attrs);
        
        Self {
            allowed_tags,
            allowed_attrs,
        }
    }
}

#[async_trait]
impl Sanitizer for HtmlSanitizer {
    async fn sanitize(&self, value: &str) -> Result<String> {
        use ammonia::Builder;
        
        let mut builder = Builder::default();
        builder
            .tags(self.allowed_tags.clone())
            .tag_attributes(self.allowed_attrs.clone())
            .url_schemes(hashset!["http", "https", "mailto"])
            .link_rel(Some("nofollow noopener noreferrer"));
        
        Ok(builder.clean(value).to_string())
    }
}

/// SQL sanitizer
struct SqlSanitizer;

impl SqlSanitizer {
    fn new() -> Self {
        Self
    }
}

#[async_trait]
impl Sanitizer for SqlSanitizer {
    async fn sanitize(&self, value: &str) -> Result<String> {
        // Basic SQL injection prevention
        let dangerous_patterns = [
            "--", "/*", "*/", "'", "\"", ";",
            "xp_", "sp_", "exec", "execute",
            "drop", "delete", "update", "insert",
            "union", "select",
        ];
        
        let mut result = value.to_string();
        for pattern in &dangerous_patterns {
            result = result.replace(pattern, "");
        }
        
        Ok(result)
    }
}
```

## Content Validation

```rust
/// Content-specific validation
pub struct ContentValidator {
    schema_registry: Arc<SchemaRegistry>,
    field_validators: Arc<FieldValidatorRegistry>,
}

impl ContentValidator {
    /// Validate content against schema
    pub async fn validate_content(
        &self,
        content_type: &str,
        data: &serde_json::Value,
    ) -> Result<ValidationResult> {
        let mut result = ValidationResult::new();
        
        // Get schema
        let schema = self.schema_registry
            .get_schema(content_type)
            .await?
            .ok_or_else(|| ValidationError::SchemaNotFound(content_type.to_string()))?;
        
        // Validate required fields
        for field in &schema.required_fields {
            if !data.get(field).is_some() {
                result.add_error(field, "required");
            }
        }
        
        // Validate each field
        if let Some(obj) = data.as_object() {
            for (field_name, field_value) in obj {
                if let Some(field_def) = schema.fields.get(field_name) {
                    self.validate_field(
                        field_name,
                        field_value,
                        field_def,
                        &mut result,
                    ).await?;
                } else if !schema.allow_extra_fields {
                    result.add_error(field_name, "unknown_field");
                }
            }
        }
        
        Ok(result)
    }
    
    /// Validate single field
    async fn validate_field(
        &self,
        field_name: &str,
        field_value: &serde_json::Value,
        field_def: &FieldDefinition,
        result: &mut ValidationResult,
    ) -> Result<()> {
        // Type validation
        if !self.validate_field_type(field_value, &field_def.field_type) {
            result.add_error(field_name, "invalid_type");
            return Ok(());
        }
        
        // Constraints validation
        for constraint in &field_def.constraints {
            match constraint {
                FieldConstraint::MinLength(min) => {
                    if let Some(s) = field_value.as_str() {
                        if s.len() < *min {
                            result.add_error(field_name, "min_length");
                        }
                    }
                }
                FieldConstraint::MaxLength(max) => {
                    if let Some(s) = field_value.as_str() {
                        if s.len() > *max {
                            result.add_error(field_name, "max_length");
                        }
                    }
                }
                FieldConstraint::Pattern(pattern) => {
                    if let Some(s) = field_value.as_str() {
                        let re = regex::Regex::new(pattern)?;
                        if !re.is_match(s) {
                            result.add_error(field_name, "pattern");
                        }
                    }
                }
                FieldConstraint::Min(min) => {
                    if let Some(n) = field_value.as_f64() {
                        if n < *min {
                            result.add_error(field_name, "min_value");
                        }
                    }
                }
                FieldConstraint::Max(max) => {
                    if let Some(n) = field_value.as_f64() {
                        if n > *max {
                            result.add_error(field_name, "max_value");
                        }
                    }
                }
                FieldConstraint::Enum(values) => {
                    if let Some(s) = field_value.as_str() {
                        if !values.contains(&s.to_string()) {
                            result.add_error(field_name, "invalid_enum_value");
                        }
                    }
                }
                FieldConstraint::Custom(validator_name) => {
                    if let Some(validator) = self.field_validators.get(validator_name) {
                        if !validator.validate(field_value).await? {
                            result.add_error(field_name, validator_name);
                        }
                    }
                }
            }
        }
        
        Ok(())
    }
    
    /// Validate field type
    fn validate_field_type(&self, value: &serde_json::Value, field_type: &FieldType) -> bool {
        match field_type {
            FieldType::String => value.is_string(),
            FieldType::Number => value.is_number(),
            FieldType::Boolean => value.is_boolean(),
            FieldType::Array => value.is_array(),
            FieldType::Object => value.is_object(),
            FieldType::Date => {
                if let Some(s) = value.as_str() {
                    chrono::DateTime::parse_from_rfc3339(s).is_ok()
                } else {
                    false
                }
            }
            FieldType::Email => {
                if let Some(s) = value.as_str() {
                    EmailValidator::new().validate_sync(s)
                } else {
                    false
                }
            }
            FieldType::Url => {
                if let Some(s) = value.as_str() {
                    url::Url::parse(s).is_ok()
                } else {
                    false
                }
            }
        }
    }
}

/// Field definition
#[derive(Debug, Clone)]
pub struct FieldDefinition {
    pub field_type: FieldType,
    pub constraints: Vec<FieldConstraint>,
    pub required: bool,
    pub description: Option<String>,
}

#[derive(Debug, Clone)]
pub enum FieldType {
    String,
    Number,
    Boolean,
    Array,
    Object,
    Date,
    Email,
    Url,
}

#[derive(Debug, Clone)]
pub enum FieldConstraint {
    MinLength(usize),
    MaxLength(usize),
    Pattern(String),
    Min(f64),
    Max(f64),
    Enum(Vec<String>),
    Custom(String),
}
```

## Security Validation

```rust
/// Security-focused validation
pub struct SecurityValidator {
    patterns: SecurityPatterns,
    config: SecurityValidationConfig,
}

impl SecurityValidator {
    pub fn new(config: SecurityValidationConfig) -> Self {
        Self {
            patterns: SecurityPatterns::default(),
            config,
        }
    }
    
    /// Check for SQL injection
    pub fn check_sql_injection(&self, value: &str) -> bool {
        for pattern in &self.patterns.sql_injection {
            if pattern.is_match(value) {
                return true;
            }
        }
        false
    }
    
    /// Check for XSS
    pub fn check_xss(&self, value: &str) -> bool {
        for pattern in &self.patterns.xss {
            if pattern.is_match(value) {
                return true;
            }
        }
        false
    }
    
    /// Check for path traversal
    pub fn check_path_traversal(&self, value: &str) -> bool {
        for pattern in &self.patterns.path_traversal {
            if pattern.is_match(value) {
                return true;
            }
        }
        false
    }
    
    /// Check for command injection
    pub fn check_command_injection(&self, value: &str) -> bool {
        for pattern in &self.patterns.command_injection {
            if pattern.is_match(value) {
                return true;
            }
        }
        false
    }
    
    /// Check for LDAP injection
    pub fn check_ldap_injection(&self, value: &str) -> bool {
        for pattern in &self.patterns.ldap_injection {
            if pattern.is_match(value) {
                return true;
            }
        }
        false
    }
}

/// Security patterns
#[derive(Debug)]
struct SecurityPatterns {
    sql_injection: Vec<regex::Regex>,
    xss: Vec<regex::Regex>,
    path_traversal: Vec<regex::Regex>,
    command_injection: Vec<regex::Regex>,
    ldap_injection: Vec<regex::Regex>,
}

impl Default for SecurityPatterns {
    fn default() -> Self {
        Self {
            sql_injection: vec![
                regex::Regex::new(r"(?i)(union\s+select|select\s+\*|drop\s+table|insert\s+into|delete\s+from|update\s+set)").unwrap(),
                regex::Regex::new(r"(?i)(exec\s*\(|execute\s*\(|xp_cmdshell)").unwrap(),
                regex::Regex::new(r"(\-\-|/\*|\*/|;|\||&&)").unwrap(),
            ],
            xss: vec![
                regex::Regex::new(r"(?i)<\s*script").unwrap(),
                regex::Regex::new(r"(?i)on\w+\s*=").unwrap(),
                regex::Regex::new(r"(?i)javascript\s*:").unwrap(),
                regex::Regex::new(r"(?i)<\s*iframe").unwrap(),
                regex::Regex::new(r"(?i)<\s*object").unwrap(),
            ],
            path_traversal: vec![
                regex::Regex::new(r"\.\.[\\/]").unwrap(),
                regex::Regex::new(r"\.\.%2[fF]").unwrap(),
                regex::Regex::new(r"\.\.%5[cC]").unwrap(),
            ],
            command_injection: vec![
                regex::Regex::new(r"[;&|`$]").unwrap(),
                regex::Regex::new(r"\$\{.*\}").unwrap(),
                regex::Regex::new(r"\$\(.*\)").unwrap(),
            ],
            ldap_injection: vec![
                regex::Regex::new(r"[()&|*]").unwrap(),
                regex::Regex::new(r"(?i)(objectclass=\*)").unwrap(),
            ],
        }
    }
}
```

## File Upload Validation

```rust
/// File upload validator
pub struct FileUploadValidator {
    allowed_types: HashSet<String>,
    max_size: usize,
    scan_for_malware: bool,
}

impl FileUploadValidator {
    pub fn new(config: FileUploadConfig) -> Self {
        Self {
            allowed_types: config.allowed_types.into_iter().collect(),
            max_size: config.max_size,
            scan_for_malware: config.scan_for_malware,
        }
    }
    
    /// Validate uploaded file
    pub async fn validate_file(
        &self,
        file_name: &str,
        file_data: &[u8],
    ) -> Result<ValidationResult> {
        let mut result = ValidationResult::new();
        
        // Check file size
        if file_data.len() > self.max_size {
            result.add_error("file_size", "exceeds_limit");
        }
        
        // Check file type
        let mime_type = self.detect_mime_type(file_data)?;
        if !self.allowed_types.contains(&mime_type) {
            result.add_error("file_type", "not_allowed");
        }
        
        // Check file extension
        if let Some(ext) = Path::new(file_name).extension() {
            let ext_str = ext.to_string_lossy().to_lowercase();
            if !self.is_extension_allowed(&ext_str) {
                result.add_error("file_extension", "not_allowed");
            }
        }
        
        // Check for malware
        if self.scan_for_malware {
            if self.contains_malware_signatures(file_data).await? {
                result.add_error("file_content", "potential_malware");
            }
        }
        
        // Check for embedded scripts
        if self.contains_embedded_scripts(file_data)? {
            result.add_error("file_content", "embedded_scripts");
        }
        
        Ok(result)
    }
    
    /// Detect MIME type
    fn detect_mime_type(&self, data: &[u8]) -> Result<String> {
        let mime = tree_magic_mini::from_u8(data);
        Ok(mime.to_string())
    }
    
    /// Check if extension is allowed
    fn is_extension_allowed(&self, ext: &str) -> bool {
        let dangerous_extensions = [
            "exe", "dll", "bat", "cmd", "com", "scr",
            "vbs", "js", "jar", "app", "dmg", "pkg",
        ];
        
        !dangerous_extensions.contains(&ext)
    }
    
    /// Check for malware signatures
    async fn contains_malware_signatures(&self, data: &[u8]) -> Result<bool> {
        // Basic signature check (in production, use proper AV)
        let signatures = [
            b"MZ", // PE header
            b"\x7fELF", // ELF header
            b"#!/bin/sh", // Shell script
            b"#!/bin/bash", // Bash script
        ];
        
        for sig in &signatures {
            if data.starts_with(sig) {
                return Ok(true);
            }
        }
        
        Ok(false)
    }
    
    /// Check for embedded scripts
    fn contains_embedded_scripts(&self, data: &[u8]) -> Result<bool> {
        let content = String::from_utf8_lossy(data);
        
        let script_patterns = [
            "<script",
            "javascript:",
            "vbscript:",
            "onload=",
            "onerror=",
        ];
        
        for pattern in &script_patterns {
            if content.to_lowercase().contains(pattern) {
                return Ok(true);
            }
        }
        
        Ok(false)
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ValidationConfig {
    pub enable_security_checks: bool,
    pub max_depth: usize,
    pub max_array_length: usize,
    pub max_string_length: usize,
    pub file_upload: FileUploadConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FileUploadConfig {
    pub allowed_types: Vec<String>,
    pub max_size: usize,
    pub scan_for_malware: bool,
}

impl Default for ValidationConfig {
    fn default() -> Self {
        Self {
            enable_security_checks: true,
            max_depth: 10,
            max_array_length: 1000,
            max_string_length: 1_000_000,
            file_upload: FileUploadConfig {
                allowed_types: vec![
                    "image/jpeg".to_string(),
                    "image/png".to_string(),
                    "image/gif".to_string(),
                    "image/webp".to_string(),
                    "application/pdf".to_string(),
                ],
                max_size: 10 * 1024 * 1024, // 10MB
                scan_for_malware: true,
            },
        }
    }
}
```

## Summary

Diese Input Validation Implementation bietet:
- **Comprehensive Validation** - Type, Format, Security Checks
- **Custom Validators** - Extensible Validation Framework
- **Sanitization** - HTML, SQL, Script Sanitization
- **Security Focus** - Injection Prevention, Malware Detection
- **File Upload Validation** - MIME Type, Size, Content Checks
- **Schema Validation** - Content Type Schema Support

Robust Input Validation für sichere Application.