# ReedCMS Content Firewall System

## Philosophy

**"PF for Content" - OpenBSD PF-inspired content protection at the field level**

The ReedCMS Content Firewall provides **modular, rule-based content validation and sanitization** using familiar PF-style syntax and CLI management. Every snippet field and template variable gets automatic protection through configurable rules.

## Architecture Overview

### **Embedded Process Design**
```rust
// Integrated firewall - no separate process needed
pub struct ReedContentFirewall {
    rule_engine: RuleEngine,              // Rule file management
    field_rules: HashMap<String, Vec<ActiveRule>>,  // field_type -> rules
    plugin_rules: PluginRuleRegistry,     // Plugin-provided rules
    cli_interface: FirewallCLI,           // CLI management
}
```

**Integration Points:**
- **Snippet Processing:** All field content filtered before save
- **Template Rendering:** All template variables automatically secured  
- **Form Handling:** HTTP form data validated on input
- **Plugin Content:** Third-party content filtered through same system

## Rule File System

### **Directory Structure**
```
config/firewall/rules/
├── text_field.toml          # Text field rules (built-in)
├── email_field.toml         # Email validation rules (built-in)
├── url_field.toml           # URL validation rules (built-in)
├── number_field.toml        # Numeric validation rules (built-in)
├── content_field.toml       # Rich content rules (built-in)
└── plugins/
    ├── profanity_filter.toml    # Plugin rule
    ├── ai_content_check.toml    # AI plugin rule
    └── custom_validator.toml    # Custom rule
```

### **Built-in Rule File Example**
```toml
# config/firewall/rules/text_field.toml
[rule_info]
name = "text_field"
version = "1.0.0"
description = "Standard text field validation and sanitization"
maintainer = "ReedCMS Core Team"

[rules.strip_html]
enabled = true
description = "Remove HTML tags from content"
implementation = "rust_builtin"
parameters = { 
    preserve_whitespace = true, 
    allowed_tags = [] 
}

[rules.max_length]
enabled = true  
description = "Enforce maximum text length"
implementation = "rust_builtin"
parameters = { 
    default_max = 10000, 
    hard_limit = 50000 
}

[rules.no_script_tags]
enabled = true
description = "Block JavaScript and script content"
implementation = "rust_builtin"
parameters = { 
    blocked_tags = ["script", "iframe", "object", "embed"],
    blocked_protocols = ["javascript:", "vbscript:", "data:"]
}

[rules.normalize_whitespace]
enabled = false  # Disabled by default
description = "Normalize whitespace and line breaks"
implementation = "rust_builtin"
parameters = { 
    preserve_paragraphs = true, 
    max_consecutive_spaces = 2 
}

[rules.profanity_filter]
enabled = false  # Optional rule
description = "Filter profanity and inappropriate content"
implementation = "plugin"
plugin_name = "profanity_filter"
parameters = { 
    severity = "medium", 
    replacement = "***" 
}
```

### **Plugin Rule Example**
```toml
# config/firewall/rules/plugins/ai_content_check.toml
[rule_info]
name = "ai_content_check"
version = "2.1.0"
description = "AI-powered content validation and filtering"
maintainer = "AI-Core Plugin Team"
plugin_dependency = "reed-plugin-ai-core"

[rules.spam_detection]
enabled = false
description = "Detect spam content using AI"
implementation = "plugin_ai_core"
parameters = { 
    confidence_threshold = 0.8,
    model = "spam-detection-v2",
    max_processing_time_ms = 500
}

[rules.sentiment_analysis]
enabled = false
description = "Analyze content sentiment"
implementation = "plugin_ai_core"
parameters = {
    block_negative_threshold = -0.9,
    alert_negative_threshold = -0.5,
    action = "alert"  # "block" or "alert"
}

[rules.content_quality_check]
enabled = false
description = "Check content quality and readability"
implementation = "plugin_ai_core" 
parameters = {
    min_quality_score = 0.6,
    check_grammar = true,
    check_readability = true
}
```

## Firewall Configuration

### **Main Firewall Config**
```toml
# config/firewall.toml - PF-inspired field-level protection
[default_rules]
# Default protection for all field types
text_field = "strip_html,max_length:10000,no_script_tags"
email_field = "validate_email,strip_html" 
url_field = "validate_url,strip_javascript"
number_field = "validate_numeric,range:0-999999"

[snippet_rules]
# Snippet-specific field protection
["hero-banner"]
title = "strip_html,max_length:100,no_script_tags"
subtitle = "strip_html,max_length:200,no_script_tags"
background_url = "validate_url,strip_javascript,max_length:500"

["contact-form"] 
email = "validate_email,strip_html,lowercase"
message = "strip_html,max_length:5000,no_script_tags,profanity_filter"
name = "strip_html,max_length:100,alpha_spaces_only"

["product-card"]
price = "validate_numeric,range:0-99999,currency_format"
description = "strip_html,max_length:1000,no_script_tags"

# Tera template key references
[tera_rules]
# Protect template variables by key name
"user.email" = "validate_email,strip_html"
"content.title" = "strip_html,max_length:200,no_script_tags"
"form.message" = "strip_html,max_length:5000,content_filter"

[http_rules]
# HTTP-level protection (PF-style)
[http_rules.fetch]
allow_domains = ["api.example.com", "cdn.cloudflare.com"]
block_domains = ["malicious.com"]
max_response_size = "10MB"
timeout_seconds = 30

[http_rules.form_post]
max_fields = 50
max_field_size = "1MB" 
require_csrf = true
rate_limit = "10/minute"

[http_rules.form_get]
max_query_params = 20
max_param_length = 1000
```

## CLI Management (PF-Style)

### **Rule Management Commands**
```bash
# List all available rules
reed fw rules list
Available Rules:
  text_field        [6 rules, 3 enabled]
  email_field       [4 rules, 4 enabled]  
  url_field         [5 rules, 5 enabled]
  ai_content_check  [3 rules, 0 enabled] [PLUGIN]

# Show specific rule file
reed fw rules show text_field
Rule File: text_field.toml
├── strip_html        ✅ enabled
├── max_length        ✅ enabled (default_max: 10000)
├── no_script_tags    ✅ enabled  
├── normalize_ws      ❌ disabled
├── profanity_filter  ❌ disabled [PLUGIN DEP]
└── custom_filter     ❌ disabled [PLUGIN DEP]

# Enable/disable rules (PF-style syntax)
reed fw rules enable text_field.normalize_whitespace
reed fw rules disable text_field.profanity_filter

# Modify rule parameters (like pfctl -f)
reed fw rules set text_field.max_length.default_max 5000
reed fw rules set email_field.validation.strict_mode true

# Apply rule changes
reed fw rules reload                    # Reload all rule files
reed fw rules reload text_field         # Reload specific rule file

# Test rules against content
reed fw test text_field "Hello <script>alert('xss')</script> World"
Result: MODIFIED
Original: "Hello <script>alert('xss')</script> World" 
Filtered: "Hello  World"
Applied Rules: strip_html, no_script_tags

# Plugin rule management
reed fw rules install profanity_filter  # Install plugin rule
reed fw rules enable text_field.profanity_filter
```

### **Status and Monitoring**
```bash
# Firewall status
reed fw status
ReedCMS Content Firewall Status:
├── Rule Files: 5 loaded, 3 with active rules
├── Active Rules: 23 total (18 built-in, 5 plugin)
├── Blocked Content: 42 attempts in last 24h
├── Performance: avg 0.05ms per field
└── Last Reload: 2025-01-15 14:30:22

# Live filtering logs
reed fw logs tail
[2025-01-15 14:35:12] FILTERED hero-banner.title: stripped HTML tags
[2025-01-15 14:35:15] BLOCKED contact-form.message: script tag detected
[2025-01-15 14:35:18] MODIFIED product-card.description: length truncated (1205 -> 1000)

# Performance statistics
reed fw stats --interval 5s
Rules Performance (5s interval):
  text_field.strip_html: 45 calls, 0.02ms avg
  text_field.max_length: 45 calls, 0.01ms avg
  email_field.validate: 12 calls, 0.08ms avg
  url_field.validate: 8 calls, 0.15ms avg
```

## Rule Implementation

### **Built-in Rule Engine**
```rust
pub struct RuleEngine {
    rule_definitions: HashMap<String, RuleFile>,      // rule_name -> rule_file
    active_rules: HashMap<String, Vec<ActiveRule>>,   // field_type -> active_rules
    plugin_registry: PluginRegistry,
    builtin_implementations: BuiltinRules,
}

#[derive(Debug, Clone)]
pub struct RuleDefinition {
    pub enabled: bool,
    pub description: String,
    pub implementation: RuleImplementation,
    pub parameters: HashMap<String, Value>,
}

#[derive(Debug, Clone)]
pub enum RuleImplementation {
    RustBuiltin(String),          // Built-in Rust function
    Plugin(String),               // Plugin-provided function
    External(String),             // External command/script
}

impl RuleEngine {
    pub fn apply_rules(&self, field_type: &str, content: &str) -> Result<String, RuleError> {
        let rules = self.active_rules.get(field_type).unwrap_or(&vec![]);
        let mut filtered_content = content.to_string();
        
        for rule in rules {
            filtered_content = self.execute_rule(rule, &filtered_content)?;
        }
        
        Ok(filtered_content)
    }
    
    fn execute_rule(&self, rule: &ActiveRule, content: &str) -> Result<String, RuleError> {
        match &rule.implementation {
            RuleImplementation::RustBuiltin(func_name) => {
                self.builtin_implementations.execute(func_name, content, &rule.parameters)
            },
            RuleImplementation::Plugin(plugin_name) => {
                self.plugin_registry.execute_rule(plugin_name, &rule.name, content, &rule.parameters).await
            },
            RuleImplementation::External(command) => {
                self.execute_external_command(command, content, &rule.parameters).await
            }
        }
    }
}
```

### **Built-in Rule Implementations**
```rust
pub struct BuiltinRules;

impl BuiltinRules {
    pub fn execute(&self, func_name: &str, content: &str, params: &HashMap<String, Value>) -> Result<String, RuleError> {
        match func_name {
            "strip_html" => self.strip_html(content, params),
            "max_length" => self.max_length(content, params),
            "no_script_tags" => self.no_script_tags(content, params),
            "validate_email" => self.validate_email(content, params),
            "validate_url" => self.validate_url(content, params),
            "normalize_whitespace" => self.normalize_whitespace(content, params),
            _ => Err(RuleError::UnknownBuiltinRule(func_name.to_string()))
        }
    }
    
    fn strip_html(&self, content: &str, params: &HashMap<String, Value>) -> Result<String, RuleError> {
        let preserve_whitespace = params.get("preserve_whitespace")
            .and_then(|v| v.as_bool())
            .unwrap_or(true);
            
        let allowed_tags: Vec<String> = params.get("allowed_tags")
            .and_then(|v| v.as_array())
            .map(|arr| arr.iter().filter_map(|v| v.as_str().map(|s| s.to_string())).collect())
            .unwrap_or_default();
        
        let mut result = content.to_string();
        
        if allowed_tags.is_empty() {
            // Remove all HTML tags
            let tag_regex = regex::Regex::new(r"<[^>]*>").unwrap();
            result = tag_regex.replace_all(&result, if preserve_whitespace { " " } else { "" }).to_string();
        } else {
            // Remove only non-allowed tags (implementation)
            // Complex logic for selective tag removal
        }
        
        Ok(result)
    }
    
    fn max_length(&self, content: &str, params: &HashMap<String, Value>) -> Result<String, RuleError> {
        let default_max = params.get("default_max")
            .and_then(|v| v.as_u64())
            .unwrap_or(10000) as usize;
            
        let hard_limit = params.get("hard_limit")
            .and_then(|v| v.as_u64())
            .unwrap_or(50000) as usize;
        
        if content.len() > hard_limit {
            return Err(RuleError::ContentTooLong(hard_limit));
        }
        
        if content.len() > default_max {
            Ok(content.chars().take(default_max).collect())
        } else {
            Ok(content.to_string())
        }
    }
    
    fn no_script_tags(&self, content: &str, params: &HashMap<String, Value>) -> Result<String, RuleError> {
        let blocked_tags: Vec<String> = params.get("blocked_tags")
            .and_then(|v| v.as_array())
            .map(|arr| arr.iter().filter_map(|v| v.as_str().map(|s| s.to_lowercase())).collect())
            .unwrap_or_else(|| vec!["script".to_string()]);
            
        let blocked_protocols: Vec<String> = params.get("blocked_protocols")
            .and_then(|v| v.as_array())
            .map(|arr| arr.iter().filter_map(|v| v.as_str().map(|s| s.to_string())).collect())
            .unwrap_or_else(|| vec!["javascript:".to_string()]);
        
        let content_lower = content.to_lowercase();
        
        // Check for blocked tags
        for tag in &blocked_tags {
            if content_lower.contains(&format!("<{}", tag)) {
                return Err(RuleError::BlockedContent(format!("Contains blocked tag: {}", tag)));
            }
        }
        
        // Check for blocked protocols
        for protocol in &blocked_protocols {
            if content_lower.contains(protocol) {
                return Err(RuleError::BlockedContent(format!("Contains blocked protocol: {}", protocol)));
            }
        }
        
        Ok(content.to_string())
    }
}
```

## Integration Points

### **Snippet Processing Integration**
```rust
impl SnippetProcessor {
    pub async fn save_snippet(&self, snippet_type: &str, data: &mut SnippetData) -> Result<(), Error> {
        // Apply firewall rules to all fields
        for (field_name, content) in &mut data.fields {
            // Determine field type for rule selection
            let field_type = self.registry.get_field_type(snippet_type, field_name)
                .unwrap_or("text_field");
            
            // Apply content firewall
            match self.firewall.filter_field_content(field_type, content) {
                Ok(filtered) => *content = filtered,
                Err(e) => return Err(Error::ContentRejected(
                    format!("Field '{}' failed firewall rules: {}", field_name, e)
                )),
            }
        }
        
        // Save filtered content
        self.database.save_snippet(snippet_type, data).await?;
        Ok(())
    }
}
```

### **Tera Template Integration**
```rust
// Secure template function with automatic firewall
fn secure_string_function(args: &HashMap<String, Value>) -> Result<Value, Error> {
    let key = args.get("0").and_then(|v| v.as_str())
        .ok_or_else(|| Error::msg("Missing translation key"))?;
    
    // Get translation
    let translation = TRANSLATION_RESOLVER.resolve_key(key, locale)?;
    
    // Apply firewall rules based on template key
    let filtered = CONTENT_FIREWALL.filter_template_variable(key, &translation)?;
    
    Ok(Value::String(filtered))
}

// Usage in templates - automatically secured
// {{ string('user.email') }}        ← Automatic email validation + HTML stripping
// {{ string('content.title') }}     ← Automatic HTML stripping + length check
```

### **Plugin Rule Extension**
```rust
// Plugin can register new rules
impl PluginRuleRegistry {
    pub async fn register_plugin_rules(&mut self, plugin_name: &str) -> Result<(), PluginError> {
        // Load plugin rule definitions
        let rule_file_path = format!("config/firewall/rules/plugins/{}.toml", plugin_name);
        let rule_file = self.load_plugin_rule_file(&rule_file_path)?;
        
        // Register with main rule engine
        self.rule_engine.register_plugin_rules(plugin_name, rule_file).await?;
        
        Ok(())
    }
    
    pub async fn execute_plugin_rule(&self, plugin_name: &str, rule_name: &str, content: &str, params: &HashMap<String, Value>) -> Result<String, RuleError> {
        let request = PluginRuleRequest {
            rule_name: rule_name.to_string(),
            content: content.to_string(),
            parameters: params.clone(),
        };
        
        let response = self.send_to_plugin(plugin_name, request).await?;
        
        match response.status {
            RuleStatus::Pass => Ok(response.content),
            RuleStatus::Modified => Ok(response.content),
            RuleStatus::Blocked => Err(RuleError::ContentBlocked(response.reason)),
        }
    }
}
```

## Performance and Monitoring

### **Performance Characteristics**
```rust
// Typical performance metrics
pub struct FirewallMetrics {
    pub avg_processing_time_us: u64,    // Microseconds per field
    pub rules_executed_per_second: u32, // Rule execution rate
    pub memory_usage_mb: u64,            // Memory footprint
    pub cache_hit_rate: f64,             // Rule result caching
}

// Expected performance:
// - Simple rules (strip_html): 10-50 microseconds
// - Complex rules (regex): 50-200 microseconds
// - Plugin rules: 100-1000 microseconds
// - Overall field processing: <1ms typical
```

### **Error Handling and Logging**
```rust
#[derive(Debug, thiserror::Error)]
pub enum RuleError {
    #[error("Content blocked: {0}")]
    BlockedContent(String),
    
    #[error("Content too long: max {0} characters")]
    ContentTooLong(usize),
    
    #[error("Invalid format: {0}")]
    InvalidFormat(String),
    
    #[error("Plugin rule execution failed: {0}")]
    PluginExecutionFailed(String),
    
    #[error("Unknown built-in rule: {0}")]
    UnknownBuiltinRule(String),
}
```

## Security Benefits

### **Protection Layers**
1. **Field-Level Protection:** Every snippet field automatically protected
2. **Template Variable Protection:** All template output secured
3. **HTTP Input Validation:** Form data validated on entry
4. **Plugin Content Filtering:** Third-party content sanitized
5. **Configurable Policies:** Rules customizable per deployment

### **Attack Mitigation**
- **XSS Prevention:** Script tag and JavaScript protocol blocking
- **HTML Injection:** Automatic HTML tag stripping with whitelist
- **Content Length Attacks:** Configurable length limits per field type
- **Data Validation:** Email, URL, and numeric format validation
- **Profanity Filtering:** Optional content appropriateness checking

---

**Result:** A comprehensive, PF-inspired content firewall system providing field-level protection with modular rules, CLI management, plugin extensibility, and real-time configuration - making ReedCMS content security operationally manageable while maintaining performance.