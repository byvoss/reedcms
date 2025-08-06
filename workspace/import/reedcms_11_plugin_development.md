# ReedCMS-11-Plugin-Development.md

## Plugin SDK Architecture

**Unified Development Experience:** Both LUA and Rust plugins use the same binary protocol and communication patterns, providing consistent development workflows regardless of chosen language.

**Plugin SDK Layers:**
- **Protocol Layer:** Binary message handling and socket communication
- **Command Layer:** High-level command abstractions and routing
- **Helper Layer:** Common utilities for validation, logging, and error handling

## LUA Plugin Development

### LUA Plugin SDK Structure
```lua
-- reed-cms-plugin-sdk.lua
local sdk = {}

-- Core plugin interface
function sdk.create_plugin(config)
    local plugin = {
        name = config.name,
        version = config.version,
        socket_path = config.socket_path,
        command_handlers = {},
        message_handlers = {}
    }
    
    -- Plugin lifecycle methods
    function plugin:register_command(cmd_id, handler)
        self.command_handlers[cmd_id] = handler
    end
    
    function plugin:register_message_handler(msg_type, handler)
        self.message_handlers[msg_type] = handler
    end
    
    function plugin:send_signal(signal_type, data)
        return sdk.send_signal(self.socket_path, signal_type, data)
    end
    
    function plugin:send_plugin_message(target_plugin, msg_type, payload)
        return sdk.send_plugin_message(self.name, target_plugin, msg_type, payload)
    end
    
    function plugin:run()
        return sdk.run_plugin_event_loop(self)
    end
    
    return plugin
end

return sdk
```

### Complete LUA Plugin Example
```lua
-- simple-seo-plugin.lua
local sdk = require("reed-cms-plugin-sdk")

-- Create plugin instance
local plugin = sdk.create_plugin({
    name = "simple-seo",
    version = "1.2.0",
    socket_path = "/tmp/reed-cms/plugins/simple-seo.sock"
})

-- SEO validation command handler
plugin:register_command(0x30, function(payload)
    local snippet = json.decode(payload)
    local issues = {}
    
    -- Title length check
    if snippet.content and snippet.content.title then
        local title_len = string.len(snippet.content.title)
        if title_len < 10 then
            table.insert(issues, {
                type = "warning",
                field = "title", 
                message = "Title too short for SEO (minimum 10 characters)",
                suggestion = "Add descriptive keywords to title"
            })
        elseif title_len > 60 then
            table.insert(issues, {
                type = "error",
                field = "title",
                message = "Title too long for SEO (maximum 60 characters)", 
                suggestion = "Shorten title, move details to meta description"
            })
        end
    end
    
    -- Meta description check
    if snippet.content and snippet.content.meta_description then
        local meta_len = string.len(snippet.content.meta_description)
        if meta_len > 0 and meta_len < 120 then
            table.insert(issues, {
                type = "warning",
                field = "meta_description",
                message = "Meta description too short (minimum 120 characters)",
                suggestion = "Add more descriptive content"
            })
        elseif meta_len > 160 then
            table.insert(issues, {
                type = "error", 
                field = "meta_description",
                message = "Meta description too long (maximum 160 characters)",
                suggestion = "Trim description to essential information"
            })
        end
    else
        table.insert(issues, {
            type = "error",
            field = "meta_description", 
            message = "Missing meta description",
            suggestion = "Add meta description for better SEO"
        })
    end
    
    -- Return validation results
    local response = {
        status = (#issues == 0) and 200 or 400,
        issues = issues,
        seo_score = calculate_seo_score(snippet, issues)
    }
    
    return json.encode(response)
end)

-- Keyword suggestion command handler  
plugin:register_command(0x31, function(payload)
    local request = json.decode(payload)
    local content = request.content or ""
    
    -- Simple keyword extraction
    local keywords = extract_keywords(content)
    local suggestions = generate_keyword_suggestions(keywords)
    
    local response = {
        status = 200,
        extracted_keywords = keywords,
        suggestions = suggestions,
        keyword_density = calculate_keyword_density(content, keywords)
    }
    
    return json.encode(response)
end)

-- Inter-plugin message handler (receives notifications from AI plugin)
plugin:register_message_handler(0x01, function(message)
    local data = json.decode(message.payload)
    
    -- Log AI optimization for analytics
    log_info("Received AI keyword optimization for snippet: " .. data.snippet_id)
    
    -- Send analytics update
    plugin:send_plugin_message("analytics", 0x10, json.encode({
        event_type = "ai_optimization",
        snippet_id = data.snippet_id,
        optimization_score = data.optimization_score,
        timestamp = os.time()
    }))
    
    -- Acknowledge receipt
    return { status = 200, message = "AI optimization logged" }
end)

-- Helper functions
function calculate_seo_score(snippet, issues)
    local base_score = 100
    local penalty = 0
    
    for _, issue in ipairs(issues) do
        if issue.type == "error" then
            penalty = penalty + 20
        elseif issue.type == "warning" then  
            penalty = penalty + 10
        end
    end
    
    return math.max(0, base_score - penalty)
end

function extract_keywords(content)
    local keywords = {}
    local stop_words = {"the", "and", "or", "but", "in", "on", "at", "to", "for", "of", "with", "by"}
    
    -- Simple word extraction and filtering
    for word in string.gmatch(string.lower(content), "%w+") do
        if string.len(word) > 3 and not table_contains(stop_words, word) then
            keywords[word] = (keywords[word] or 0) + 1
        end
    end
    
    -- Convert to sorted array
    local result = {}
    for word, count in pairs(keywords) do
        table.insert(result, {word = word, count = count})
    end
    
    table.sort(result, function(a, b) return a.count > b.count end)
    return result
end

function generate_keyword_suggestions(keywords)
    local suggestions = {}
    
    for i, kw in ipairs(keywords) do
        if i <= 5 then -- Top 5 keywords
            table.insert(suggestions, {
                keyword = kw.word,
                suggestion = "Consider using '" .. kw.word .. "' in title or headings",
                priority = "high"
            })
        end
    end
    
    return suggestions
end

function calculate_keyword_density(content, keywords)
    local total_words = 0
    for _ in string.gmatch(content, "%w+") do
        total_words = total_words + 1
    end
    
    local densities = {}
    for _, kw in ipairs(keywords) do
        if kw.count > 0 then
            densities[kw.word] = (kw.count / total_words) * 100
        end
    end
    
    return densities
end

function table_contains(table, element)
    for _, value in pairs(table) do
        if value == element then
            return true
        end
    end
    return false
end

function log_info(message)
    print("[" .. os.date("%Y-%m-%d %H:%M:%S") .. "] [INFO] " .. message)
end

-- Start plugin event loop
plugin:run()
```

### LUA Plugin Deployment
```bash
# LUA plugin structure
plugins/simple-seo/
├── simple-seo.lua              # Main plugin code
├── plugin.toml                 # Plugin configuration
├── README.md                   # Documentation
└── tests/
    └── test_seo_validation.lua  # Unit tests

# Plugin configuration
# plugin.toml
[plugin]
name = "simple-seo"
version = "1.2.0"
type = "lua"
description = "SEO validation and optimization suggestions"

[runtime]
socket_path = "/tmp/reed-cms/plugins/simple-seo.sock"
auto_start = false
sleep_timeout_ms = 180000

[permissions]
required = ["snippet_read"]
user_groups = ["editor", "admin"]

[commands]
seo_validate = 0x30
keyword_suggest = 0x31

[messages]
ai_optimization = 0x01
```

## Rust Pro Plugin Development

### Rust Plugin SDK Architecture
```rust
// reed-cms-plugin-sdk/src/lib.rs
use tokio::net::UnixStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use serde::{Serialize, Deserialize};
use std::collections::HashMap;

#[repr(C, packed)]
pub struct ReedMessage {
    pub magic: u32,
    pub version: u8,
    pub command_id: u8,
    pub flags: u8,
    pub reserved: u8,
    pub payload_len: u32,
}

pub trait ReedPlugin {
    async fn handle_command(&mut self, cmd_id: u8, payload: &[u8]) -> Result<Vec<u8>, PluginError>;
    async fn handle_signal(&mut self, signal: SignalMessage) -> Result<(), PluginError>;
    async fn handle_plugin_message(&mut self, message: PluginMessage) -> Result<Vec<u8>, PluginError>;
}

pub struct PluginRuntime {
    socket: UnixStream,
    plugin: Box<dyn ReedPlugin + Send>,
    command_stats: HashMap<u8, CommandStats>,
}

impl PluginRuntime {
    pub async fn new(socket_path: &str, plugin: Box<dyn ReedPlugin + Send>) -> Result<Self, PluginError> {
        let socket = UnixStream::connect(socket_path).await?;
        
        Ok(PluginRuntime {
            socket,
            plugin,
            command_stats: HashMap::new(),
        })
    }
    
    pub async fn register_and_run(&mut self) -> Result<(), PluginError> {
        // Send plugin registration
        self.send_plugin_register().await?;
        
        // Send ready signal
        self.send_signal(SIGNAL_PLUGIN_READY, &[]).await?;
        
        // Main event loop
        self.event_loop().await
    }
    
    async fn event_loop(&mut self) -> Result<(), PluginError> {
        let mut buffer = vec![0u8; 65536]; // 64KB buffer
        
        loop {
            // Read message header
            let header_bytes = self.socket.read_exact(&mut buffer[..12]).await?;
            let header: ReedMessage = unsafe { 
                std::ptr::read(buffer.as_ptr() as *const ReedMessage) 
            };
            
            // Validate magic bytes
            if header.magic != 0x52454544 { // "REED"
                return Err(PluginError::InvalidMessage);
            }
            
            // Read payload if present
            let mut payload = vec![0u8; header.payload_len as usize];
            if header.payload_len > 0 {
                self.socket.read_exact(&mut payload).await?;
            }
            
            // Handle message based on flags
            let response = if header.flags & FLAG_SIGNAL != 0 {
                // Handle signal
                let signal: SignalMessage = bincode::deserialize(&payload)?;
                self.plugin.handle_signal(signal).await?;
                None
            } else {
                // Handle command
                let start_time = std::time::Instant::now();
                let result = self.plugin.handle_command(header.command_id, &payload).await;
                
                // Record performance stats
                self.record_command_stats(header.command_id, start_time.elapsed());
                
                Some(result?)
            };
            
            // Send response if not async
            if header.flags & FLAG_ASYNC == 0 {
                if let Some(response_data) = response {
                    self.send_response(200, &response_data).await?;
                }
            }
        }
    }
    
    async fn send_response(&mut self, status: u16, data: &[u8]) -> Result<(), PluginError> {
        let response_header = ReedMessage {
            magic: 0x52454544,
            version: 0x01,
            command_id: 0x00, // Response
            flags: 0x00,
            reserved: 0x00,
            payload_len: data.len() as u32 + 2, // +2 for status
        };
        
        // Send header
        self.socket.write_all(unsafe {
            std::slice::from_raw_parts(&response_header as *const _ as *const u8, 12)
        }).await?;
        
        // Send status
        self.socket.write_all(&status.to_le_bytes()).await?;
        
        // Send data
        self.socket.write_all(data).await?;
        
        Ok(())
    }
    
    fn record_command_stats(&mut self, cmd_id: u8, duration: std::time::Duration) {
        let stats = self.command_stats.entry(cmd_id).or_insert_with(CommandStats::new);
        stats.record_execution(duration);
    }
}

#[derive(Debug)]
pub struct CommandStats {
    pub call_count: u64,
    pub total_duration: std::time::Duration,
    pub avg_duration: std::time::Duration,
    pub max_duration: std::time::Duration,
}

impl CommandStats {
    fn new() -> Self {
        CommandStats {
            call_count: 0,
            total_duration: std::time::Duration::ZERO,
            avg_duration: std::time::Duration::ZERO,
            max_duration: std::time::Duration::ZERO,
        }
    }
    
    fn record_execution(&mut self, duration: std::time::Duration) {
        self.call_count += 1;
        self.total_duration += duration;
        self.avg_duration = self.total_duration / self.call_count as u32;
        if duration > self.max_duration {
            self.max_duration = duration;
        }
    }
}

// Error types
#[derive(Debug, thiserror::Error)]
pub enum PluginError {
    #[error("Invalid message format")]
    InvalidMessage,
    #[error("Unknown command: {0}")]
    UnknownCommand(u8),
    #[error("Permission denied")]
    PermissionDenied,
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    #[error("Serialization error: {0}")]
    Serialization(#[from] bincode::Error),
}
```

### Complete Rust Pro Plugin Example
```rust
// ai-core-plugin/src/main.rs
use reed_cms_plugin_sdk::*;
use tokio::time::{timeout, Duration};
use serde::{Serialize, Deserialize};
use std::collections::HashMap;

#[derive(Serialize, Deserialize)]
struct KeywordOptimizeRequest {
    snippet_id: String,
    content: String,
    target_audience: String,
    current_keywords: Vec<String>,
}

#[derive(Serialize, Deserialize)]
struct KeywordOptimizeResponse {
    status: u16,
    optimized_keywords: Vec<String>,
    confidence_score: f64,
    removed_keywords: Vec<String>,
    added_keywords: Vec<String>,
}

#[derive(Serialize, Deserialize)]
struct ContentGenerateRequest {
    prompt: String,
    snippet_type: String,
    target_length: usize,
    tone: String,
}

#[derive(Serialize, Deserialize)]
struct ContentGenerateResponse {
    status: u16,
    generated_content: String,
    alternative_versions: Vec<String>,
    quality_score: f64,
}

struct AICorePlugin {
    ai_client: AIClient,
    optimization_cache: HashMap<String, OptimizationResult>,
    performance_stats: PerformanceTracker,
}

impl AICorePlugin {
    async fn new() -> Result<Self, PluginError> {
        let ai_client = AIClient::new().await?;
        
        Ok(AICorePlugin {
            ai_client,
            optimization_cache: HashMap::new(),
            performance_stats: PerformanceTracker::new(),
        })
    }
}

impl ReedPlugin for AICorePlugin {
    async fn handle_command(&mut self, cmd_id: u8, payload: &[u8]) -> Result<Vec<u8>, PluginError> {
        match cmd_id {
            0x40 => self.optimize_keywords(payload).await,
            0x41 => self.generate_content(payload).await,
            0x42 => self.analyze_seo_potential(payload).await,
            0x43 => self.get_performance_stats().await,
            _ => Err(PluginError::UnknownCommand(cmd_id))
        }
    }
    
    async fn handle_signal(&mut self, signal: SignalMessage) -> Result<(), PluginError> {
        match signal.signal_type {
            SIGNAL_HEALTH_CHECK => {
                // Health check - verify AI client connectivity
                self.ai_client.health_check().await?;
                Ok(())
            },
            SIGNAL_RELOAD_CONFIG => {
                // Reload AI configuration
                self.ai_client.reload_config().await?;
                Ok(())
            },
            _ => Ok(())
        }
    }
    
    async fn handle_plugin_message(&mut self, message: PluginMessage) -> Result<Vec<u8>, PluginError> {
        match message.message_type {
            0x10 => {
                // Analytics requesting optimization stats
                let stats = self.performance_stats.get_summary();
                Ok(bincode::serialize(&stats)?)
            },
            _ => Ok(vec![])
        }
    }
}

impl AICorePlugin {
    async fn optimize_keywords(&mut self, payload: &[u8]) -> Result<Vec<u8>, PluginError> {
        let request: KeywordOptimizeRequest = bincode::deserialize(payload)?;
        
        // Check cache first
        if let Some(cached) = self.optimization_cache.get(&request.snippet_id) {
            if cached.is_fresh() {
                return Ok(bincode::serialize(&cached.response)?);
            }
        }
        
        // Perform AI optimization with timeout
        let optimization_future = self.ai_client.optimize_keywords(
            &request.content,
            &request.target_audience,
            &request.current_keywords
        );
        
        let result = timeout(Duration::from_secs(30), optimization_future).await
            .map_err(|_| PluginError::Timeout)?
            .map_err(|e| PluginError::AIError(e))?;
        
        let response = KeywordOptimizeResponse {
            status: 200,
            optimized_keywords: result.keywords,
            confidence_score: result.confidence,
            removed_keywords: result.removed,
            added_keywords: result.added,
        };
        
        // Cache result
        self.optimization_cache.insert(
            request.snippet_id.clone(),
            OptimizationResult::new(response.clone())
        );
        
        // Notify other plugins of optimization
        self.notify_keyword_optimization(&request.snippet_id, &response.optimized_keywords).await?;
        
        Ok(bincode::serialize(&response)?)
    }
    
    async fn generate_content(&mut self, payload: &[u8]) -> Result<Vec<u8>, PluginError> {
        let request: ContentGenerateRequest = bincode::deserialize(payload)?;
        
        // Parallel generation for multiple versions
        let (primary_future, alt1_future, alt2_future) = tokio::join!(
            self.ai_client.generate_content(&request.prompt, &request.tone),
            self.ai_client.generate_content(&format!("{} (alternative style)", request.prompt), "professional"),
            self.ai_client.generate_content(&format!("{} (creative style)", request.prompt), "creative")
        );
        
        let primary_content = primary_future?;
        let alt1_content = alt1_future.unwrap_or_default();
        let alt2_content = alt2_future.unwrap_or_default();
        
        // Quality scoring
        let quality_score = self.calculate_content_quality(&primary_content, &request);
        
        let response = ContentGenerateResponse {
            status: 200,
            generated_content: primary_content,
            alternative_versions: vec![alt1_content, alt2_content],
            quality_score,
        };
        
        Ok(bincode::serialize(&response)?)
    }
    
    async fn analyze_seo_potential(&mut self, payload: &[u8]) -> Result<Vec<u8>, PluginError> {
        // SEO analysis implementation
        let content: String = bincode::deserialize(payload)?;
        let analysis = self.ai_client.analyze_seo_potential(&content).await?;
        Ok(bincode::serialize(&analysis)?)
    }
    
    async fn get_performance_stats(&self) -> Result<Vec<u8>, PluginError> {
        let stats = self.performance_stats.get_detailed_stats();
        Ok(bincode::serialize(&stats)?)
    }
    
    async fn notify_keyword_optimization(&self, snippet_id: &str, keywords: &[String]) -> Result<(), PluginError> {
        // Send notification to analytics plugin via FIFO
        let message = PluginMessage {
            magic: 0x504C4D47,
            version: 0x01,
            message_type: 0x01, // MSG_KEYWORD_OPTIMIZED
            source_plugin: b"ai-core\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0",
            target_plugin: b"analytics\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0",
            timestamp: std::time::SystemTime::now().duration_since(std::time::UNIX_EPOCH)?.as_micros() as u64,
            payload_len: 0,
        };
        
        // Send via FIFO (implementation would use FIFO manager)
        // fifo_manager.send_plugin_message(message).await?;
        
        Ok(())
    }
    
    fn calculate_content_quality(&self, content: &str, request: &ContentGenerateRequest) -> f64 {
        let mut score = 0.0;
        
        // Length scoring
        let length_ratio = content.len() as f64 / request.target_length as f64;
        if length_ratio >= 0.8 && length_ratio <= 1.2 {
            score += 25.0;
        }
        
        // Readability scoring (simplified)
        let sentences = content.split('.').count();
        let words = content.split_whitespace().count();
        if sentences > 0 {
            let avg_sentence_length = words as f64 / sentences as f64;
            if avg_sentence_length >= 10.0 && avg_sentence_length <= 25.0 {
                score += 25.0;
            }
        }
        
        // Keyword presence (basic check)
        if content.to_lowercase().contains(&request.snippet_type.to_lowercase()) {
            score += 25.0;
        }
        
        // Tone consistency (simplified)
        let tone_words = match request.tone.as_str() {
            "professional" => vec!["solution", "expertise", "professional", "quality"],
            "creative" => vec!["innovative", "unique", "creative", "inspiring"],
            "casual" => vec!["easy", "simple", "friendly", "approachable"],
            _ => vec![]
        };
        
        let tone_matches = tone_words.iter()
            .filter(|word| content.to_lowercase().contains(*word))
            .count();
        
        if tone_matches > 0 {
            score += 25.0;
        }
        
        score / 100.0 // Normalize to 0-1 range
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let plugin = AICorePlugin::new().await?;
    let mut runtime = PluginRuntime::new(
        "/tmp/reed-cms/plugins/ai-core.sock",
        Box::new(plugin)
    ).await?;
    
    runtime.register_and_run().await?;
    Ok(())
}
```

### Rust Plugin Deployment
```toml
# Cargo.toml
[package]
name = "ai-core-plugin"
version = "1.0.0"
edition = "2021"

[dependencies]
reed-cms-plugin-sdk = { path = "../reed-cms-plugin-sdk" }
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
bincode = "1.3"
thiserror = "1.0"
reqwest = { version = "0.11", features = ["json"] }
tracing = "0.1"

# Plugin configuration
# plugin.toml
[plugin]
name = "ai-core"
version = "1.0.0"
type = "native"
description = "AI-powered content optimization and generation"

[runtime]
socket_path = "/tmp/reed-cms/plugins/ai-core.sock"
auto_start = true
sleep_timeout_ms = 300000

[permissions]
required = ["snippet_read", "snippet_write", "search_index"]
user_groups = ["admin"]

[commands]
optimize_keywords = 0x40
generate_content = 0x41
analyze_seo = 0x42
performance_stats = 0x43

[ai_config]
provider = "openai"
model = "gpt-4"
max_tokens = 2000
timeout_seconds = 30
```

## Plugin Testing Framework

### LUA Plugin Testing
```lua
-- tests/test_seo_validation.lua
local plugin = require("../simple-seo")
local test = require("test-framework")

test.describe("SEO Validation", function()
    test.it("should validate title length", function()
        local snippet = {
            content = {
                title = "Short"
            }
        }
        
        local result = plugin.validate_seo(snippet)
        test.assert(result.status == 400)
        test.assert(#result.issues > 0)
        test.assert(result.issues[1].field == "title")
    end)
    
    test.it("should pass valid content", function()
        local snippet = {
            content = {
                title = "This is a properly sized title for SEO validation",
                meta_description = "This is a comprehensive meta description that provides enough detail for search engines while staying within the recommended length limits."
            }
        }
        
        local result = plugin.validate_seo(snippet)
        test.assert(result.status == 200)
        test.assert(#result.issues == 0)
        test.assert(result.seo_score > 80)
    end)
end)
```

### Rust Plugin Testing
```rust
// tests/integration_tests.rs
use ai_core_plugin::*;
use reed_cms_plugin_sdk::*;

#[tokio::test]
async fn test_keyword_optimization() {
    let mut plugin = AICorePlugin::new().await.unwrap();
    
    let request = KeywordOptimizeRequest {
        snippet_id: "test-snippet".to_string(),
        content: "This is test content about modern web development".to_string(),
        target_audience: "developers".to_string(),
        current_keywords: vec!["web".to_string(), "development".to_string()],
    };
    
    let payload = bincode::serialize(&request).unwrap();
    let response_bytes = plugin.handle_command(0x40, &payload).await.unwrap();
    let response: KeywordOptimizeResponse = bincode::deserialize(&response_bytes).unwrap();
    
    assert_eq!(response.status, 200);
    assert!(!response.optimized_keywords.is_empty());
    assert!(response.confidence_score > 0.0);
}

#[tokio::test]
async fn test_content_generation() {
    let mut plugin = AICorePlugin::new().await.unwrap();
    
    let request = ContentGenerateRequest {
        prompt: "Write about sustainable web development".to_string(),
        snippet_type: "blog-post".to_string(),
        target_length: 500,
        tone: "professional".to_string(),
    };
    
    let payload = bincode::serialize(&request).unwrap();
    let response_bytes = plugin.handle_command(0x41, &payload).await.unwrap();
    let response: ContentGenerateResponse = bincode::deserialize(&response_bytes).unwrap();
    
    assert_eq!(response.status, 200);
    assert!(!response.generated_content.is_empty());
    assert!(response.quality_score > 0.5);
    assert_eq!(response.alternative_versions.len(), 2);
}
```

This development guide provides comprehensive examples and patterns for building both LUA and Rust plugins, ensuring consistent development experience while leveraging the strengths of each language.

---

**Next: ReedCMS-12-Plugin-Registry.md** - Plugin lifecycle management, permission system, and deployment workflows.