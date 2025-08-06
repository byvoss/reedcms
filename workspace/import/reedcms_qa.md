# ReedCMS Technical Q&A

## Critical Architecture Questions

### Q1: Redis Persistence Strategy (RESOLVED)

**Question:** Redis als Cache oder Source of Truth für UCG? Was passiert bei Redis-Restart - rebuild from PostgreSQL UCG?

**Answer:** Dafür haben wir ja die CSV und das UCG PG backup.

**Technical Implementation:**
Redis ist **pure Performance Cache** - niemals persistence. Das UCG-System hat bereits perfekte Backup-Strategien:

```rust
// Redis Recovery Strategy
pub async fn redis_startup_recovery() -> Result<(), Error> {
    // 1. CSV Layer: Immer aktuell, Git-versionable
    let ucg_structure = load_ucg_from_csv().await?;
    
    // 2. PostgreSQL UCG: Compressed backup für consistency
    let ucg_backup = postgres_ucg.load_structure_backup().await?;
    
    // 3. Warm Redis cache from authoritative sources
    redis.populate_ucg_cache(&ucg_structure).await?;
    redis.rebuild_search_index_from_postgres().await?;
}
```

**4-Layer Responsibility:**
- **CSV:** Schema definitions (Source of Truth)
- **PostgreSQL UCG:** Structure backup (compressed, recovery)  
- **PostgreSQL Main:** Live content (ACID transactions)
- **Redis:** Performance only (0.1ms queries, rebuildable)

**Result:** Redis crash = no data loss. System recovers from CSV + PostgreSQL UCG backup in seconds.

---

### Q2: Plugin Conflict Resolution (RESOLVED)

**Question:** Zwei Plugins installieren identische Snippet-Namen (z.B. "seo-widget"). Wer gewinnt? Wie wird Snippet ownership verwaltet? Gibt es Versionierung oder Error handling?

**Answer:** Es gibt viele mögliche Konfliktstellen, die unterschiedlicher Lösungen bedürfen:

**A) Plugin vs Plugin Konflikte:**
- Inkompatibilitäten werden bereits vor Auswahl im Plugin Shop angezeigt
- Plugin-Maintainer müssen sorgfältig testen und in Metafile klar eintragen
- Orientierung am FreeBSD pkg System (robust dependency management)
- Partner-Zertifizierung: KI-gestützte Tests auf Herz und Nieren
- Beta-Modus für instabile Plugins → nur im Developer Store Mode sichtbar

**B) Plugin vs ReedCMS Kern-Bibliotheken:**
- Klare pkg-style dependency declarations welche libs in welcher Version
- Warnsystem bei entstehenden Inkompatibilitäten (wie Craft CMS)
- Automatischer Beta-Status bei ReedCMS Core Updates
- Plugin nur für Entwickler sichtbar bis Kompatibilität wiederhergestellt

**C) Plugin vs Server Environment:**
- FreeBSD pkg-Pattern für System-Dependencies
- Beta-Pattern bei Server-Library Updates
- Entwickler-only Visibility bis Environment-Kompatibilität geklärt

**Technical Implementation:**
```toml
# plugin.toml - FreeBSD pkg-style dependencies
[dependencies]
reedcms_core = ">=1.0.0,<2.0.0"
openssl = ">=1.1.0"
conflicts_with = ["other-seo-plugin", "legacy-analytics"]

[certification]
partner_certified = true
test_coverage = 95
beta_status = false
```

**Additional Clarifications:**
- **Beta-Status ist umgebungsabhängig:** Plugin kann auf ReedCMS 1.5 stabil sein, aber auf 2.0 Beta-Status haben
- **Security Integration:** Extra Reiter für Security-Infos im Plugin View mit bekannten Vulnerabilities aus CVE-Datenbanken

**Technical Implementation Extended:**
```toml
# plugin.toml - Environment-specific compatibility
[compatibility]
reedcms_1_5 = "stable"
reedcms_2_0 = "beta"    # Only this version gets beta treatment
openssl_1_1 = "stable"
openssl_3_0 = "incompatible"

[security]
cve_database_check = true
last_security_scan = "2025-01-15"
known_vulnerabilities = []
security_grade = "A"
```

**Plugin Store UI:**
```
Plugin: AI-Core v1.2.0
├── Compatibility: ✅ Stable (ReedCMS 1.5) | ⚠️ Beta (ReedCMS 2.0)  
├── Dependencies: openssl >=1.1.0, postgres >=12
├── Security: 🛡️ Grade A | Last Scan: 2025-01-15 | 0 CVEs
└── Partner Certified: ✅ KI-tested & approved
```

**Plugin Runtime Behavior:**
- **Installierte Plugins bleiben aktiv** auch bei Beta-Degradation
- **Keine automatische Deaktivierung** bei Kompatibilitätsproblemen
- **Admin Panel Security Section** zeigt Statusmeldungen mit Details
- **Dashboard Warning Entries** verlinken auf Security Section

**Admin UI Flow:**
```
Dashboard:
⚠️ Plugin "ai-core" degraded to beta on ReedCMS 2.0 → [Details]

Security Section:
Plugin: AI-Core v1.2.0
Status: ⚠️ Beta (compatibility issue with ReedCMS 2.0)  
Action: Update available v1.3.0 (restores stable status)
CVE Status: 🛡️ No known vulnerabilities
```

**Result:** Professioneller Package Manager (FreeBSD pkg-style) mit non-disruptive Security & Compatibility Monitoring. Plugins bleiben funktional, Admins werden informiert.

---

### Q3: Asset Pipeline Implementation (RESOLVED)

**Question:** Wie wird das Rust-basierte Asset Bundling implementiert? Eigene Pipeline oder externes Tool? Wie funktioniert Tree-shaking und Bundle-Hash-Generation konkret?

**Answer:** Default eigene 200-Zeilen Lösung ganz simpel, aber auch default mit dabei und per Flag aktivierbar: eines der anderen (was besser performt und KISSer ist). Nur eines von den beiden Tools - das was wirklich more KISS and simple ist. Merke: Nix mit Node.js oder anderen superkomplexen Frameworks! Optimal: Single file, KISS and stable. Das gilt für alle externen Tools!

**Technical Implementation:**
```rust
// Default: Simple Rust-native pipeline
pub struct AssetPipeline {
    mode: AssetMode,
}

pub enum AssetMode {
    Native,    // Default: 200-line simple bundler
    ESBuild,   // Flag: Single Go binary, no Node.js complexity
}

impl AssetPipeline {
    pub async fn build_bundle(&self) -> Result<Bundle, Error> {
        match self.mode {
            AssetMode::Native => self.build_native().await,
            AssetMode::ESBuild => self.build_esbuild().await, // Single binary call
        }
    }
    
    async fn build_native(&self) -> Result<Bundle, Error> {
        // Stupid simple: concatenate + hash
        let css = self.concatenate_css_files().await?;
        let js = self.concatenate_js_modules().await?;
        let hash = self.calculate_content_hash(&css, &js);
        
        self.write_bundle(&css, &js, &hash).await?;
        Ok(Bundle { hash })
    }
    
    async fn build_esbuild(&self) -> Result<Bundle, Error> {
        // Single binary execution, no Node.js ecosystem
        let output = tokio::process::Command::new("esbuild")
            .args(&["--bundle", "--minify", "--outdir=build/assets/"])
            .output()
            .await?;
        
        // Parse output, generate hash
        self.process_esbuild_output(output).await
    }
}
```

**External Tool Requirements:**
- ✅ ESBuild: Single Go binary, no dependencies
- ❌ Vite: Node.js ecosystem, complex framework
- ❌ Webpack: Node.js nightmare complexity
- ❌ Rollup: Node.js, plugin ecosystem

**Result:** Native 200-line bundler + optional ESBuild (single binary only). Zero Node.js complexity.

**Result:** KISS default mit 200-Zeilen Rust-native, aber battle-tested externes Tool per Flag für Performance-kritische Projekte.

---

### Q4: i18n Locale Detection (RESOLVED)

**Question:** Wie wird User Locale erkannt und aufgelöst? Browser-Header? URL-Parameter? Session? Fallback-Chain bei unbekannten Locales?

**Answer:** Beim Setup des CMS fragen, welche Sprache default ist und das festlegen. Dann das System des Benutzers auswerten und schauen ob es dazu eine hinterlegte Übersetzung gibt: wenn nicht default, wenn der zwar gesetzt ist aber für den String keine Entsprechung eingepflegt hat, dann English. Da wo man die default lang setzen kann gibt es wieder so ein [+] neue Sprache Widget was aufgebaut ist wie ein Photoshop Layer Widget. Über dieser Tabelle gibt es einen Switch für die drei Methoden, was dann die Felder global regelt. Sowas wie durcheinander ist ja sowas von un-KISS.

**UI Setup Widget:**
```
Languages Configuration

Global Method: ○ Header    ● Path    ○ Subdomain

Default Language: 🇩🇪 German (de_DE)

Additional Languages:
┌─────────────────────────────────────────────────────────────┐
│ [+] Add Language                                            │
├─────────────────┬─────────────────────────────────────────┤
│ Language        │ Identifier                              │
├─────────────────┼─────────────────────────────────────────┤
│ 🇺🇸 English     │ /en/ (auto-generated from method)      │
│ 🇫🇷 French      │ /fr/                                    │  
│ 🇪🇸 Spanish     │ /es/                                    │
│ 🇮🇹 Italian     │ /it/                                    │
└─────────────────┴─────────────────────────────────────────┘
```

**Global Method Examples:**
- **Header:** Identifier column shows "(automatic)" for all
- **Path:** Identifier shows "/en/", "/fr/", "/es/" etc.
- **Subdomain:** Identifier shows "en.site.com", "fr.site.com" etc.

**Technical Implementation:**
```rust
#[derive(Debug, Clone)]
pub struct LocaleConfig {
    pub global_method: LocaleMethod,
    pub default_locale: String,
    pub additional_locales: Vec<String>,
}

pub enum LocaleMethod {
    Header,           // All languages via browser header
    Path,            // All languages via /lang/ prefix  
    Subdomain,       // All languages via lang.domain.com
}
```

**Result:** Ein globaler Switch für ALLE Sprachen - konsistent und KISS. Keine Method-Mischung möglich. Was man dort angelegt hat wird eben auch in den CSV Files ausgewertet - hat man z.B. kein Baskisch angelegt als Sprache, kann man es auch nicht in der translation.csv des Snippets oder als Sprachversion im Admin Panel für einen Content auswählen, da es natürlich auch keine Spalte im Content PG dafür gibt. Wollen wir die translate.csv besser in translate.de_DE.csv umbenennen und je angelegter Sprache nach einer suchen (man muss sie ja nicht anlegen, aber eine Trennung erleichtert das Lesen für den Entwickler auf Code-Ebene).

**Language-Specific Translation Files:**
```
config/
├── translations.de_DE.csv      # German translations
├── translations.en_US.csv      # English translations  
├── translations.fr_FR.csv      # French translations
├── translations.es_ES.csv      # Spanish translations (optional)
└── translations.eu.csv         # Basque translations (ISO 639-1: eu) 🏔️
```

**ISO Standard Clarification:**
Basque hat den einheitlichen ISO 639-1 Code "eu" ohne Country-Suffix. Es gibt **keine** separaten `eu_ES` oder `eu_FR` Codes - Euskera ist eine **einheitliche Sprache** unabhängig von den Landesgrenzen.

**Regional Handling:**
- **Euskera Batua:** Standardisierte Form entwickelt in den späten 1960ern
- **Dialekte:** Biscayan, Gipuzkoan, Upper Navarrese (Spanien) und Navarrese–Lapurdian, Souletin (Frankreich)
- **Praktisch:** Eine `translations.eu.csv` für alle Basken, Dialekt-Unterschiede sind minimal

**Technical Implementation:**
```rust
// Basque gets unified treatment
pub fn get_basque_locale() -> &'static str {
    "eu"  // No country suffix needed
}
```

**CSV File Structure:**
```csv
# translations.eu_ES.csv (Basque - weil wir die Basken lieben!)
snippet_name,field_name,label
hero-banner,title,"Izenburua"
hero-banner,subtitle,"Azpititulua"
product-card,name,"Produktuaren izena"
product-card,price,"Prezioa"

# translations.de_DE.csv
snippet_name,field_name,label
hero-banner,title,"Titel"
hero-banner,subtitle,"Untertitel"
product-card,name,"Produktname"
product-card,price,"Preis"
```

**Technical Implementation:**
```rust
pub struct TranslationLoader {
    configured_locales: Vec<String>,
}

impl TranslationLoader {
    pub async fn load_all_translations(&self) -> Result<TranslationMap, Error> {
        let mut translations = TranslationMap::new();
        
        for locale in &self.configured_locales {
            let file_path = format!("config/translations.{}.csv", locale);
            
            // Optional - file doesn't need to exist
            if Path::new(&file_path).exists() {
                let locale_translations = self.load_translation_file(&file_path).await?;
                translations.insert(locale.clone(), locale_translations);
            }
        }
        
        Ok(translations)
    }
}
```

**Developer Benefits:**
- ✅ **File per language** - easier to read/edit
- ✅ **Optional files** - don't need all languages
- ✅ **Clear separation** - no mixed-language CSV confusion
- ✅ **Git-friendly** - separate diffs per language

---

## Additional Technical Questions

### Q5: Database Migration System (RESOLVED)

**Question:** Wie funktioniert Schema Evolution zwischen ReedCMS Versionen? Automatische Migrations? Manual intervention required?

**Answer:** Da fragst du was... dump und im neuen reinorgeln?

**KISS Migration Strategy:**
```bash
# ReedCMS Update Process
reed backup create full-backup-v1.5.sql    # Dump everything
reed update --version 2.0                  # Update ReedCMS binary
reed restore from full-backup-v1.5.sql     # Import into new schema
```

**Why this works for ReedCMS:**
- **CSV Schema:** Always current (Git-tracked)
- **PostgreSQL UCG:** Backup/recovery optimized 
- **Redis:** Rebuildable from authoritative sources
- **4-Layer Architecture:** Clean separation allows full restore

**Technical Implementation:**
```rust
pub async fn migration_strategy() -> Result<(), Error> {
    // 1. Export all user data
    let content_dump = postgres_main.export_all_content().await?;
    let ucg_dump = postgres_ucg.export_structure().await?;
    
    // 2. Update ReedCMS (new binary + CSV schemas)
    update_reedcms_binary().await?;
    
    // 3. Initialize new schema from CSV
    initialize_from_csv().await?;
    
    // 4. Import user data into new structure
    import_content_with_schema_mapping(content_dump).await?;
    import_ucg_structure(ucg_dump).await?;
    
    // 5. Rebuild Redis cache
    warm_redis_cache().await?;
}
```

**Result:** KISS "dump and restore" strategy leveraging 4-layer architecture - no complex incremental migrations needed.

---

### Q6: Socket File Management (RESOLVED)

**Question:** Was passiert bei System-Restart mit Socket Files in /tmp/reed-cms/? Cleanup strategy? Permission handling zwischen Prozessen?

**Answer:** Wie handhabt das ein nginx?

**Nginx Socket Management (Best Practices):**
- **Stale Socket Problem:** nginx leaves old sockets after restart/crash
- **Permissions:** 0660 für user/group access, 0666 für world access
- **Cleanup:** atexit() handlers + signal handlers für proper cleanup
- **Private Directory:** Sockets in dedicated directory statt /tmp

**ReedCMS Socket Strategy (nginx-inspired):**
```rust
pub struct SocketManager {
    socket_dir: PathBuf,  // /var/run/reed-cms/ statt /tmp/
    cleanup_on_start: bool,
}

impl SocketManager {
    pub async fn initialize() -> Result<(), Error> {
        let socket_dir = Path::new("/var/run/reed-cms");
        
        // 1. Create private socket directory
        fs::create_dir_all(&socket_dir).await?;
        fs::set_permissions(&socket_dir, Permissions::from_mode(0o750)).await?;
        
        // 2. Cleanup stale sockets on startup
        self.cleanup_stale_sockets().await?;
        
        // 3. Register cleanup handlers
        self.register_cleanup_handlers()?;
        
        Ok(())
    }
    
    async fn cleanup_stale_sockets(&self) -> Result<(), Error> {
        // Check /proc/net/unix for active sockets
        let active_sockets = self.get_active_unix_sockets().await?;
        
        // Remove files that aren't in /proc/net/unix
        for socket_file in fs::read_dir(&self.socket_dir).await? {
            if !active_sockets.contains(&socket_file.path()) {
                fs::remove_file(socket_file.path()).await?;
            }
        }
        
        Ok(())
    }
    
    fn register_cleanup_handlers(&self) -> Result<(), Error> {
        // atexit() for normal shutdown
        unsafe { libc::atexit(cleanup_all_sockets) };
        
        // Signal handlers for SIGTERM, SIGINT
        signal_hook::flag::register(SIGTERM, Arc::clone(&SHUTDOWN_FLAG))?;
        signal_hook::flag::register(SIGINT, Arc::clone(&SHUTDOWN_FLAG))?;
        
        Ok(())
    }
}
```

**Socket Permissions:**
```bash
# ReedCMS socket directory
/var/run/reed-cms/           # 0750 reed:reed
├── core.sock               # 0660 reed:reed  
└── plugins/
    ├── ai-core.sock        # 0660 reed:reed
    └── simple-seo.sock     # 0660 reed:reed
```

**Result:** Nginx-style Socket Management - private directory, stale cleanup, proper permissions, signal handlers für graceful shutdown.

---

### Q7: Memory Management & Scaling (RESOLVED)

**Question:** Bei "100K snippets → 1.3GB" - was passiert bei Redis Memory Limits? LRU eviction policy? Scaling strategy für größere Datenmengen?

**Answer:** Da habe ich nicht viel Ahnung. Wie macht denn Craft sowas?

**Craft CMS Memory Management Lessons:**
- **PHP Memory:** Standard 256MB RAM für PHP, bei großen Sites bis 512MB
- **Performance Issues:** 400,000+ Users = PHP memory issues auch bei 128M
- **Query Optimization:** Über 50 DB queries = Problem, eager loading nutzen
- **Load Balancing:** Redis für Sessions, Read/Write DB splitting
- **Scaling:** Horizontal scaling mit Load Balancer + CDN für Assets

**ReedCMS Memory Strategy (Craft-inspired):**
```rust
pub struct MemoryManager {
    redis_max_memory: String,        // "1gb", "2gb"
    redis_policy: EvictionPolicy,    // allkeys-lru
    connection_pools: ConnectionPools,
}

pub enum EvictionPolicy {
    AllKeysLRU,      // Evict least recently used keys
    VolatileLRU,     // Only evict keys with TTL
    NoEviction,      // Return errors when memory full
}

impl MemoryManager {
    pub async fn configure_redis_memory(&self) -> Result<(), Error> {
        // Configure Redis memory limits
        redis.config_set("maxmemory", &self.redis_max_memory).await?;
        redis.config_set("maxmemory-policy", "allkeys-lru").await?;
        
        // Monitor memory usage
        self.setup_memory_monitoring().await?;
    }
    
    pub async fn handle_memory_pressure(&self) -> Result<(), Error> {
        let memory_usage = redis.memory_usage().await?;
        
        if memory_usage > 0.9 { // 90% threshold
            // 1. Clear old search indexes
            self.cleanup_old_search_indexes().await?;
            
            // 2. Remove stale session data
            self.cleanup_expired_sessions().await?;
            
            // 3. Alert admin
            self.send_memory_alert(memory_usage).await?;
        }
        
        Ok(())
    }
}
```

**Scaling Strategy:**
```rust
// Craft-style horizontal scaling
pub struct ScalingConfig {
    pub redis_cluster: bool,           // Multiple Redis instances
    pub postgres_read_replicas: Vec<String>, // Read/write splitting
    pub cdn_assets: bool,              // S3 + CloudFront for assets
    pub load_balancer: bool,           // Multiple ReedCMS instances
}
```

**Memory Thresholds:**
- **Small:** <10K snippets = 128MB Redis
- **Medium:** 10K-50K snippets = 512MB Redis  
- **Large:** 50K+ snippets = 1GB+ Redis + clustering
- **Alert:** 90% memory usage triggers cleanup + admin notification

**Result:** Craft-inspirierte Memory Management - LRU eviction, monitoring, horizontal scaling bei Bedarf.

---

### Q8: PostgreSQL Connection Management (RESOLVED)

**Question:** Connection pool size? Timeout handling? Transaction management für UCG operations? Read replica strategy?

**Answer:** Was ist denn battle-tested?

**Battle-Tested PostgreSQL Connection Management:**

**SQLx + Connection Pooling (Rust Standard):**
- SQLx Pool mit max_connections(5-20) ist production standard
- Built-in connection pooling mit sqlx::Pool
- Compile-time checked queries für Sicherheit
- Async/await + tokio runtime optimiert

**PgBouncer (External Connection Pooler):**
- Battle-tested für Enterprise environments
- Transaction pooling mode für bessere Performance
- **Problem:** Prepared statements "already exists" errors mit SQLx
- **Solution:** Statement cache capacity = 0 als Workaround

**ReedCMS Strategy (Best of Both):**
```rust
pub struct DatabaseManager {
    postgres_main_pool: PgPool,
    postgres_ucg_pool: PgPool,
    connection_config: ConnectionConfig,
}

#[derive(Debug, Clone)]
pub struct ConnectionConfig {
    pub max_connections: u32,      // 10-20 für kleine Sites
    pub min_connections: u32,      // 2-5 minimum
    pub connection_timeout: Duration, // 30 seconds
    pub idle_timeout: Duration,    // 10 minutes
    pub statement_cache_capacity: usize, // 100 queries
}

impl DatabaseManager {
    pub async fn new(database_urls: DatabaseUrls) -> Result<Self, Error> {
        let config = ConnectionConfig::default();
        
        // Main content database pool
        let postgres_main_pool = PgPoolOptions::new()
            .max_connections(config.max_connections)
            .min_connections(config.min_connections) 
            .acquire_timeout(config.connection_timeout)
            .idle_timeout(config.idle_timeout)
            .connect(&database_urls.main)
            .await?;
            
        // UCG backup database pool (lighter usage)
        let postgres_ucg_pool = PgPoolOptions::new()
            .max_connections(config.max_connections / 2) // Less connections needed
            .connect(&database_urls.ucg)
            .await?;
            
        Ok(DatabaseManager {
            postgres_main_pool,
            postgres_ucg_pool,
            connection_config: config,
        })
    }
    
    pub async fn execute_transaction<F, R>(&self, f: F) -> Result<R, Error> 
    where
        F: FnOnce(&mut Transaction<'_, Postgres>) -> BoxFuture<'_, Result<R, Error>>,
    {
        let mut tx = self.postgres_main_pool.begin().await?;
        let result = f(&mut tx).await?;
        tx.commit().await?;
        Ok(result)
    }
}
```

**Production Settings:**
```rust
impl Default for ConnectionConfig {
    fn default() -> Self {
        ConnectionConfig {
            max_connections: 20,           // Battle-tested default
            min_connections: 5,
            connection_timeout: Duration::from_secs(30),
            idle_timeout: Duration::from_secs(600), // 10 minutes
            statement_cache_capacity: 100,  // SQLx default
        }
    }
}
```

**Read Replica Support:**
```rust
pub struct DatabaseUrls {
    pub main: String,              // Write operations
    pub ucg: String,               // UCG backup operations  
    pub read_replica: Option<String>, // Optional read scaling
}
```

**Result:** SQLx Connection Pooling als Rust-native Standard, optional PgBouncer für Enterprise scaling, battle-tested defaults aus SQLx community.

---

## Core Philosophy Questions

### Q-1: External Tool Policy (RESOLVED)

**Question:** Welche externen Tools sind für ReedCMS akzeptabel? Node.js ecosystem? Complex frameworks?

**Answer:** Nix mit Node.js oder anderen superkomplexen Frameworks! Optimal: Single file, KISS and stable. Das gilt für alle externen Tools!

**ReedCMS External Tool Requirements:**
- ✅ **Single binary** (Go, Rust, C executables)
- ✅ **Zero dependencies** - no ecosystem mess
- ✅ **Stable, battle-tested** - proven in production
- ❌ **Node.js ecosystems** - npm/yarn complexity
- ❌ **Complex frameworks** - plugin dependencies  
- ❌ **Breaking change prone** - unstable APIs

**Examples:**
```
✅ ESBuild: Single Go binary, no dependencies
✅ PostgreSQL: Single process, stable API
✅ Redis: Single binary, battle-tested
❌ Vite: Node.js ecosystem complexity
❌ Webpack: npm dependency nightmare
❌ Any Node.js-based tool with plugins
```

**Result:** ReedCMS bleibt deployment-friendly durch strikte Single-Binary Policy für alle externen Dependencies.

---

## Resolved Questions

### Q0: Template Scoping System (RESOLVED)

**Question:** Wie funktioniert Context Scoping wenn Templates nicht überschreibbar sind? Nur CSS/JS Override?

**Answer:** Du erstellst ein Base Theme bei Installation mit eindeutigem Name-Key. Das erzeugt automatisch unter themes/ die notwendigen Files zum Überschreiben der Snippet-Standards. Für Scopes legst du entsprechende Files in die Pfadstruktur an. Selbst Tera Files kannst du ändern, indem du das parent Tera als Parent im Head reinlädst und neu schreibst. Im UI/CLI wird jedem Scope der Theme Entry fest zugewiesen - so kann der Redakteur ohne Umwege auf ein Weihnachtstheme umschalten.

**Technical Implementation:**
- Web Components bekommen `scope` Attribute direkt: `<hero-banner scope="{{ scope }}">`  
- CSS targeting via `hero-banner[scope*="berlin"]`
- Theme Assignment via CLI: `reed theme assign scope "berlin" theme "corporate-berlin"`
- Tera Template Inheritance: `{% extends "snippets/hero-banner/hero-banner.tera" %}`
- Ein-Klick Theme Switching für Content-Teams ohne Developer

**Result:** Craft CMS-like UX für Redakteure mit Rust-Performance für Developers.

---

## Implementation Priority

1. ✅ Template Scoping System (RESOLVED)
2. ✅ Redis Persistence Strategy (RESOLVED)
3. ✅ Plugin Conflict Resolution (RESOLVED)
4. ✅ Asset Pipeline Implementation (RESOLVED)
5. ✅ i18n Locale Detection (RESOLVED)
6. ✅ Database Migration System (RESOLVED)
7. ⏳ Socket File Management
8. ⏳ Memory Management & Scaling
9. ⏳ PostgreSQL Connection Management
