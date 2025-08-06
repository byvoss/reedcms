# ReedCMS Memory Management - KISS Minimal Strategy

## Core Philosophy

**Single Redis + Smart Key Protection = Maximum Simplicity**

No complex two-tier setups, no clusters, no over-engineering. Just one Redis with intelligent TTL-based key protection that aligns with KISS and 4k Demo Scene principles.

## Single Redis Architecture

### **Memory Strategy: TTL-Based Key Protection**
```rust
pub struct UCGMemoryManager {
    redis: RedisClient,              // Single Redis for everything
    csv_loader: CsvLoader,           // Recovery from source of truth
}

impl UCGMemoryManager {
    pub async fn store_ucg_structure(&self, key: &str, data: &str) -> Result<(), Error> {
        // UCG structure - NO TTL (protected from eviction)
        self.redis.set(key, data).await?;
        self.redis.persist(key).await?;  // Remove any TTL = never evicted
        
        Ok(())
    }
    
    pub async fn store_cache_data(&self, key: &str, data: &str, ttl: Duration) -> Result<(), Error> {
        // Cache data - WITH TTL (can be evicted)
        self.redis.setex(key, ttl.as_secs() as usize, data).await?;
        
        Ok(())
    }
    
    pub async fn store_search_index(&self, key: &str, data: &[String]) -> Result<(), Error> {
        // Search index - NO TTL but recoverable from content
        self.redis.sadd(key, data).await?;
        self.redis.persist(key).await?;
        
        Ok(())
    }
}
```

### **Redis Key Categories**
```redis
# Protected Keys (NO TTL - never evicted)
HSET entity:snippet:homepage data '{"title":"Home"}'
SADD children:content.1 "content.1.1" "content.1.2"
HSET assoc:content.1.1 parent_id "homepage" weight 0
SADD word:modern "snippet:homepage" "snippet:about"

# Cache Keys (WITH TTL - can be evicted by LRU)
SETEX page:cache:home:de_DE 3600 "<html>...</html>"
SETEX session:user123 1800 '{"cart_id":"cart456"}'
SETEX template:cache:hero 900 "<hero-banner>...</hero-banner>"
```

## Simple Memory Configuration

### **Single Redis Setup**
```toml
# config/redis.toml - Minimal configuration
[redis]
host = "localhost"
port = 6379
database = 0
maxmemory = "512MB"                  # Conservative default
maxmemory_policy = "volatile-lru"    # Only evict keys WITH TTL
save = "900 1 300 10 60 10000"      # Persist UCG structure to disk
```

### **Memory Allocation Strategy**
```rust
// Realistic memory usage for typical sites
pub struct MemoryAllocation {
    // Protected (never evicted)
    ucg_structure: 64_MB,      // 1,000 snippets = ~64MB
    search_index: 16_MB,       // Search words + associations
    
    // Cache (can be evicted)
    page_cache: 32_MB,         // ~50 cached pages
    sessions: 8_MB,            // ~100 concurrent users
    templates: 4_MB,           // Compiled template cache
    
    // Redis overhead
    overhead: 8_MB,            // ~6% overhead
    
    total: 132_MB,             // Fits in 512MB easily
}
```

## Memory Recovery - KISS Style

### **Simple Recovery Strategy**
```rust
impl UCGMemoryManager {
    pub async fn handle_memory_pressure(&self) -> Result<(), Error> {
        let memory_usage = self.redis.memory_usage_percentage().await?;
        
        if memory_usage > 0.9 {  // 90% threshold
            // Step 1: Clear cache data (keep UCG structure)
            self.clear_expendable_cache().await?;
            
            // Step 2: Still full? Rebuild from CSV (nuclear option)
            if self.redis.memory_usage_percentage().await? > 0.9 {
                self.rebuild_from_csv().await?;
            }
        }
        
        Ok(())
    }
    
    async fn clear_expendable_cache(&self) -> Result<(), Error> {
        // Only delete keys WITH TTL (cache data)
        let cache_patterns = vec![
            "page:cache:*",
            "session:*", 
            "template:cache:*"
        ];
        
        for pattern in cache_patterns {
            let keys = self.redis.scan_match(pattern).await?;
            if !keys.is_empty() {
                self.redis.del(&keys).await?;
            }
        }
        
        tracing::info!("Cleared expendable cache - UCG structure preserved");
        Ok(())
    }
    
    async fn rebuild_from_csv(&self) -> Result<(), Error> {
        tracing::warn!("Memory critical - rebuilding UCG from CSV");
        
        // Nuclear option: clear everything and rebuild
        self.redis.flushdb().await?;
        
        // Rebuild UCG structure from CSV (source of truth)
        let associations = self.csv_loader.load_associations().await?;
        let snippets = self.csv_loader.load_snippets().await?;
        
        // Rebuild structure keys (no TTL)
        for assoc in associations {
            self.store_ucg_structure(&format!("assoc:{}", assoc.path), &assoc.to_json()).await?;
        }
        
        for snippet in snippets {
            self.store_ucg_structure(&format!("entity:snippet:{}", snippet.name), &snippet.to_json()).await?;
        }
        
        tracing::info!("UCG rebuilt from CSV - system operational");
        Ok(())
    }
}
```

## Memory Monitoring - Minimal

### **Background Memory Check**
```rust
pub struct MemoryMonitor {
    redis: RedisClient,
    check_interval: Duration,
}

impl MemoryMonitor {
    pub async fn start_monitoring(&self) -> Result<(), Error> {
        let mut interval = tokio::time::interval(self.check_interval);
        
        tokio::spawn(async move {
            loop {
                interval.tick().await;
                self.check_memory_status().await.ok();
            }
        });
        
        Ok(())
    }
    
    async fn check_memory_status(&self) -> Result<(), Error> {
        let usage = self.redis.memory_usage_percentage().await?;
        
        match usage {
            x if x > 0.95 => {
                tracing::error!("Redis memory critical: {:.1}% - initiating recovery", x * 100.0);
                self.handle_memory_pressure().await?;
            },
            x if x > 0.85 => {
                tracing::warn!("Redis memory high: {:.1}% - monitoring", x * 100.0);
            },
            _ => {
                // Normal operation - no logging needed
            }
        }
        
        Ok(())
    }
}
```

## CLI Integration - Simple Commands

### **Memory Management Commands**
```bash
# Simple memory status
reed memory status
Redis Memory: 234MB / 512MB (46%) - OK
UCG Structure: ~180MB (protected)
Cache Data: ~54MB (expendable)

# Clear cache when needed
reed memory clear-cache
✓ Cleared page cache (32MB freed)
✓ Cleared sessions (8MB freed)
✓ UCG structure preserved

# Emergency rebuild from CSV
reed memory rebuild
⚠ This will clear ALL Redis data and rebuild from CSV
Continue? [y/N]: y
✓ Redis cleared
✓ UCG structure rebuilt (1,247 entities)
✓ System operational

# Simple monitoring
reed memory monitor --interval 60s
[14:30:15] Memory: 187MB (37%) - OK
[14:31:15] Memory: 198MB (39%) - OK
[14:32:15] Memory: 445MB (87%) - WARNING
```

## Deployment Configuration

### **Docker Setup - Single Container**
```yaml
# docker-compose.yml - Minimal setup
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: >
      redis-server 
      --maxmemory 512mb 
      --maxmemory-policy volatile-lru
      --save 900 1 300 10 60 10000
    volumes:
      - redis-data:/data

volumes:
  redis-data:
```

### **Memory Scaling - Linear Growth**
```rust
// Simple scaling based on content size
pub fn calculate_redis_memory(snippet_count: usize) -> String {
    let base_mb = match snippet_count {
        0..=1_000 => 256,        // 256MB for small sites
        1_001..=10_000 => 512,   // 512MB for medium sites  
        10_001..=50_000 => 1024, // 1GB for large sites
        _ => 2048,               // 2GB for very large sites
    };
    
    format!("{}MB", base_mb)
}
```

## Real-World Memory Usage

### **Actual Memory Consumption**
```rust
// Based on real data structures, not theoretical overhead
pub struct ActualMemoryUsage {
    // 1,000 snippets (typical small business)
    small_site: MemoryBreakdown {
        ucg_entities: 20_MB,      // 20KB per snippet avg
        ucg_associations: 15_MB,  // Parent-child relationships
        search_index: 8_MB,       // Word associations
        page_cache: 16_MB,        // 25 cached pages
        sessions: 4_MB,           // 50 active users
        total: 63_MB,             // Fits in 256MB easily
    },
    
    // 10,000 snippets (large site)
    large_site: MemoryBreakdown {
        ucg_entities: 200_MB,     
        ucg_associations: 150_MB,
        search_index: 80_MB,
        page_cache: 64_MB,        // 100 cached pages
        sessions: 20_MB,          // 250 active users
        total: 514_MB,            // Fits in 1GB with room to spare
    },
}
```

### **Memory Efficiency Benefits**
```rust
// Why this approach is minimal yet robust
pub struct EfficiencyBenefits {
    // Single Redis instance
    deployment_complexity: "Minimal - one container",
    configuration_files: 1,        // Just redis.toml
    monitoring_overhead: "~1MB",   // Simple memory checker
    
    // TTL-based protection  
    eviction_safety: "Automatic", // UCG keys never evicted
    cache_management: "Automatic", // LRU handles cache cleanup
    recovery_complexity: "Simple", // Just rebuild from CSV
    
    // Performance characteristics
    query_performance: "<1ms",     // Single Redis lookup
    memory_overhead: "<10%",       // No cluster communication
    scaling_path: "Linear",        // Add more memory when needed
}
```

## Error Handling - Graceful Degradation

### **Memory Exhaustion Scenarios**
```rust
impl UCGMemoryManager {
    pub async fn handle_redis_oom(&self) -> Result<(), Error> {
        match self.diagnose_memory_issue().await? {
            MemoryIssue::TooMuchCache => {
                // Clear cache, keep structure
                self.clear_expendable_cache().await?;
                Ok(())
            },
            MemoryIssue::TooMuchStructure => {
                // This shouldn't happen with realistic content
                tracing::error!("UCG structure too large for configured memory");
                Err(Error::InsufficientMemory)
            },
            MemoryIssue::RedisDown => {
                // Fallback to PostgreSQL direct queries (slow but functional)
                self.enable_postgres_fallback().await?;
                Ok(())
            }
        }
    }
}
```

**Result:** Minimal, KISS-compliant memory management - single Redis with intelligent TTL-based protection, simple recovery from CSV source of truth, linear scaling, and bulletproof simplicity that any developer can understand and deploy.
