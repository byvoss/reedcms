# Registry Field Validation

## Overview

Field Validation System für das Registry. Validiert Snippet-Felder basierend auf definierten Regeln.

## Field Validators

```rust
use regex::Regex;
use std::collections::HashMap;
use std::sync::Arc;

/// Central field validation system
pub struct FieldValidators {
    validators: HashMap<String, Box<dyn FieldValidator>>,
    patterns: Arc<ValidationPatterns>,
}

impl FieldValidators {
    pub fn new() -> Self {
        let mut validators = HashMap::new();
        let patterns = Arc::new(ValidationPatterns::new());
        
        // Register built-in validators
        validators.insert("text".to_string(), 
            Box::new(TextValidator::new(patterns.clone())) as Box<dyn FieldValidator>);
        validators.insert("email".to_string(), 
            Box::new(EmailValidator::new(patterns.clone())) as Box<dyn FieldValidator>);
        validators.insert("url".to_string(), 
            Box::new(UrlValidator::new(patterns.clone())) as Box<dyn FieldValidator>);
        validators.insert("number".to_string(), 
            Box::new(NumberValidator::new()) as Box<dyn FieldValidator>);
        validators.insert("date".to_string(), 
            Box::new(DateValidator::new()) as Box<dyn FieldValidator>);
        
        Self { validators, patterns }
    }
    
    /// Validate field value
    pub fn validate_field(
        &self,
        field_type: &FieldType,
        value: &serde_json::Value,
        rules: &[ValidationRule],
    ) -> ValidationResult {
        // Get base validator for field type
        let validator = self.get_validator(field_type);
        
        // Run base validation
        let mut result = validator.validate(value);
        
        // Apply additional rules
        for rule in rules {
            if !result.is_valid {
                break; // Stop on first error
            }
            
            result = self.apply_rule(value, rule);
        }
        
        result
    }
    
    fn get_validator(&self, field_type: &FieldType) -> &dyn FieldValidator {
        match field_type {
            FieldType::Text | FieldType::Textarea => &*self.validators["text"],
            FieldType::Email => &*self.validators["email"],
            FieldType::Url => &*self.validators["url"],
            FieldType::Number => &*self.validators["number"],
            FieldType::Date => &*self.validators["date"],
            _ => &*self.validators["text"], // Default to text validator
        }
    }
}

/// Validation result
#[derive(Debug, Clone)]
pub struct ValidationResult {
    pub is_valid: bool,
    pub errors: Vec<ValidationError>,
    pub warnings: Vec<String>,
}

impl ValidationResult {
    pub fn valid() -> Self {
        Self {
            is_valid: true,
            errors: vec![],
            warnings: vec![],
        }
    }
    
    pub fn invalid(error: ValidationError) -> Self {
        Self {
            is_valid: false,
            errors: vec![error],
            warnings: vec![],
        }
    }
    
    pub fn merge(mut self, other: ValidationResult) -> Self {
        self.is_valid = self.is_valid && other.is_valid;
        self.errors.extend(other.errors);
        self.warnings.extend(other.warnings);
        self
    }
}

#[derive(Debug, Clone)]
pub struct ValidationError {
    pub field: String,
    pub message: String,
    pub code: String,
}
```

## Base Validator Trait

```rust
/// Trait for field validators
pub trait FieldValidator: Send + Sync {
    fn validate(&self, value: &serde_json::Value) -> ValidationResult;
    fn name(&self) -> &'static str;
}

/// Common validation patterns
pub struct ValidationPatterns {
    email: Regex,
    url: Regex,
    slug: Regex,
    phone: Regex,
    alphanumeric: Regex,
}

impl ValidationPatterns {
    pub fn new() -> Self {
        Self {
            email: Regex::new(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$").unwrap(),
            url: Regex::new(r"^https?://[^\s/$.?#].[^\s]*$").unwrap(),
            slug: Regex::new(r"^[a-z0-9]+(?:-[a-z0-9]+)*$").unwrap(),
            phone: Regex::new(r"^\+?[0-9\s\-\(\)]+$").unwrap(),
            alphanumeric: Regex::new(r"^[a-zA-Z0-9]+$").unwrap(),
        }
    }
}
```

## Built-in Validators

```rust
/// Text field validator
pub struct TextValidator {
    patterns: Arc<ValidationPatterns>,
}

impl TextValidator {
    pub fn new(patterns: Arc<ValidationPatterns>) -> Self {
        Self { patterns }
    }
}

impl FieldValidator for TextValidator {
    fn validate(&self, value: &serde_json::Value) -> ValidationResult {
        match value {
            serde_json::Value::String(s) => ValidationResult::valid(),
            serde_json::Value::Null => ValidationResult::valid(),
            _ => ValidationResult::invalid(ValidationError {
                field: String::new(),
                message: "Value must be a string".to_string(),
                code: "invalid_type".to_string(),
            }),
        }
    }
    
    fn name(&self) -> &'static str {
        "text"
    }
}

/// Email validator
pub struct EmailValidator {
    patterns: Arc<ValidationPatterns>,
}

impl EmailValidator {
    pub fn new(patterns: Arc<ValidationPatterns>) -> Self {
        Self { patterns }
    }
}

impl FieldValidator for EmailValidator {
    fn validate(&self, value: &serde_json::Value) -> ValidationResult {
        match value {
            serde_json::Value::String(s) => {
                if s.is_empty() {
                    return ValidationResult::valid();
                }
                
                if self.patterns.email.is_match(s) {
                    ValidationResult::valid()
                } else {
                    ValidationResult::invalid(ValidationError {
                        field: String::new(),
                        message: "Invalid email format".to_string(),
                        code: "invalid_email".to_string(),
                    })
                }
            }
            serde_json::Value::Null => ValidationResult::valid(),
            _ => ValidationResult::invalid(ValidationError {
                field: String::new(),
                message: "Email must be a string".to_string(),
                code: "invalid_type".to_string(),
            }),
        }
    }
    
    fn name(&self) -> &'static str {
        "email"
    }
}

/// URL validator
pub struct UrlValidator {
    patterns: Arc<ValidationPatterns>,
}

impl UrlValidator {
    pub fn new(patterns: Arc<ValidationPatterns>) -> Self {
        Self { patterns }
    }
}

impl FieldValidator for UrlValidator {
    fn validate(&self, value: &serde_json::Value) -> ValidationResult {
        match value {
            serde_json::Value::String(s) => {
                if s.is_empty() {
                    return ValidationResult::valid();
                }
                
                if self.patterns.url.is_match(s) {
                    ValidationResult::valid()
                } else {
                    ValidationResult::invalid(ValidationError {
                        field: String::new(),
                        message: "Invalid URL format".to_string(),
                        code: "invalid_url".to_string(),
                    })
                }
            }
            serde_json::Value::Null => ValidationResult::valid(),
            _ => ValidationResult::invalid(ValidationError {
                field: String::new(),
                message: "URL must be a string".to_string(),
                code: "invalid_type".to_string(),
            }),
        }
    }
    
    fn name(&self) -> &'static str {
        "url"
    }
}

/// Number validator
pub struct NumberValidator;

impl NumberValidator {
    pub fn new() -> Self {
        Self
    }
}

impl FieldValidator for NumberValidator {
    fn validate(&self, value: &serde_json::Value) -> ValidationResult {
        match value {
            serde_json::Value::Number(_) => ValidationResult::valid(),
            serde_json::Value::Null => ValidationResult::valid(),
            serde_json::Value::String(s) => {
                // Try to parse string as number
                if s.parse::<f64>().is_ok() {
                    ValidationResult::valid()
                } else {
                    ValidationResult::invalid(ValidationError {
                        field: String::new(),
                        message: "Invalid number format".to_string(),
                        code: "invalid_number".to_string(),
                    })
                }
            }
            _ => ValidationResult::invalid(ValidationError {
                field: String::new(),
                message: "Value must be a number".to_string(),
                code: "invalid_type".to_string(),
            }),
        }
    }
    
    fn name(&self) -> &'static str {
        "number"
    }
}

/// Date validator
pub struct DateValidator;

impl DateValidator {
    pub fn new() -> Self {
        Self
    }
}

impl FieldValidator for DateValidator {
    fn validate(&self, value: &serde_json::Value) -> ValidationResult {
        match value {
            serde_json::Value::String(s) => {
                if s.is_empty() {
                    return ValidationResult::valid();
                }
                
                // Try to parse as ISO date
                if chrono::DateTime::parse_from_rfc3339(s).is_ok() {
                    ValidationResult::valid()
                } else if chrono::NaiveDate::parse_from_str(s, "%Y-%m-%d").is_ok() {
                    ValidationResult::valid()
                } else {
                    ValidationResult::invalid(ValidationError {
                        field: String::new(),
                        message: "Invalid date format. Use YYYY-MM-DD or ISO 8601".to_string(),
                        code: "invalid_date".to_string(),
                    })
                }
            }
            serde_json::Value::Null => ValidationResult::valid(),
            _ => ValidationResult::invalid(ValidationError {
                field: String::new(),
                message: "Date must be a string".to_string(),
                code: "invalid_type".to_string(),
            }),
        }
    }
    
    fn name(&self) -> &'static str {
        "date"
    }
}
```

## Validation Rules Application

```rust
impl FieldValidators {
    /// Apply validation rule to value
    fn apply_rule(
        &self,
        value: &serde_json::Value,
        rule: &ValidationRule,
    ) -> ValidationResult {
        match &rule.rule_type {
            ValidationType::Required => self.validate_required(value, rule),
            ValidationType::MinLength => self.validate_min_length(value, rule),
            ValidationType::MaxLength => self.validate_max_length(value, rule),
            ValidationType::Pattern => self.validate_pattern(value, rule),
            ValidationType::Range => self.validate_range(value, rule),
            ValidationType::Custom(name) => self.validate_custom(value, rule, name),
        }
    }
    
    fn validate_required(
        &self,
        value: &serde_json::Value,
        rule: &ValidationRule,
    ) -> ValidationResult {
        let is_empty = match value {
            serde_json::Value::Null => true,
            serde_json::Value::String(s) => s.is_empty(),
            serde_json::Value::Array(a) => a.is_empty(),
            serde_json::Value::Object(o) => o.is_empty(),
            _ => false,
        };
        
        if is_empty {
            ValidationResult::invalid(ValidationError {
                field: rule.field.clone(),
                message: rule.error_message.clone(),
                code: "required".to_string(),
            })
        } else {
            ValidationResult::valid()
        }
    }
    
    fn validate_min_length(
        &self,
        value: &serde_json::Value,
        rule: &ValidationRule,
    ) -> ValidationResult {
        let min = rule.parameters.get("min")
            .and_then(|v| v.as_u64())
            .unwrap_or(0) as usize;
        
        let length = match value {
            serde_json::Value::String(s) => s.len(),
            serde_json::Value::Array(a) => a.len(),
            _ => return ValidationResult::valid(),
        };
        
        if length < min {
            ValidationResult::invalid(ValidationError {
                field: rule.field.clone(),
                message: rule.error_message.clone(),
                code: "min_length".to_string(),
            })
        } else {
            ValidationResult::valid()
        }
    }
    
    fn validate_max_length(
        &self,
        value: &serde_json::Value,
        rule: &ValidationRule,
    ) -> ValidationResult {
        let max = rule.parameters.get("max")
            .and_then(|v| v.as_u64())
            .unwrap_or(u64::MAX) as usize;
        
        let length = match value {
            serde_json::Value::String(s) => s.len(),
            serde_json::Value::Array(a) => a.len(),
            _ => return ValidationResult::valid(),
        };
        
        if length > max {
            ValidationResult::invalid(ValidationError {
                field: rule.field.clone(),
                message: rule.error_message.clone(),
                code: "max_length".to_string(),
            })
        } else {
            ValidationResult::valid()
        }
    }
    
    fn validate_pattern(
        &self,
        value: &serde_json::Value,
        rule: &ValidationRule,
    ) -> ValidationResult {
        let pattern_str = rule.parameters.get("pattern")
            .and_then(|v| v.as_str())
            .unwrap_or("");
        
        if let Ok(pattern) = Regex::new(pattern_str) {
            if let serde_json::Value::String(s) = value {
                if pattern.is_match(s) {
                    ValidationResult::valid()
                } else {
                    ValidationResult::invalid(ValidationError {
                        field: rule.field.clone(),
                        message: rule.error_message.clone(),
                        code: "pattern".to_string(),
                    })
                }
            } else {
                ValidationResult::valid()
            }
        } else {
            ValidationResult::invalid(ValidationError {
                field: rule.field.clone(),
                message: format!("Invalid pattern: {}", pattern_str),
                code: "invalid_pattern".to_string(),
            })
        }
    }
    
    fn validate_range(
        &self,
        value: &serde_json::Value,
        rule: &ValidationRule,
    ) -> ValidationResult {
        let min = rule.parameters.get("min")
            .and_then(|v| v.as_f64());
        let max = rule.parameters.get("max")
            .and_then(|v| v.as_f64());
        
        if let Some(num) = value.as_f64() {
            if let Some(min_val) = min {
                if num < min_val {
                    return ValidationResult::invalid(ValidationError {
                        field: rule.field.clone(),
                        message: rule.error_message.clone(),
                        code: "below_minimum".to_string(),
                    });
                }
            }
            
            if let Some(max_val) = max {
                if num > max_val {
                    return ValidationResult::invalid(ValidationError {
                        field: rule.field.clone(),
                        message: rule.error_message.clone(),
                        code: "above_maximum".to_string(),
                    });
                }
            }
        }
        
        ValidationResult::valid()
    }
    
    fn validate_custom(
        &self,
        value: &serde_json::Value,
        rule: &ValidationRule,
        name: &str,
    ) -> ValidationResult {
        // Custom validators would be registered separately
        tracing::warn!("Custom validator '{}' not implemented", name);
        ValidationResult::valid()
    }
}
```

## Snippet Validation

```rust
/// Validates complete snippet data
pub struct SnippetValidator {
    registry: Arc<RegistryManager>,
    validators: Arc<FieldValidators>,
}

impl SnippetValidator {
    pub fn new(
        registry: Arc<RegistryManager>,
        validators: Arc<FieldValidators>,
    ) -> Self {
        Self { registry, validators }
    }
    
    /// Validate snippet data
    pub async fn validate_snippet(
        &self,
        snippet_type: &str,
        data: &HashMap<String, serde_json::Value>,
    ) -> Result<ValidationResult> {
        let definition = self.registry.get_definition(snippet_type).await?;
        let mut result = ValidationResult::valid();
        
        // Validate each field
        for field_def in &definition.fields {
            let value = data.get(&field_def.name)
                .unwrap_or(&serde_json::Value::Null);
            
            // Get validation rules for this field
            let field_rules: Vec<&ValidationRule> = definition.validation_rules.iter()
                .filter(|rule| rule.field == field_def.name)
                .collect();
            
            // Check required fields
            if field_def.required && value.is_null() {
                result = result.merge(ValidationResult::invalid(ValidationError {
                    field: field_def.name.clone(),
                    message: format!("{} is required", field_def.display_name),
                    code: "required".to_string(),
                }));
                continue;
            }
            
            // Validate field type and rules
            let field_result = self.validators.validate_field(
                &field_def.field_type,
                value,
                &field_rules,
            );
            
            result = result.merge(field_result);
        }
        
        Ok(result)
    }
}
```

## Summary

Dieses Validation System bietet:
- **Type-safe Validation** - Strikte Typ-Prüfung für alle Felder
- **Extensible Rules** - Einfach erweiterbare Validierungsregeln
- **Pattern Matching** - Regex-basierte Validierung
- **Custom Validators** - Plugin-fähige Validierung
- **Detailed Errors** - Klare Fehlermeldungen für Nutzer

Validierung erfolgt auf Registry-Ebene für Konsistenz.