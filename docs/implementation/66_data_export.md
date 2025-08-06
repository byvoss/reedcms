# Data Import/Export

## Overview

Data Import/Export Implementation für ReedCMS. Bulk Data Operations mit Multiple Format Support.

## Export Manager

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use tokio::io::AsyncWrite;

/// Main export service
pub struct ExportManager {
    exporters: HashMap<ExportFormat, Box<dyn Exporter>>,
    processor: Arc<ExportProcessor>,
    storage: Arc<ExportStorage>,
    config: ExportConfig,
}

impl ExportManager {
    pub fn new(config: ExportConfig) -> Self {
        let mut exporters: HashMap<ExportFormat, Box<dyn Exporter>> = HashMap::new();
        
        // Register exporters
        exporters.insert(ExportFormat::Json, Box::new(JsonExporter::new()));
        exporters.insert(ExportFormat::Csv, Box::new(CsvExporter::new()));
        exporters.insert(ExportFormat::Xml, Box::new(XmlExporter::new()));
        exporters.insert(ExportFormat::Yaml, Box::new(YamlExporter::new()));
        exporters.insert(ExportFormat::Excel, Box::new(ExcelExporter::new()));
        
        let processor = Arc::new(ExportProcessor::new());
        let storage = Arc::new(ExportStorage::new(config.storage.clone()));
        
        Self {
            exporters,
            processor,
            storage,
            config,
        }
    }
    
    /// Export data
    pub async fn export(&self, request: ExportRequest) -> Result<ExportResult> {
        // Validate request
        self.validate_export_request(&request)?;
        
        // Get exporter
        let exporter = self.exporters
            .get(&request.format)
            .ok_or_else(|| ExportError::UnsupportedFormat(request.format))?;
        
        // Create export job
        let job = ExportJob {
            id: ExportJobId::new(),
            request: request.clone(),
            status: ExportStatus::Processing,
            started_at: chrono::Utc::now(),
            progress: ExportProgress::default(),
        };
        
        // Process export
        let export_file = self.processor
            .process(&job, exporter.as_ref())
            .await?;
        
        // Store result
        let download_url = self.storage
            .store(&job.id, export_file)
            .await?;
        
        Ok(ExportResult {
            job_id: job.id,
            format: request.format,
            download_url,
            expires_at: chrono::Utc::now() + self.config.download_expiry,
            metadata: job.progress.to_metadata(),
        })
    }
    
    /// Stream export
    pub async fn export_stream<W>(
        &self,
        request: ExportRequest,
        writer: W,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        let exporter = self.exporters
            .get(&request.format)
            .ok_or_else(|| ExportError::UnsupportedFormat(request.format))?;
        
        // Create streaming job
        let job = ExportJob {
            id: ExportJobId::new(),
            request,
            status: ExportStatus::Processing,
            started_at: chrono::Utc::now(),
            progress: ExportProgress::default(),
        };
        
        // Stream data
        self.processor
            .stream(&job, exporter.as_ref(), writer)
            .await
    }
}

/// Export processor
pub struct ExportProcessor {
    chunk_size: usize,
}

impl ExportProcessor {
    /// Process export job
    pub async fn process(
        &self,
        job: &ExportJob,
        exporter: &dyn Exporter,
    ) -> Result<ExportFile> {
        let mut file = ExportFile::new(&job.id);
        let mut writer = file.writer().await?;
        
        // Write header
        exporter.write_header(&mut writer, &job.request).await?;
        
        // Export data in chunks
        let mut offset = 0;
        let mut total_exported = 0;
        
        loop {
            let items = self.fetch_items(&job.request, offset, self.chunk_size).await?;
            if items.is_empty() {
                break;
            }
            
            for item in &items {
                exporter.write_item(&mut writer, item).await?;
                total_exported += 1;
            }
            
            offset += items.len();
            
            // Update progress
            job.progress.update(total_exported, None);
        }
        
        // Write footer
        exporter.write_footer(&mut writer).await?;
        
        writer.flush().await?;
        
        Ok(file)
    }
    
    /// Stream export
    pub async fn stream<W>(
        &self,
        job: &ExportJob,
        exporter: &dyn Exporter,
        mut writer: W,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        // Write header
        exporter.write_header(&mut writer, &job.request).await?;
        
        // Stream data
        let mut offset = 0;
        
        loop {
            let items = self.fetch_items(&job.request, offset, self.chunk_size).await?;
            if items.is_empty() {
                break;
            }
            
            for item in &items {
                exporter.write_item(&mut writer, item).await?;
            }
            
            offset += items.len();
        }
        
        // Write footer
        exporter.write_footer(&mut writer).await?;
        
        writer.flush().await?;
        
        Ok(())
    }
    
    /// Fetch items to export
    async fn fetch_items(
        &self,
        request: &ExportRequest,
        offset: usize,
        limit: usize,
    ) -> Result<Vec<ExportItem>> {
        match &request.source {
            ExportSource::Content { content_type, filter } => {
                let contents = get_content_service()
                    .list_content(filter.clone(), Pagination { offset, limit })
                    .await?;
                
                Ok(contents.into_iter()
                    .map(|c| ExportItem::Content(c))
                    .collect())
            }
            ExportSource::Users { filter } => {
                let users = get_user_service()
                    .list_users(filter.clone(), Pagination { offset, limit })
                    .await?;
                
                Ok(users.into_iter()
                    .map(|u| ExportItem::User(u))
                    .collect())
            }
            ExportSource::Query { sql } => {
                let rows = execute_export_query(sql, offset, limit).await?;
                
                Ok(rows.into_iter()
                    .map(|r| ExportItem::Row(r))
                    .collect())
            }
        }
    }
}

/// Export request
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExportRequest {
    pub format: ExportFormat,
    pub source: ExportSource,
    pub fields: Option<Vec<String>>,
    pub options: ExportOptions,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ExportFormat {
    Json,
    Csv,
    Xml,
    Yaml,
    Excel,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ExportSource {
    Content {
        content_type: String,
        filter: ContentFilter,
    },
    Users {
        filter: UserFilter,
    },
    Query {
        sql: String,
    },
}
```

## Import Manager

```rust
/// Main import service
pub struct ImportManager {
    importers: HashMap<ImportFormat, Box<dyn Importer>>,
    processor: Arc<ImportProcessor>,
    validator: Arc<ImportValidator>,
    config: ImportConfig,
}

impl ImportManager {
    pub fn new(config: ImportConfig) -> Self {
        let mut importers: HashMap<ImportFormat, Box<dyn Importer>> = HashMap::new();
        
        // Register importers
        importers.insert(ImportFormat::Json, Box::new(JsonImporter::new()));
        importers.insert(ImportFormat::Csv, Box::new(CsvImporter::new()));
        importers.insert(ImportFormat::Xml, Box::new(XmlImporter::new()));
        importers.insert(ImportFormat::Yaml, Box::new(YamlImporter::new()));
        importers.insert(ImportFormat::Excel, Box::new(ExcelImporter::new()));
        
        let processor = Arc::new(ImportProcessor::new());
        let validator = Arc::new(ImportValidator::new());
        
        Self {
            importers,
            processor,
            validator,
            config,
        }
    }
    
    /// Import data
    pub async fn import(&self, request: ImportRequest) -> Result<ImportResult> {
        // Validate request
        self.validate_import_request(&request)?;
        
        // Get importer
        let importer = self.importers
            .get(&request.format)
            .ok_or_else(|| ImportError::UnsupportedFormat(request.format))?;
        
        // Create import job
        let job = ImportJob {
            id: ImportJobId::new(),
            request: request.clone(),
            status: ImportStatus::Processing,
            started_at: chrono::Utc::now(),
            progress: ImportProgress::default(),
        };
        
        // Process import
        let result = self.processor
            .process(&job, importer.as_ref())
            .await?;
        
        Ok(result)
    }
    
    /// Preview import
    pub async fn preview(
        &self,
        request: ImportRequest,
        limit: usize,
    ) -> Result<ImportPreview> {
        let importer = self.importers
            .get(&request.format)
            .ok_or_else(|| ImportError::UnsupportedFormat(request.format))?;
        
        // Parse file
        let items = importer
            .parse_file(&request.file_path, Some(limit))
            .await?;
        
        // Validate items
        let validation_results = self.validator
            .validate_items(&items, &request.target)
            .await?;
        
        Ok(ImportPreview {
            total_items: items.len(),
            sample_items: items.into_iter().take(10).collect(),
            validation_summary: validation_results.summary(),
            field_mapping: self.suggest_field_mapping(&items[0], &request.target)?,
        })
    }
}

/// Import processor
pub struct ImportProcessor {
    batch_size: usize,
    transaction_mode: TransactionMode,
}

impl ImportProcessor {
    /// Process import job
    pub async fn process(
        &self,
        job: &ImportJob,
        importer: &dyn Importer,
    ) -> Result<ImportResult> {
        let mut result = ImportResult::default();
        
        // Parse file
        let items = importer
            .parse_file(&job.request.file_path, None)
            .await?;
        
        result.total_items = items.len();
        
        // Process in batches
        for chunk in items.chunks(self.batch_size) {
            match self.transaction_mode {
                TransactionMode::PerBatch => {
                    self.process_batch_transactional(chunk, &job.request, &mut result).await?;
                }
                TransactionMode::PerItem => {
                    self.process_batch_individual(chunk, &job.request, &mut result).await?;
                }
                TransactionMode::All => {
                    // Handled at higher level
                    self.process_batch(chunk, &job.request, &mut result).await?;
                }
            }
            
            // Update progress
            job.progress.update(result.imported_count, Some(result.total_items));
        }
        
        Ok(result)
    }
    
    /// Process batch of items
    async fn process_batch(
        &self,
        items: &[ImportItem],
        request: &ImportRequest,
        result: &mut ImportResult,
    ) -> Result<()> {
        for item in items {
            match self.process_item(item, request).await {
                Ok(()) => {
                    result.imported_count += 1;
                }
                Err(e) => {
                    result.error_count += 1;
                    result.errors.push(ImportError {
                        line: item.line_number(),
                        error: e.to_string(),
                        data: Some(item.raw_data()),
                    });
                    
                    if !request.options.continue_on_error {
                        return Err(e);
                    }
                }
            }
        }
        
        Ok(())
    }
    
    /// Process single item
    async fn process_item(
        &self,
        item: &ImportItem,
        request: &ImportRequest,
    ) -> Result<()> {
        match &request.target {
            ImportTarget::Content { content_type } => {
                let content = self.map_to_content(item, content_type, &request.field_mapping)?;
                
                if request.options.update_existing {
                    get_content_service()
                        .upsert_content(content)
                        .await?;
                } else {
                    get_content_service()
                        .create_content(content)
                        .await?;
                }
            }
            ImportTarget::Users => {
                let user = self.map_to_user(item, &request.field_mapping)?;
                
                if request.options.update_existing {
                    get_user_service()
                        .upsert_user(user)
                        .await?;
                } else {
                    get_user_service()
                        .create_user(user)
                        .await?;
                }
            }
            ImportTarget::Custom { handler } => {
                handler.process_item(item).await?;
            }
        }
        
        Ok(())
    }
}
```

## Format Exporters

```rust
/// Exporter trait
#[async_trait]
pub trait Exporter: Send + Sync {
    async fn write_header<W>(
        &self,
        writer: &mut W,
        request: &ExportRequest,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send;
    
    async fn write_item<W>(
        &self,
        writer: &mut W,
        item: &ExportItem,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send;
    
    async fn write_footer<W>(
        &self,
        writer: &mut W,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send;
}

/// JSON exporter
struct JsonExporter {
    pretty: bool,
}

#[async_trait]
impl Exporter for JsonExporter {
    async fn write_header<W>(
        &self,
        writer: &mut W,
        _request: &ExportRequest,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        writer.write_all(b"[\n").await?;
        Ok(())
    }
    
    async fn write_item<W>(
        &self,
        writer: &mut W,
        item: &ExportItem,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        let json = if self.pretty {
            serde_json::to_string_pretty(item)?
        } else {
            serde_json::to_string(item)?
        };
        
        writer.write_all(json.as_bytes()).await?;
        writer.write_all(b",\n").await?;
        
        Ok(())
    }
    
    async fn write_footer<W>(
        &self,
        writer: &mut W,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        writer.write_all(b"]\n").await?;
        Ok(())
    }
}

/// CSV exporter
struct CsvExporter {
    delimiter: u8,
    quote_style: csv::QuoteStyle,
}

#[async_trait]
impl Exporter for CsvExporter {
    async fn write_header<W>(
        &self,
        writer: &mut W,
        request: &ExportRequest,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        if let Some(fields) = &request.fields {
            let header = fields.join(&(self.delimiter as char).to_string());
            writer.write_all(header.as_bytes()).await?;
            writer.write_all(b"\n").await?;
        }
        
        Ok(())
    }
    
    async fn write_item<W>(
        &self,
        writer: &mut W,
        item: &ExportItem,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        let values = item.to_csv_values();
        let row = values.join(&(self.delimiter as char).to_string());
        
        writer.write_all(row.as_bytes()).await?;
        writer.write_all(b"\n").await?;
        
        Ok(())
    }
    
    async fn write_footer<W>(
        &self,
        _writer: &mut W,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        Ok(())
    }
}

/// Excel exporter
struct ExcelExporter {
    worksheet_name: String,
}

#[async_trait]
impl Exporter for ExcelExporter {
    async fn write_header<W>(
        &self,
        writer: &mut W,
        request: &ExportRequest,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        // Create workbook
        let workbook = xlsxwriter::Workbook::new();
        let mut worksheet = workbook.add_worksheet(Some(&self.worksheet_name))?;
        
        // Write headers
        if let Some(fields) = &request.fields {
            for (col, field) in fields.iter().enumerate() {
                worksheet.write_string(0, col as u16, field, None)?;
            }
        }
        
        Ok(())
    }
    
    async fn write_item<W>(
        &self,
        writer: &mut W,
        item: &ExportItem,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        // Write row data
        let values = item.to_excel_values();
        
        for (col, value) in values.iter().enumerate() {
            match value {
                ExcelValue::String(s) => {
                    worksheet.write_string(self.current_row, col as u16, s, None)?;
                }
                ExcelValue::Number(n) => {
                    worksheet.write_number(self.current_row, col as u16, *n, None)?;
                }
                ExcelValue::Date(d) => {
                    worksheet.write_datetime(self.current_row, col as u16, d, None)?;
                }
            }
        }
        
        self.current_row += 1;
        
        Ok(())
    }
    
    async fn write_footer<W>(
        &self,
        writer: &mut W,
    ) -> Result<()>
    where
        W: AsyncWrite + Unpin + Send,
    {
        // Close and write workbook
        self.workbook.close()?;
        
        Ok(())
    }
}
```

## Data Transformation

```rust
/// Data transformer for import/export
pub struct DataTransformer {
    transformations: Vec<Box<dyn Transformation>>,
}

impl DataTransformer {
    /// Apply transformations
    pub fn transform(&self, data: &mut TransformableData) -> Result<()> {
        for transformation in &self.transformations {
            transformation.apply(data)?;
        }
        
        Ok(())
    }
}

/// Transformation trait
trait Transformation: Send + Sync {
    fn apply(&self, data: &mut TransformableData) -> Result<()>;
}

/// Field mapping transformation
struct FieldMappingTransformation {
    mapping: HashMap<String, String>,
}

impl Transformation for FieldMappingTransformation {
    fn apply(&self, data: &mut TransformableData) -> Result<()> {
        let mut mapped = serde_json::Map::new();
        
        if let Some(obj) = data.as_object() {
            for (key, value) in obj {
                if let Some(mapped_key) = self.mapping.get(key) {
                    mapped.insert(mapped_key.clone(), value.clone());
                } else if !self.mapping.values().any(|v| v == key) {
                    // Keep unmapped fields
                    mapped.insert(key.clone(), value.clone());
                }
            }
        }
        
        *data = TransformableData::Object(mapped);
        
        Ok(())
    }
}

/// Value transformation
struct ValueTransformation {
    field: String,
    transformer: Box<dyn Fn(&serde_json::Value) -> serde_json::Value + Send + Sync>,
}

impl Transformation for ValueTransformation {
    fn apply(&self, data: &mut TransformableData) -> Result<()> {
        if let Some(obj) = data.as_object_mut() {
            if let Some(value) = obj.get_mut(&self.field) {
                *value = (self.transformer)(value);
            }
        }
        
        Ok(())
    }
}

/// Data validation
struct DataValidation {
    rules: Vec<ValidationRule>,
}

impl DataValidation {
    /// Validate data
    pub fn validate(&self, data: &TransformableData) -> ValidationResult {
        let mut errors = Vec::new();
        
        for rule in &self.rules {
            if let Err(e) = rule.validate(data) {
                errors.push(e);
            }
        }
        
        ValidationResult {
            valid: errors.is_empty(),
            errors,
        }
    }
}

/// Validation rule
struct ValidationRule {
    field: String,
    validator: Box<dyn Fn(&serde_json::Value) -> Result<()> + Send + Sync>,
}

impl ValidationRule {
    /// Required field rule
    pub fn required(field: String) -> Self {
        Self {
            field: field.clone(),
            validator: Box::new(move |value| {
                if value.is_null() {
                    Err(ValidationError::FieldRequired(field.clone()))
                } else {
                    Ok(())
                }
            }),
        }
    }
    
    /// Pattern rule
    pub fn pattern(field: String, pattern: regex::Regex) -> Self {
        Self {
            field: field.clone(),
            validator: Box::new(move |value| {
                if let Some(s) = value.as_str() {
                    if !pattern.is_match(s) {
                        Err(ValidationError::PatternMismatch(field.clone()))
                    } else {
                        Ok(())
                    }
                } else {
                    Ok(())
                }
            }),
        }
    }
}
```

## Batch Operations

```rust
/// Batch operation manager
pub struct BatchOperationManager {
    import_manager: Arc<ImportManager>,
    export_manager: Arc<ExportManager>,
    scheduler: Arc<BatchScheduler>,
}

impl BatchOperationManager {
    /// Schedule batch export
    pub async fn schedule_export(
        &self,
        request: BatchExportRequest,
    ) -> Result<BatchJobId> {
        let job = BatchJob {
            id: BatchJobId::new(),
            operation: BatchOperation::Export(request),
            schedule: request.schedule,
            status: BatchJobStatus::Scheduled,
            created_at: chrono::Utc::now(),
        };
        
        self.scheduler.schedule(job).await
    }
    
    /// Schedule batch import
    pub async fn schedule_import(
        &self,
        request: BatchImportRequest,
    ) -> Result<BatchJobId> {
        let job = BatchJob {
            id: BatchJobId::new(),
            operation: BatchOperation::Import(request),
            schedule: request.schedule,
            status: BatchJobStatus::Scheduled,
            created_at: chrono::Utc::now(),
        };
        
        self.scheduler.schedule(job).await
    }
    
    /// Run batch sync
    pub async fn sync(
        &self,
        source: DataSource,
        target: DataTarget,
        options: SyncOptions,
    ) -> Result<SyncResult> {
        // Export from source
        let export_result = self.export_manager
            .export(ExportRequest {
                format: ExportFormat::Json,
                source: source.to_export_source(),
                fields: None,
                options: Default::default(),
            })
            .await?;
        
        // Transform data if needed
        let transformed_data = if let Some(transformer) = &options.transformer {
            transformer.transform_file(&export_result.file_path).await?
        } else {
            export_result.file_path
        };
        
        // Import to target
        let import_result = self.import_manager
            .import(ImportRequest {
                format: ImportFormat::Json,
                file_path: transformed_data,
                target: target.to_import_target(),
                field_mapping: options.field_mapping,
                options: ImportOptions {
                    update_existing: options.update_existing,
                    continue_on_error: options.continue_on_error,
                    validation_mode: options.validation_mode,
                },
            })
            .await?;
        
        Ok(SyncResult {
            exported_count: export_result.metadata.total_items,
            imported_count: import_result.imported_count,
            error_count: import_result.error_count,
            duration: chrono::Utc::now() - export_result.started_at,
        })
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ImportExportConfig {
    pub export: ExportConfig,
    pub import: ImportConfig,
    pub batch: BatchConfig,
    pub storage: StorageConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExportConfig {
    pub max_export_size: usize,
    pub chunk_size: usize,
    pub download_expiry: chrono::Duration,
    pub allowed_formats: Vec<ExportFormat>,
    pub storage: StorageConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ImportConfig {
    pub max_import_size: usize,
    pub batch_size: usize,
    pub transaction_mode: TransactionMode,
    pub allowed_formats: Vec<ImportFormat>,
    pub validation_mode: ValidationMode,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BatchConfig {
    pub max_concurrent_jobs: usize,
    pub job_timeout: Duration,
    pub retry_attempts: u32,
    pub schedule_lookahead: chrono::Duration,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum TransactionMode {
    PerItem,
    PerBatch,
    All,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ValidationMode {
    Strict,
    Lenient,
    None,
}

impl Default for ImportExportConfig {
    fn default() -> Self {
        Self {
            export: ExportConfig {
                max_export_size: 100 * 1024 * 1024, // 100MB
                chunk_size: 1000,
                download_expiry: chrono::Duration::hours(24),
                allowed_formats: vec![
                    ExportFormat::Json,
                    ExportFormat::Csv,
                    ExportFormat::Excel,
                ],
                storage: StorageConfig::default(),
            },
            import: ImportConfig {
                max_import_size: 50 * 1024 * 1024, // 50MB
                batch_size: 100,
                transaction_mode: TransactionMode::PerBatch,
                allowed_formats: vec![
                    ImportFormat::Json,
                    ImportFormat::Csv,
                    ImportFormat::Excel,
                ],
                validation_mode: ValidationMode::Strict,
            },
            batch: BatchConfig {
                max_concurrent_jobs: 5,
                job_timeout: Duration::from_secs(3600),
                retry_attempts: 3,
                schedule_lookahead: chrono::Duration::days(7),
            },
            storage: StorageConfig::default(),
        }
    }
}
```

## Summary

Diese Data Import/Export Implementation bietet:
- **Multiple Formats** - JSON, CSV, XML, YAML, Excel
- **Streaming Support** - Large File Handling
- **Data Transformation** - Field Mapping, Value Conversion
- **Validation** - Import Validation with Preview
- **Batch Operations** - Scheduled Import/Export
- **Error Handling** - Detailed Error Reporting

Complete Import/Export System für Data Portability.