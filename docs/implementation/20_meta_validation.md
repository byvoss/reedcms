# Meta-Validation System

## Overview

Meta-Validation erweitert das Registry System um komplexe Validierungsregeln zwischen Snippets. Cross-Snippet Constraints und Business Rules.

## Meta-Validation Manager

```rust
use std::collections::HashMap;
use std::sync::Arc;

/// Manages cross-snippet validation rules
pub struct MetaValidationManager {
    registry: Arc<RegistryManager>,
    meta_manager: Arc<MetaSnippetManager>,
    validators: Arc<FieldValidators>,
    rules: Arc<RwLock<Vec<MetaValidationRule>>>,
}

impl MetaValidationManager {
    pub fn new(
        registry: Arc<RegistryManager>,
        meta_manager: Arc<MetaSnippetManager>,
        validators: Arc<FieldValidators>,
    ) -> Self {
        Self {
            registry,
            meta_manager,
            validators,
            rules: Arc::new(RwLock::new(Vec::new())),
        }
    }
    
    /// Initialize meta-validation rules
    pub async fn initialize(&self) -> Result<()> {
        let rules = self.load_rules_from_csv().await?;
        
        let mut rule_store = self.rules.write().await;
        rule_store.clear();
        rule_store.extend(rules);
        
        tracing::info!("Loaded {} meta-validation rules", rule_store.len());
        Ok(())
    }
    
    /// Validate entity against meta rules
    pub async fn validate_entity(
        &self,
        entity: &Entity,
        context: &ValidationContext,
    ) -> Result<MetaValidationResult> {
        let mut result = MetaValidationResult::valid();
        
        // Get applicable rules
        let rules = self.get_applicable_rules(entity, context).await?;
        
        for rule in rules {
            let rule_result = self.execute_rule(&rule, entity, context).await?;
            result = result.merge(rule_result);
        }
        
        Ok(result)
    }
}

/// Validation context for meta rules
#[derive(Debug, Clone)]
pub struct ValidationContext {
    pub parent_entity: Option<Entity>,
    pub siblings: Vec<Entity>,
    pub depth: u32,
    pub user_context: UserContext,
}

/// Meta validation result
#[derive(Debug, Clone)]
pub struct MetaValidationResult {
    pub is_valid: bool,
    pub errors: Vec<MetaValidationError>,
    pub warnings: Vec<String>,
    pub hints: Vec<String>, // Intelligence hints
}

impl MetaValidationResult {
    pub fn valid() -> Self {
        Self {
            is_valid: true,
            errors: vec![],
            warnings: vec![],
            hints: vec![],
        }
    }
    
    pub fn merge(mut self, other: MetaValidationResult) -> Self {
        self.is_valid = self.is_valid && other.is_valid;
        self.errors.extend(other.errors);
        self.warnings.extend(other.warnings);
        self.hints.extend(other.hints);
        self
    }
}
```

## Meta-Validation Rules

```rust
/// Cross-snippet validation rule
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MetaValidationRule {
    pub id: String,
    pub name: String,
    pub rule_type: MetaRuleType,
    pub scope: RuleScope,
    pub conditions: Vec<RuleCondition>,
    pub constraints: Vec<RuleConstraint>,
    pub error_template: String,
    pub severity: RuleSeverity,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum MetaRuleType {
    Uniqueness,         // Field must be unique across scope
    Dependency,         // Field depends on other fields/entities
    Cardinality,        // Limits on child counts
    Consistency,        // Cross-entity consistency checks
    BusinessRule,       // Custom business logic
    Hierarchy,          // Parent-child constraints
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum RuleScope {
    Global,             // All entities
    SnippetType(String), // Specific snippet type
    Parent,             // Within parent entity
    Site,               // Within site/theme
    Custom(String),     // Custom scope
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RuleCondition {
    pub field: String,
    pub operator: ConditionOperator,
    pub value: serde_json::Value,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ConditionOperator {
    Equals,
    NotEquals,
    Contains,
    StartsWith,
    EndsWith,
    GreaterThan,
    LessThan,
    Matches(String), // Regex
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RuleConstraint {
    pub constraint_type: ConstraintType,
    pub parameters: HashMap<String, serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ConstraintType {
    UniqueField(String),
    MaxCount(u32),
    MinCount(u32),
    RequiredIf(String, serde_json::Value),
    ForbiddenIf(String, serde_json::Value),
    CrossEntitySum(String, f64), // Sum across entities
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum RuleSeverity {
    Error,      // Blocks save
    Warning,    // Shows warning
    Info,       // Informational only
}
```

## Rule Execution Engine

```rust
impl MetaValidationManager {
    /// Get rules applicable to entity
    async fn get_applicable_rules(
        &self,
        entity: &Entity,
        context: &ValidationContext,
    ) -> Result<Vec<MetaValidationRule>> {
        let rules = self.rules.read().await;
        
        let applicable: Vec<_> = rules.iter()
            .filter(|rule| self.is_rule_applicable(rule, entity, context))
            .cloned()
            .collect();
        
        Ok(applicable)
    }
    
    /// Check if rule applies to entity
    fn is_rule_applicable(
        &self,
        rule: &MetaValidationRule,
        entity: &Entity,
        context: &ValidationContext,
    ) -> bool {
        match &rule.scope {
            RuleScope::Global => true,
            RuleScope::SnippetType(snippet_type) => {
                entity.entity_type == EntityType::Snippet &&
                entity.data.get("snippet_type")
                    .and_then(|v| v.as_str())
                    .map(|t| t == snippet_type)
                    .unwrap_or(false)
            }
            RuleScope::Parent => context.parent_entity.is_some(),
            RuleScope::Site => true, // Check site context
            RuleScope::Custom(scope) => {
                // Custom scope logic
                false
            }
        }
    }
    
    /// Execute single rule
    async fn execute_rule(
        &self,
        rule: &MetaValidationRule,
        entity: &Entity,
        context: &ValidationContext,
    ) -> Result<MetaValidationResult> {
        // Check conditions first
        if !self.check_conditions(&rule.conditions, entity, context).await? {
            return Ok(MetaValidationResult::valid());
        }
        
        // Execute constraints
        let mut result = MetaValidationResult::valid();
        
        for constraint in &rule.constraints {
            let constraint_result = self.execute_constraint(
                constraint,
                entity,
                context,
                rule,
            ).await?;
            
            result = result.merge(constraint_result);
        }
        
        Ok(result)
    }
    
    /// Check rule conditions
    async fn check_conditions(
        &self,
        conditions: &[RuleCondition],
        entity: &Entity,
        context: &ValidationContext,
    ) -> Result<bool> {
        for condition in conditions {
            let field_value = entity.data.get(&condition.field);
            
            let matches = match &condition.operator {
                ConditionOperator::Equals => {
                    field_value == Some(&condition.value)
                }
                ConditionOperator::NotEquals => {
                    field_value != Some(&condition.value)
                }
                ConditionOperator::Contains => {
                    field_value
                        .and_then(|v| v.as_str())
                        .and_then(|s| condition.value.as_str())
                        .map(|needle| s.contains(needle))
                        .unwrap_or(false)
                }
                ConditionOperator::Matches(pattern) => {
                    if let Ok(regex) = regex::Regex::new(pattern) {
                        field_value
                            .and_then(|v| v.as_str())
                            .map(|s| regex.is_match(s))
                            .unwrap_or(false)
                    } else {
                        false
                    }
                }
                // Implement other operators...
                _ => true,
            };
            
            if !matches {
                return Ok(false);
            }
        }
        
        Ok(true)
    }
}
```

## Constraint Executors

```rust
impl MetaValidationManager {
    /// Execute constraint check
    async fn execute_constraint(
        &self,
        constraint: &RuleConstraint,
        entity: &Entity,
        context: &ValidationContext,
        rule: &MetaValidationRule,
    ) -> Result<MetaValidationResult> {
        match &constraint.constraint_type {
            ConstraintType::UniqueField(field_name) => {
                self.check_unique_field(field_name, entity, context, rule).await
            }
            ConstraintType::MaxCount(max) => {
                self.check_max_count(*max, entity, context, rule).await
            }
            ConstraintType::RequiredIf(field, value) => {
                self.check_required_if(field, value, entity, rule).await
            }
            ConstraintType::CrossEntitySum(field, max_sum) => {
                self.check_cross_entity_sum(field, *max_sum, context, rule).await
            }
            _ => Ok(MetaValidationResult::valid()),
        }
    }
    
    /// Check field uniqueness
    async fn check_unique_field(
        &self,
        field_name: &str,
        entity: &Entity,
        context: &ValidationContext,
        rule: &MetaValidationRule,
    ) -> Result<MetaValidationResult> {
        let field_value = entity.data.get(field_name);
        
        if field_value.is_none() {
            return Ok(MetaValidationResult::valid());
        }
        
        // Check siblings for duplicate
        for sibling in &context.siblings {
            if sibling.id != entity.id {
                if sibling.data.get(field_name) == field_value {
                    return Ok(MetaValidationResult {
                        is_valid: false,
                        errors: vec![MetaValidationError {
                            rule_id: rule.id.clone(),
                            field: field_name.to_string(),
                            message: self.format_error_message(rule, entity, context),
                            severity: rule.severity,
                        }],
                        warnings: vec![],
                        hints: vec![format!(
                            "Field '{}' must be unique within scope",
                            field_name
                        )],
                    });
                }
            }
        }
        
        Ok(MetaValidationResult::valid())
    }
    
    /// Check maximum count constraint
    async fn check_max_count(
        &self,
        max: u32,
        entity: &Entity,
        context: &ValidationContext,
        rule: &MetaValidationRule,
    ) -> Result<MetaValidationResult> {
        let count = context.siblings.iter()
            .filter(|s| {
                s.entity_type == entity.entity_type &&
                s.data.get("snippet_type") == entity.data.get("snippet_type")
            })
            .count() as u32;
        
        if count > max {
            let mut result = MetaValidationResult::valid();
            result.is_valid = false;
            result.errors.push(MetaValidationError {
                rule_id: rule.id.clone(),
                field: String::new(),
                message: format!("Maximum {} instances allowed", max),
                severity: rule.severity,
            });
            
            // Add intelligence hint
            result.hints.push(format!(
                "Consider using a different snippet type or removing existing instances"
            ));
            
            return Ok(result);
        }
        
        Ok(MetaValidationResult::valid())
    }
}

#[derive(Debug, Clone)]
pub struct MetaValidationError {
    pub rule_id: String,
    pub field: String,
    pub message: String,
    pub severity: RuleSeverity,
}
```

## CSV Rule Loading

```rust
impl MetaValidationManager {
    /// Load validation rules from CSV
    async fn load_rules_from_csv(&self) -> Result<Vec<MetaValidationRule>> {
        let csv_path = self.registry.csv_path.join("meta_validation_rules.csv");
        
        if !csv_path.exists() {
            return Ok(self.default_rules());
        }
        
        let mut reader = csv::Reader::from_path(&csv_path)?;
        let mut rules = Vec::new();
        
        for result in reader.records() {
            let record = result?;
            
            let id = record.get(0).unwrap_or("").to_string();
            let name = record.get(1).unwrap_or("").to_string();
            let rule_type = self.parse_rule_type(record.get(2).unwrap_or(""));
            let scope = self.parse_scope(record.get(3).unwrap_or(""));
            let conditions_json = record.get(4).unwrap_or("[]");
            let constraints_json = record.get(5).unwrap_or("[]");
            let error_template = record.get(6).unwrap_or("Validation failed").to_string();
            let severity = self.parse_severity(record.get(7).unwrap_or("error"));
            
            let conditions: Vec<RuleCondition> = serde_json::from_str(conditions_json)?;
            let constraints: Vec<RuleConstraint> = serde_json::from_str(constraints_json)?;
            
            rules.push(MetaValidationRule {
                id,
                name,
                rule_type,
                scope,
                conditions,
                constraints,
                error_template,
                severity,
            });
        }
        
        Ok(rules)
    }
    
    /// Default built-in rules
    fn default_rules(&self) -> Vec<MetaValidationRule> {
        vec![
            // Unique slug within parent
            MetaValidationRule {
                id: "unique-slug".to_string(),
                name: "Unique Slug Rule".to_string(),
                rule_type: MetaRuleType::Uniqueness,
                scope: RuleScope::Parent,
                conditions: vec![],
                constraints: vec![
                    RuleConstraint {
                        constraint_type: ConstraintType::UniqueField("slug".to_string()),
                        parameters: HashMap::new(),
                    }
                ],
                error_template: "Slug must be unique within parent".to_string(),
                severity: RuleSeverity::Error,
            },
            // Max hero banners per page
            MetaValidationRule {
                id: "max-hero-banners".to_string(),
                name: "Maximum Hero Banners".to_string(),
                rule_type: MetaRuleType::Cardinality,
                scope: RuleScope::Parent,
                conditions: vec![
                    RuleCondition {
                        field: "snippet_type".to_string(),
                        operator: ConditionOperator::Equals,
                        value: serde_json::Value::String("hero-banner".to_string()),
                    }
                ],
                constraints: vec![
                    RuleConstraint {
                        constraint_type: ConstraintType::MaxCount(1),
                        parameters: HashMap::new(),
                    }
                ],
                error_template: "Only one hero banner allowed per page".to_string(),
                severity: RuleSeverity::Error,
            },
        ]
    }
}
```

## Intelligence Integration

```rust
impl MetaValidationManager {
    /// Add intelligence hints to validation result
    async fn add_intelligence_hints(
        &self,
        result: &mut MetaValidationResult,
        entity: &Entity,
        context: &ValidationContext,
    ) -> Result<()> {
        // Check for common patterns
        if !result.is_valid {
            // Duplicate content hint
            if result.errors.iter().any(|e| e.message.contains("unique")) {
                result.hints.push(
                    "Consider using a reference to existing content instead of duplicating"
                        .to_string()
                );
            }
            
            // Structure hint
            if result.errors.iter().any(|e| e.message.contains("maximum")) {
                result.hints.push(
                    "Review content structure - perhaps this belongs in a different section"
                        .to_string()
                );
            }
        }
        
        Ok(())
    }
    
    /// Analyze validation patterns for optimization
    pub async fn analyze_validation_patterns(&self) -> ValidationAnalysis {
        let rules = self.rules.read().await;
        
        ValidationAnalysis {
            total_rules: rules.len(),
            rules_by_type: self.count_by_type(&rules),
            most_violated: self.find_most_violated(&rules),
            performance_impact: self.estimate_performance_impact(&rules),
        }
    }
}

#[derive(Debug, Serialize)]
pub struct ValidationAnalysis {
    pub total_rules: usize,
    pub rules_by_type: HashMap<String, usize>,
    pub most_violated: Vec<String>,
    pub performance_impact: PerformanceImpact,
}

#[derive(Debug, Serialize)]
pub struct PerformanceImpact {
    pub estimated_ms: f64,
    pub heavy_rules: Vec<String>,
}
```

## Summary

Dieses Meta-Validation System bietet:
- **Cross-Snippet Rules** - Validierung über Entity-Grenzen
- **Business Logic** - Komplexe Geschäftsregeln
- **Scope-based Validation** - Flexible Gültigkeitsbereiche
- **Intelligence Hints** - Hilfreiche Hinweise bei Fehlern
- **Performance Analysis** - Optimierung der Regel-Ausführung

Meta-Validation macht ReedCMS intelligent ohne AI.