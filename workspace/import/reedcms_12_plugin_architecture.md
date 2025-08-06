# ReedCMS-10-Plugin-Architecture.md

## Plugin Philosophy

**Unix Domain Sockets + Binary Protocol = Zero Overhead Extension System**

ReedCMS plugins run as isolated processes communicating through high-performance Unix Domain Sockets with a lean binary protocol. Plugin crashes never affect core system stability while maintaining sub-millisecond performance.

**Two-Language Plugin System:**
- **LUA Latest (Plugin):** Simple, lightweight plugins for content processing, validation, and workflows
- **Rust Latest (Pro Plugin):** High-performance, system-level plugins for critical operations

**KISS-Brain Development:** Fixed struct definitions, HTTP status codes, and predictable communication patterns.

## Architecture Overview

### Three-Layer Plugin Communication
```
┌─────────────────────────────────────────────────────────┐
│                Plugin-to-Plugin Layer                   │
│   FIFO pipes for inter-plugin communication            │
│   LUA ↔ LUA, Rust ↔ Rust, LUA ↔ Rust communication    │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                Core Communication Layer                 │
│   Unix Domain Sockets + Binary Protocol                │
│   Request/Response + Bidirectional Signals             │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                Plugin Registry Layer                    │
│   CSV-driven plugin definitions and permissions        │
│   LUA and Rust process lifecycle management            │
└─────────────────────────────────────────────────────────┘
```

## Socket File Structure

```
/tmp/reed-cms/
├── core.sock                    # Main CLI-API endpoint
├── plugins/
│   ├── ai-core.sock              # Rust Pro Plugin
│   ├── analytics.sock            # Rust Pro Plugin
│   ├── simple-seo.sock          # LUA Plugin
│   └── form-validator.sock      # LUA Plugin
├── fifos/
│   ├── ai-core→analytics         # AI sends to Analytics
│   ├── analytics→ai-core         # Analytics responds to AI
│   ├── simple-seo→analytics      # LUA to Rust communication
│   └── analytics→simple-seo      # Rust to LUA response
└── pid/
    ├── core.pid
    └── plugins/
        ├── ai-core.pid          # Rust binary process
        └── simple-seo.pid       # LUA runtime process
```

## Binary Protocol Specification

### Message Format
```rust
#[repr(C, packed)]
struct ReedMessage {
    magic: u32,           // "REED" (0x52454544) magic bytes
    version: u8,          // Protocol version (0x01)
    command_id: u8,       // Command identifier (0-255)
    flags: u8,            // Message flags (async, signal, etc.)
    reserved: u8,         // Reserved for future use
    payload_len: u32,     // Payload length (max 4GB, practically <64KB)
    // Variable payload follows immediately
}

// Message flags
const FLAG_ASYNC: u8 = 0x01;        // Async operation, no response expected
const FLAG_SIGNAL: u8 = 0x02;       // Signal message
const FLAG_BROADCAST: u8 = 0x04;    // Broadcast to all plugins
```

### Core Command IDs
```rust
// Snippet Operations (0x01-0x0F)
const CMD_SNIPPET_CREATE: u8 = 0x01;
const CMD_SNIPPET_UPDATE: u8 = 0x02;
const CMD_SNIPPET_DELETE: u8 = 0x03;
const CMD_SNIPPET_GET: u8 = 0x04;
const CMD_SNIPPET_LIST: u8 = 0x05;

// Schema Operations (0x10-0x1F)
const CMD_SCHEMA_BOND: u8 = 0x10;
const CMD_SCHEMA_UNBOND: u8 = 0x11;
const CMD_SCHEMA_TREE: u8 = 0x12;

// Registry Operations (0x20-0x2F)
const CMD_REGISTRY_DEFINE: u8 = 0x20;
const CMD_REGISTRY_GET: u8 = 0x21;
const CMD_REGISTRY_LIST: u8 = 0x22;

// Search Operations (0x30-0x3F)
const CMD_SEARCH_QUERY: u8 = 0x30;
const CMD_SEARCH_INDEX: u8 = 0x31;

// Plugin System (0xF0-0xFF)
const CMD_PLUGIN_REGISTER: u8 = 0xF0;
const CMD_PLUGIN_HEALTH: u8 = 0xF2;
const CMD_PLUGIN_SIGNAL: u8 = 0xF3;
```

### Fixed Struct Examples
```rust
#[repr(C, packed)]
struct SnippetCreateRequest {
    snippet_type: [u8; 32],      // Null-terminated snippet type name
    semantic_name: [u8; 32],     // Null-terminated semantic name
    content_len: u32,            // JSON content length
    // JSON content follows
}

#[repr(C, packed)]
struct SnippetCreateResponse {
    status: u16,                 // HTTP status code
    snippet_id: [u8; 36],        // UUID string (null-terminated)
    error_len: u16,              // Error message length (0 if success)
    // Error message follows if error_len > 0
}
```

## Signal System

```rust
// Plugin → Core Signals
const SIGNAL_PLUGIN_READY: u8 = 0x01;      // Plugin fully initialized
const SIGNAL_PLUGIN_BUSY: u8 = 0x02;       // Plugin processing request
const SIGNAL_PLUGIN_SLEEPING: u8 = 0x04;   // Plugin entered sleep mode
const SIGNAL_PLUGIN_ERROR: u8 = 0x05;      // Plugin encountered error

// Core → Plugin Signals
const SIGNAL_SHUTDOWN_GRACEFUL: u8 = 0x10; // Graceful shutdown request
const SIGNAL_RELOAD_CONFIG: u8 = 0x12;     // Reload configuration
const SIGNAL_HEALTH_CHECK: u8 = 0x13;      // Health check ping
const SIGNAL_WAKE_UP: u8 = 0x15;           // Wake from sleep mode

#[repr(C, packed)]
struct SignalMessage {
    signal_type: u8,             // Signal type from constants above
    timestamp: u64,              // Unix timestamp microseconds
    plugin_name: [u8; 32],       // Source/target plugin name
    data_len: u16,               // Additional signal data length
    // Additional data follows if data_len > 0
}
```

## Plugin-to-Plugin Communication

### FIFO-Based Bidirectional Messaging
```rust
// Universal message format for LUA ↔ Rust communication
#[repr(C, packed)]
struct PluginMessage {
    magic: u32,                  // "PLMG" (0x504C4D47)
    version: u8,                 // Message format version
    message_type: u8,            // Message type identifier
    source_plugin: [u8; 32],     // Source plugin name
    target_plugin: [u8; 32],     // Target plugin name
    timestamp: u64,              // Unix timestamp microseconds
    payload_len: u32,            // Payload length
    // JSON payload follows
}

// Standard message types
const MSG_KEYWORD_OPTIMIZED: u8 = 0x01;   // AI → Analytics: Keywords updated
const MSG_CONTENT_ANALYZED: u8 = 0x02;    // Analytics → Backup: Content stats
const MSG_RESPONSE_ACK: u8 = 0xF0;        // Standard acknowledgment
const MSG_RESPONSE_ERROR: u8 = 0xF1;      // Standard error response
const MSG_RESPONSE_DATA: u8 = 0xF2;       // Standard data response
```

**Auto-Pair Creation:** Every plugin pair gets bidirectional FIFOs automatically created for request-response patterns.

## Plugin Language Types

### LUA Plugins (Latest Stable)
**Use Case:** Content processing, validation, workflows, SEO checks, form handling

**Characteristics:**
- Runtime: Embedded Lua latest stable
- Performance: 5-200ms operations  
- Memory: 1-8MB per plugin
- Development: Rapid prototyping, simple syntax

**When to Choose:**
- Content validation and filtering
- Form processing and validation
- Simple SEO checks and suggestions
- Workflow automation
- Configuration-driven functionality

### Rust Pro Plugins (Latest Stable)  
**Use Case:** AI/ML integration, high-performance operations, system-level functionality

**Characteristics:**
- Runtime: Native compiled binary
- Performance: Sub-millisecond to 30s operations
- Memory: 5-50MB per plugin  
- Development: Type safety, system integration

**When to Choose:**
- AI/ML integration and processing
- High-performance search operations
- Database integration and migrations
- System monitoring and analytics
- External API integrations requiring parallelism
- Memory-intensive operations

## Plugin Registry Integration

### Plugin Definition CSV
```csv
# plugin-registry.csv
plugin_name,plugin_type,socket_path,version,required_permissions,user_groups,auto_start,description
ai-core,native,/tmp/reed-cms/plugins/ai-core.sock,1.0.0,"snippet_read,snippet_write","admin",true,"AI optimization (Rust Pro)"
simple-seo,lua,/tmp/reed-cms/plugins/simple-seo.sock,1.2.0,"snippet_read","editor,admin",false,"SEO validation (LUA)"
form-validator,lua,/tmp/reed-cms/plugins/form-validator.sock,2.1.0,"snippet_read","editor",false,"Form validation (LUA)"
analytics,native,/tmp/reed-cms/plugins/analytics.sock,1.0.0,"system_stats","admin",false,"Analytics (Rust Pro)"
```

### Permission System
```rust
// Plugin permissions integrated with UCG
const PERM_SNIPPET_READ: &str = "snippet_read";
const PERM_SNIPPET_WRITE: &str = "snippet_write";
const PERM_SCHEMA_READ: &str = "schema_read";
const PERM_REGISTRY_READ: &str = "registry_read";
const PERM_SEARCH_READ: &str = "search_read";
const PERM_SYSTEM_STATS: &str = "system_stats";
const PERM_FILE_WRITE: &str = "file_write";
```

## CLI Integration

### Plugin Management Commands
```bash
# Plugin lifecycle (works for both LUA and Rust)
reed plugin list                    # List all available plugins
reed plugin status                  # Show plugin health status
reed plugin start ai-core           # Start Rust Pro Plugin
reed plugin start simple-seo        # Start LUA Plugin
reed plugin stop analytics          # Stop plugin gracefully

# Plugin communication (same interface for both types)
reed plugin send ai-core optimize $snippet_id        # Send to Rust plugin
reed plugin send simple-seo validate $snippet_id     # Send to LUA plugin
reed plugin signal analytics wake                    # Send signal to plugin

# Development tools
reed plugin scaffold new-lua-plugin       # Generate LUA plugin template
reed plugin scaffold new-rust-plugin      # Generate Rust plugin template
reed plugin test ai-core                  # Run plugin health checks
```

## Performance Characteristics

### By Plugin Type
```rust
// LUA Plugin Benchmarks
LUA Plugin startup:           500-2000ms
Simple validation:            5-50ms
Content filtering:            10-100ms
Memory per plugin:            1-5MB
Concurrent plugins:           20-50 supported

// Rust Pro Plugin Benchmarks  
Rust Plugin startup:          100-500ms
AI operations:                50-30000ms (API dependent)
Analytics operations:         0.5-10ms
Memory per plugin:            5-50MB
Concurrent plugins:           10-20 supported
```

### System Scaling
```rust
// Memory usage by plugin mix
5 LUA + 2 Rust plugins:       25-75MB total
10 LUA + 5 Rust plugins:      80-200MB total
20 LUA + 10 Rust plugins:     200-500MB total

// Binary protocol performance
Socket connection setup:      < 1ms
Simple commands:              < 0.5ms
Complex commands:             < 5ms
Signal propagation:           < 0.1ms
```

## Error Handling

### Error Response Format
```rust
#[repr(C, packed)]
struct ErrorResponse {
    status: u16,                 // HTTP status code (400, 403, 500, etc.)
    error_code: u32,             // ReedCMS specific error code
    error_len: u16,              // Error message length
    // Error message follows (UTF-8)
}

// ReedCMS error codes
const ERR_PLUGIN_NOT_FOUND: u32 = 10001;
const ERR_PLUGIN_OFFLINE: u32 = 10002;
const ERR_PLUGIN_TIMEOUT: u32 = 10003;
const ERR_PERMISSION_DENIED: u32 = 10004;
```

This plugin architecture provides maximum flexibility through the LUA/Rust dual-language approach while maintaining ReedCMS's core performance and reliability principles through process isolation, binary protocol efficiency, and graceful degradation patterns.

---

**Next: ReedCMS-11-Plugin-Development.md** - Detailed development guides, SDK documentation, and implementation examples for both LUA and Rust plugins.