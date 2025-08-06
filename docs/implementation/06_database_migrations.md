# Database Migration System

## Overview

KISS Migration System - einfaches Backup/Restore statt komplexer inkrementeller Migrationen.

## Migration Strategy

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// Migration manager for ReedCMS upgrades
pub struct MigrationManager {
    db_manager: Arc<DatabaseManager>,
    backup_path: PathBuf,
}

impl MigrationManager {
    pub fn new(db_manager: Arc<DatabaseManager>, backup_path: PathBuf) -> Self {
        Self {
            db_manager,
            backup_path,
        }
    }
    
    /// Full backup before migration
    pub async fn create_backup(&self, version: &str) -> Result<BackupInfo> {
        let timestamp = Utc::now().format("%Y%m%d_%H%M%S");
        let backup_name = format!("reed_backup_v{}_{}.sql", version, timestamp);
        let backup_file = self.backup_path.join(&backup_name);
        
        tracing::info!("Creating backup: {}", backup_name);
        
        // Export main database
        self.export_database(&backup_file).await?;
        
        // Create backup metadata
        let info = BackupInfo {
            version: version.to_string(),
            created_at: Utc::now(),
            file_path: backup_file,
            size_bytes: std::fs::metadata(&backup_file)?.len(),
            table_count: self.count_tables().await?,
        };
        
        // Save backup info
        self.save_backup_info(&info).await?;
        
        tracing::info!("Backup completed: {} MB", info.size_bytes / 1_048_576);
        Ok(info)
    }
    
    /// Export database using pg_dump
    async fn export_database(&self, output_path: &Path) -> Result<()> {
        let db_url = &self.db_manager.config.main_url;
        
        let output = tokio::process::Command::new("pg_dump")
            .arg(db_url)
            .arg("--clean")
            .arg("--if-exists")
            .arg("--no-owner")
            .arg("--no-privileges")
            .arg("-f")
            .arg(output_path)
            .output()
            .await?;
        
        if !output.status.success() {
            let error = String::from_utf8_lossy(&output.stderr);
            return Err(ReedError::Internal(format!("pg_dump failed: {}", error)));
        }
        
        Ok(())
    }
}
```

## Backup and Restore

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BackupInfo {
    pub version: String,
    pub created_at: DateTime<Utc>,
    pub file_path: PathBuf,
    pub size_bytes: u64,
    pub table_count: usize,
}

impl MigrationManager {
    /// Restore from backup
    pub async fn restore_backup(&self, backup_info: &BackupInfo) -> Result<()> {
        tracing::warn!("Starting database restore from: {:?}", backup_info.file_path);
        
        // Verify backup file exists
        if !backup_info.file_path.exists() {
            return Err(ReedError::Configuration(
                "Backup file not found".to_string()
            ));
        }
        
        // Close all connections
        self.db_manager.shutdown().await?;
        
        // Restore using psql
        let output = tokio::process::Command::new("psql")
            .arg(&self.db_manager.config.main_url)
            .arg("-f")
            .arg(&backup_info.file_path)
            .output()
            .await?;
        
        if !output.status.success() {
            let error = String::from_utf8_lossy(&output.stderr);
            return Err(ReedError::Internal(format!("psql restore failed: {}", error)));
        }
        
        tracing::info!("Database restored successfully");
        Ok(())
    }
    
    /// List available backups
    pub async fn list_backups(&self) -> Result<Vec<BackupInfo>> {
        let mut backups = Vec::new();
        
        let mut entries = tokio::fs::read_dir(&self.backup_path).await?;
        while let Some(entry) = entries.next_entry().await? {
            if let Some(name) = entry.file_name().to_str() {
                if name.starts_with("reed_backup_") && name.ends_with(".sql") {
                    if let Ok(info) = self.load_backup_info(&entry.path()).await {
                        backups.push(info);
                    }
                }
            }
        }
        
        // Sort by date, newest first
        backups.sort_by(|a, b| b.created_at.cmp(&a.created_at));
        Ok(backups)
    }
}
```

## Schema Version Tracking

```rust
/// Simple version tracking
pub struct SchemaVersion {
    pub version: u32,
    pub name: String,
    pub applied_at: DateTime<Utc>,
}

impl MigrationManager {
    /// Get current schema version
    pub async fn current_version(&self) -> Result<u32> {
        let version: Option<(i32,)> = sqlx::query_as(
            "SELECT MAX(version) FROM schema_migrations"
        )
        .fetch_optional(self.db_manager.main_pool())
        .await?;
        
        Ok(version.map(|(v,)| v as u32).unwrap_or(0))
    }
    
    /// Record schema version
    pub async fn record_version(&self, version: u32, name: &str) -> Result<()> {
        sqlx::query(
            "INSERT INTO schema_migrations (version, name) VALUES ($1, $2)"
        )
        .bind(version as i32)
        .bind(name)
        .execute(self.db_manager.main_pool())
        .await?;
        
        Ok(())
    }
}
```

## CSV to Database Import

```rust
/// Import configuration from CSV files
pub struct CsvImporter {
    db_manager: Arc<DatabaseManager>,
    csv_path: PathBuf,
}

impl CsvImporter {
    /// Import all CSV configurations
    pub async fn import_all(&self) -> Result<ImportStats> {
        let mut stats = ImportStats::default();
        
        // Import snippets registry
        stats.snippets = self.import_snippets().await?;
        
        // Import associations
        stats.associations = self.import_associations().await?;
        
        // Import themes
        stats.themes = self.import_themes().await?;
        
        // Import translations
        stats.translations = self.import_translations().await?;
        
        tracing::info!("CSV import completed: {:?}", stats);
        Ok(stats)
    }
    
    async fn import_snippets(&self) -> Result<usize> {
        let csv_file = self.csv_path.join("snippets.csv");
        let mut reader = csv::Reader::from_path(csv_file)?;
        let mut count = 0;
        
        for result in reader.records() {
            let record = result?;
            
            // Parse CSV record into snippet
            let snippet_type = record.get(0).ok_or_else(|| 
                ReedError::Configuration("Missing snippet type".to_string())
            )?;
            
            // Create snippet entity
            let entity = Entity {
                id: Uuid::new_v4(),
                entity_type: EntityType::Snippet,
                semantic_name: None,
                created_at: Utc::now(),
                updated_at: Utc::now(),
                created_by: "system".to_string(),
                data: serde_json::json!({
                    "snippet_type": snippet_type,
                }),
            };
            
            // Store in Redis (UCG)
            self.store_entity_in_ucg(&entity).await?;
            count += 1;
        }
        
        Ok(count)
    }
    
    async fn store_entity_in_ucg(&self, entity: &Entity) -> Result<()> {
        // This would normally go to Redis
        // For migration, we're rebuilding the UCG structure
        let key = format!("entity:{}:{}", entity.entity_type, entity.id);
        let data = serde_json::to_string(entity)?;
        
        // Store in Redis when system starts
        // For now, we track it for later loading
        Ok(())
    }
}

#[derive(Debug, Default)]
pub struct ImportStats {
    pub snippets: usize,
    pub associations: usize,
    pub themes: usize,
    pub translations: usize,
}
```

## Migration Commands

```rust
/// CLI commands for migration
pub struct MigrationCommands;

impl MigrationCommands {
    /// Create backup command
    pub async fn backup(manager: &MigrationManager, version: &str) -> Result<()> {
        println!("Creating backup for version {}...", version);
        
        let info = manager.create_backup(version).await?;
        
        println!("✓ Backup created successfully");
        println!("  File: {}", info.file_path.display());
        println!("  Size: {} MB", info.size_bytes / 1_048_576);
        println!("  Tables: {}", info.table_count);
        
        Ok(())
    }
    
    /// Restore command
    pub async fn restore(manager: &MigrationManager, backup_name: &str) -> Result<()> {
        let backups = manager.list_backups().await?;
        
        let backup = backups.iter()
            .find(|b| b.file_path.to_string_lossy().contains(backup_name))
            .ok_or_else(|| ReedError::Configuration("Backup not found".to_string()))?;
        
        println!("⚠ This will restore the database from backup:");
        println!("  File: {}", backup.file_path.display());
        println!("  Created: {}", backup.created_at);
        println!("  Version: {}", backup.version);
        println!("\nALL CURRENT DATA WILL BE LOST!");
        println!("Continue? [y/N]: ");
        
        // Read confirmation
        let mut input = String::new();
        std::io::stdin().read_line(&mut input)?;
        
        if input.trim().to_lowercase() != "y" {
            println!("Restore cancelled");
            return Ok(());
        }
        
        manager.restore_backup(backup).await?;
        println!("✓ Database restored successfully");
        
        Ok(())
    }
    
    /// List backups command
    pub async fn list(manager: &MigrationManager) -> Result<()> {
        let backups = manager.list_backups().await?;
        
        if backups.is_empty() {
            println!("No backups found");
            return Ok(());
        }
        
        println!("Available backups:");
        println!("{:<40} {:<20} {:<10}", "File", "Created", "Size");
        println!("{}", "-".repeat(70));
        
        for backup in backups {
            println!(
                "{:<40} {:<20} {:<10}",
                backup.file_path.file_name().unwrap().to_string_lossy(),
                backup.created_at.format("%Y-%m-%d %H:%M"),
                format!("{} MB", backup.size_bytes / 1_048_576)
            );
        }
        
        Ok(())
    }
}
```

## Summary

Dieses Migration System bietet:
- **Simple Backup/Restore** statt komplexer Migrationen
- **Full Database Export** mit pg_dump
- **CSV Import** für Konfigurationsdaten
- **Version Tracking** für Schema-Updates
- **CLI Integration** für einfache Verwaltung

KISS-Prinzip: Dump old, import new - fertig!