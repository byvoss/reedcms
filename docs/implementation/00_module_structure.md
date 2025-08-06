# ReedCMS Implementation Module Structure

## Didaktische Reihenfolge der Implementierung

Diese Struktur baut logisch aufeinander auf. Jedes Modul hat max. 500 Zeilen und eine klare Verantwortung.

### Phase 1: Core Foundation (Basis-System)

#### 1.1 Type Definitions & Core Structs
- `01_core_types.md` - Basis-Datentypen (UUID, Entity, Association)
- `02_error_types.md` - Error Handling & Result Types
- `03_config_types.md` - Configuration Structs

#### 1.2 Database Layer
- `04_database_connection.md` - SQLx Connection Pool Setup
- `05_database_schema.md` - PostgreSQL Schema Definition
- `06_database_migrations.md` - Migration System

#### 1.3 Redis Layer
- `07_redis_connection.md` - Redis Client Setup
- `08_redis_keys.md` - Key Naming Convention
- `09_memory_manager.md` - TTL-based Memory Protection

### Phase 2: UCG (Universal Content Graph)

#### 2.1 UCG Core
- `10_ucg_entity.md` - Entity Management
- `11_ucg_association.md` - Association Handling
- `12_ucg_path.md` - Path Resolution (content.1.1)

#### 2.2 UCG Operations
- `13_ucg_query.md` - Query Operations
- `14_ucg_builder.md` - Graph Building
- `15_ucg_recovery.md` - CSV Recovery System

### Phase 3: Registry System

#### 3.1 Snippet Registry
- `16_registry_core.md` - Registry Structure
- `17_registry_loader.md` - CSV Loading
- `18_registry_validation.md` - Field Validation

#### 3.2 Meta-Snippets
- `19_meta_snippet.md` - Meta-Snippet Definition
- `20_meta_validation.md` - Validation Rules

### Phase 4: Template System

#### 4.1 Tera Integration
- `21_tera_setup.md` - Tera Engine Setup
- `22_tera_functions.md` - Custom Functions (string, render_snippet)
- `23_tera_context.md` - Context Building

#### 4.2 EPC (Explicit Path Chain)
- `24_epc_resolver.md` - File Resolution Logic
- `25_epc_theme_chain.md` - Theme Chain Building

### Phase 5: Content Management

#### 5.1 Snippet System
- `26_snippet_core.md` - Snippet CRUD Operations
- `27_snippet_storage.md` - PostgreSQL Storage
- `28_snippet_cache.md` - Redis Caching

#### 5.2 Translation System
- `29_translation_loader.md` - Multi-Layer Loading
- `30_translation_resolver.md` - Resolution Logic

### Phase 6: CLI Implementation

#### 6.1 CLI Core
- `31_cli_parser.md` - FreeBSD-style Parser
- `32_cli_commands.md` - Command Registry
- `33_cli_output.md` - Output Formatting

#### 6.2 CLI Commands
- `34_cmd_snippet.md` - Snippet Commands
- `35_cmd_theme.md` - Theme Commands
- `36_cmd_schema.md` - Schema Commands
- `37_cmd_whois.md` - Whois Debugging

### Phase 7: Web Server

#### 7.1 HTTP Server
- `38_server_setup.md` - Axum Server Setup
- `39_server_routes.md` - Route Definitions
- `40_server_middleware.md` - Security Middleware

#### 7.2 Request Handling
- `41_request_context.md` - Request Context Building
- `42_response_cache.md` - Response Caching

### Phase 8: Asset Pipeline

#### 8.1 Asset Building
- `43_asset_bundler.md` - Simple 200-Line Bundler
- `44_asset_hash.md` - Content Hashing
- `45_asset_serving.md` - Static File Serving

### Phase 9: Plugin System

#### 9.1 Plugin Core
- `46_plugin_manager.md` - Plugin Lifecycle
- `47_plugin_socket.md` - Unix Socket Communication
- `48_plugin_protocol.md` - Binary Protocol

#### 9.2 Plugin Types
- `49_plugin_rust.md` - Rust Pro Plugin Handler
- `50_plugin_lua.md` - LUA Simple Plugin Handler

### Phase 10: Search System

#### 10.1 Search Index
- `51_search_indexer.md` - Word Indexing
- `52_search_query.md` - Query Execution
- `53_search_ranking.md` - Result Ranking

### Phase 11: Security

#### 11.1 Authentication
- `54_auth_session.md` - Session Management
- `55_auth_csrf.md` - CSRF Protection

#### 11.2 Content Firewall
- `56_firewall_rules.md` - Rule Engine
- `57_firewall_builtin.md` - Built-in Rules
- `58_firewall_cli.md` - CLI Management

### Phase 12: Performance

#### 12.1 Monitoring
- `59_performance_metrics.md` - Metric Collection
- `60_performance_benchmark.md` - Benchmarking

#### 12.2 Speed Badge
- `61_speed_badge.md` - Badge System
- `62_speed_ranking.md` - Community Ranking

### Phase 13: Advanced Features

#### 13.1 Feature Toggles
- `63_feature_scopes.md` - Scope-based Toggles
- `64_feature_routing.md` - Content Routing

#### 13.2 Hot Reload
- `65_hot_reload.md` - File Watcher
- `66_hot_websocket.md` - WebSocket Updates

### Phase 14: Testing & DevOps

#### 14.1 Testing
- `67_test_framework.md` - Test Setup
- `68_test_integration.md` - Integration Tests

#### 14.2 Deployment
- `69_deployment_docker.md` - Docker Configuration
- `70_deployment_binary.md` - Single Binary Build

## Implementierungs-Prinzipien

1. **Bottom-Up**: Erst Foundation, dann Features
2. **KISS**: Jedes Modul hat EINE klare Aufgabe
3. **Testbar**: Jedes Modul kann isoliert getestet werden
4. **Dokumentiert**: Jedes Modul erklärt sein "Warum"

## Nächster Schritt

Beginne mit Phase 1.1 - Core Type Definitions als solides Fundament.