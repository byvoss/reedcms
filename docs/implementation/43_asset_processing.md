# Asset Processing

## Overview

Asset Processing Pipeline für ReedCMS. Image Optimization, CSS/JS Minification und Asset Fingerprinting mit EPC-aware Resolution.

## Asset Processor Core

```rust
use std::path::{Path, PathBuf};
use sha2::{Sha256, Digest};
use image::{DynamicImage, ImageFormat};

/// Main asset processor
pub struct AssetProcessor {
    config: AssetConfig,
    image_processor: ImageProcessor,
    css_processor: CssProcessor,
    js_processor: JsProcessor,
    cache_dir: PathBuf,
}

impl AssetProcessor {
    pub fn new(config: AssetConfig) -> Result<Self> {
        let cache_dir = config.cache_dir.clone();
        std::fs::create_dir_all(&cache_dir)?;
        
        Ok(Self {
            config: config.clone(),
            image_processor: ImageProcessor::new(config.image),
            css_processor: CssProcessor::new(config.css),
            js_processor: JsProcessor::new(config.js),
            cache_dir,
        })
    }
    
    /// Process asset based on type
    pub async fn process_asset(
        &self,
        input_path: &Path,
        options: ProcessOptions,
    ) -> Result<ProcessedAsset> {
        let ext = input_path.extension()
            .and_then(|e| e.to_str())
            .unwrap_or("");
        
        match ext.to_lowercase().as_str() {
            // Images
            "jpg" | "jpeg" | "png" | "gif" | "webp" => {
                self.image_processor.process(input_path, options).await
            }
            // CSS
            "css" => {
                self.css_processor.process(input_path, options).await
            }
            // JavaScript
            "js" => {
                self.js_processor.process(input_path, options).await
            }
            // Pass through
            _ => {
                self.passthrough_asset(input_path, options).await
            }
        }
    }
    
    /// Generate asset fingerprint
    pub fn generate_fingerprint(content: &[u8]) -> String {
        let mut hasher = Sha256::new();
        hasher.update(content);
        format!("{:x}", hasher.finalize())
            .chars()
            .take(8)
            .collect()
    }
    
    /// Get cached asset path
    fn get_cache_path(
        &self,
        original: &Path,
        fingerprint: &str,
        suffix: Option<&str>,
    ) -> PathBuf {
        let filename = original.file_stem()
            .and_then(|s| s.to_str())
            .unwrap_or("asset");
        
        let ext = original.extension()
            .and_then(|e| e.to_str())
            .unwrap_or("");
        
        let cached_name = if let Some(suffix) = suffix {
            format!("{}-{}-{}.{}", filename, suffix, fingerprint, ext)
        } else {
            format!("{}-{}.{}", filename, fingerprint, ext)
        };
        
        self.cache_dir.join(cached_name)
    }
}

/// Processed asset result
#[derive(Debug, Clone)]
pub struct ProcessedAsset {
    pub original_path: PathBuf,
    pub processed_path: PathBuf,
    pub fingerprint: String,
    pub size: u64,
    pub mime_type: String,
    pub metadata: AssetMetadata,
}

#[derive(Debug, Clone)]
pub struct AssetMetadata {
    pub width: Option<u32>,
    pub height: Option<u32>,
    pub format: Option<String>,
    pub optimized: bool,
}
```

## Image Processing

```rust
/// Image processor with optimization
pub struct ImageProcessor {
    config: ImageConfig,
    optimizer: ImageOptimizer,
}

impl ImageProcessor {
    pub fn new(config: ImageConfig) -> Self {
        Self {
            config: config.clone(),
            optimizer: ImageOptimizer::new(config),
        }
    }
    
    /// Process image with options
    pub async fn process(
        &self,
        input_path: &Path,
        options: ProcessOptions,
    ) -> Result<ProcessedAsset> {
        // Load image
        let img = image::open(input_path)?;
        let original_size = std::fs::metadata(input_path)?.len();
        
        // Apply transformations
        let processed = if let Some(resize) = options.resize {
            self.resize_image(img, resize)?
        } else {
            img
        };
        
        // Optimize
        let (optimized, format) = self.optimizer
            .optimize(&processed, options.quality)
            .await?;
        
        // Generate fingerprint
        let fingerprint = AssetProcessor::generate_fingerprint(&optimized);
        
        // Determine output path
        let suffix = options.resize.as_ref()
            .map(|r| format!("{}x{}", r.width, r.height));
        
        let output_path = self.get_output_path(
            input_path,
            &fingerprint,
            suffix.as_deref(),
        );
        
        // Write optimized image
        tokio::fs::write(&output_path, &optimized).await?;
        
        Ok(ProcessedAsset {
            original_path: input_path.to_path_buf(),
            processed_path: output_path,
            fingerprint,
            size: optimized.len() as u64,
            mime_type: format.to_mime_type().to_string(),
            metadata: AssetMetadata {
                width: Some(processed.width()),
                height: Some(processed.height()),
                format: Some(format.to_string()),
                optimized: optimized.len() < original_size as usize,
            },
        })
    }
    
    /// Resize image
    fn resize_image(
        &self,
        img: DynamicImage,
        resize: ResizeOptions,
    ) -> Result<DynamicImage> {
        use image::imageops::FilterType;
        
        let filter = match resize.filter {
            ResizeFilter::Nearest => FilterType::Nearest,
            ResizeFilter::Triangle => FilterType::Triangle,
            ResizeFilter::CatmullRom => FilterType::CatmullRom,
            ResizeFilter::Gaussian => FilterType::Gaussian,
            ResizeFilter::Lanczos3 => FilterType::Lanczos3,
        };
        
        let resized = match resize.mode {
            ResizeMode::Exact => {
                img.resize_exact(resize.width, resize.height, filter)
            }
            ResizeMode::Fit => {
                img.resize(resize.width, resize.height, filter)
            }
            ResizeMode::Fill => {
                img.resize_to_fill(resize.width, resize.height, filter)
            }
            ResizeMode::Crop => {
                let ratio = (resize.width as f32 / resize.height as f32)
                    .min(img.width() as f32 / img.height() as f32);
                
                let crop_width = (resize.width as f32 / ratio) as u32;
                let crop_height = (resize.height as f32 / ratio) as u32;
                
                img.crop_imm(
                    (img.width() - crop_width) / 2,
                    (img.height() - crop_height) / 2,
                    crop_width,
                    crop_height,
                ).resize_exact(resize.width, resize.height, filter)
            }
        };
        
        Ok(resized)
    }
}

/// Image optimization
struct ImageOptimizer {
    config: ImageConfig,
}

impl ImageOptimizer {
    pub fn new(config: ImageConfig) -> Self {
        Self { config }
    }
    
    /// Optimize image for web
    pub async fn optimize(
        &self,
        img: &DynamicImage,
        quality: Option<u8>,
    ) -> Result<(Vec<u8>, ImageFormat)> {
        let quality = quality.unwrap_or(self.config.default_quality);
        
        // Determine best format
        let format = self.determine_format(img);
        
        // Encode with optimization
        let optimized = match format {
            ImageFormat::Jpeg => {
                self.optimize_jpeg(img, quality)?
            }
            ImageFormat::Png => {
                self.optimize_png(img)?
            }
            ImageFormat::WebP => {
                self.optimize_webp(img, quality)?
            }
            _ => {
                // Fallback to PNG
                self.optimize_png(img)?
            }
        };
        
        Ok((optimized, format))
    }
    
    /// Optimize JPEG
    fn optimize_jpeg(
        &self,
        img: &DynamicImage,
        quality: u8,
    ) -> Result<Vec<u8>> {
        use image::codecs::jpeg::JpegEncoder;
        
        let mut buffer = Vec::new();
        let encoder = JpegEncoder::new_with_quality(&mut buffer, quality);
        encoder.encode_image(img)?;
        
        // Further optimize with mozjpeg if available
        if self.config.use_mozjpeg {
            buffer = self.mozjpeg_optimize(buffer, quality)?;
        }
        
        Ok(buffer)
    }
    
    /// Optimize PNG
    fn optimize_png(&self, img: &DynamicImage) -> Result<Vec<u8>> {
        use image::codecs::png::{PngEncoder, CompressionType};
        
        let mut buffer = Vec::new();
        let encoder = PngEncoder::new_with_quality(
            &mut buffer,
            CompressionType::Best,
            image::codecs::png::FilterType::Sub,
        );
        encoder.encode_image(img)?;
        
        // Further optimize with oxipng if available
        if self.config.use_oxipng {
            buffer = self.oxipng_optimize(buffer)?;
        }
        
        Ok(buffer)
    }
}

#[derive(Debug, Clone)]
pub struct ResizeOptions {
    pub width: u32,
    pub height: u32,
    pub mode: ResizeMode,
    pub filter: ResizeFilter,
}

#[derive(Debug, Clone)]
pub enum ResizeMode {
    Exact,  // Exact dimensions, may distort
    Fit,    // Fit within bounds, maintain aspect
    Fill,   // Fill bounds, maintain aspect, crop excess
    Crop,   // Center crop to dimensions
}

#[derive(Debug, Clone)]
pub enum ResizeFilter {
    Nearest,
    Triangle,
    CatmullRom,
    Gaussian,
    Lanczos3,
}
```

## CSS Processing

```rust
/// CSS processor with minification
pub struct CssProcessor {
    config: CssConfig,
    minifier: CssMinifier,
}

impl CssProcessor {
    pub fn new(config: CssConfig) -> Self {
        Self {
            config: config.clone(),
            minifier: CssMinifier::new(config),
        }
    }
    
    /// Process CSS file
    pub async fn process(
        &self,
        input_path: &Path,
        options: ProcessOptions,
    ) -> Result<ProcessedAsset> {
        // Read CSS
        let content = tokio::fs::read_to_string(input_path).await?;
        
        // Process imports
        let resolved = self.resolve_imports(&content, input_path).await?;
        
        // Minify if enabled
        let processed = if options.minify.unwrap_or(self.config.minify) {
            self.minifier.minify(&resolved)?
        } else {
            resolved
        };
        
        // Post-process
        let final_css = self.post_process(&processed)?;
        
        // Generate fingerprint
        let fingerprint = AssetProcessor::generate_fingerprint(final_css.as_bytes());
        
        // Write processed file
        let output_path = self.get_output_path(input_path, &fingerprint);
        tokio::fs::write(&output_path, &final_css).await?;
        
        Ok(ProcessedAsset {
            original_path: input_path.to_path_buf(),
            processed_path: output_path,
            fingerprint,
            size: final_css.len() as u64,
            mime_type: "text/css".to_string(),
            metadata: AssetMetadata {
                width: None,
                height: None,
                format: Some("css".to_string()),
                optimized: final_css.len() < content.len(),
            },
        })
    }
    
    /// Resolve @import statements
    async fn resolve_imports(
        &self,
        content: &str,
        base_path: &Path,
    ) -> Result<String> {
        let import_regex = regex::Regex::new(
            r#"@import\s+["']([^"']+)["'];"#
        )?;
        
        let mut resolved = content.to_string();
        let base_dir = base_path.parent().unwrap_or(Path::new(""));
        
        for cap in import_regex.captures_iter(content) {
            let import_path = &cap[1];
            let full_path = base_dir.join(import_path);
            
            if full_path.exists() {
                let imported = tokio::fs::read_to_string(&full_path).await?;
                // Recursively resolve imports in imported file
                let imported_resolved = Box::pin(
                    self.resolve_imports(&imported, &full_path)
                ).await?;
                
                resolved = resolved.replace(&cap[0], &imported_resolved);
            }
        }
        
        Ok(resolved)
    }
    
    /// Post-process CSS
    fn post_process(&self, css: &str) -> Result<String> {
        let mut processed = css.to_string();
        
        // Add vendor prefixes if enabled
        if self.config.autoprefixer {
            processed = self.add_vendor_prefixes(&processed)?;
        }
        
        // Optimize colors
        processed = self.optimize_colors(&processed);
        
        // Remove unused whitespace
        processed = self.remove_extra_whitespace(&processed);
        
        Ok(processed)
    }
}

/// CSS minification
struct CssMinifier {
    config: CssConfig,
}

impl CssMinifier {
    pub fn new(config: CssConfig) -> Self {
        Self { config }
    }
    
    /// Minify CSS
    pub fn minify(&self, css: &str) -> Result<String> {
        let mut minified = String::with_capacity(css.len());
        let mut in_string = false;
        let mut string_char = ' ';
        let mut prev_char = ' ';
        
        for ch in css.chars() {
            // Handle strings
            if !in_string && (ch == '"' || ch == '\'') {
                in_string = true;
                string_char = ch;
                minified.push(ch);
            } else if in_string && ch == string_char && prev_char != '\\' {
                in_string = false;
                minified.push(ch);
            } else if in_string {
                minified.push(ch);
            } else {
                // Outside strings - minify
                match ch {
                    // Remove comments
                    '/' if css.chars().nth(minified.len() + 1) == Some('*') => {
                        // Skip comment
                        continue;
                    }
                    // Remove unnecessary whitespace
                    ' ' | '\t' | '\n' | '\r' => {
                        if !minified.is_empty() && 
                           !matches!(minified.chars().last(), Some(' ' | '{' | '}' | ';' | ':' | ',')) {
                            minified.push(' ');
                        }
                    }
                    // Keep other characters
                    _ => minified.push(ch),
                }
            }
            
            prev_char = ch;
        }
        
        Ok(minified)
    }
}
```

## JavaScript Processing

```rust
/// JavaScript processor
pub struct JsProcessor {
    config: JsConfig,
    minifier: JsMinifier,
}

impl JsProcessor {
    pub fn new(config: JsConfig) -> Self {
        Self {
            config: config.clone(),
            minifier: JsMinifier::new(config),
        }
    }
    
    /// Process JavaScript file
    pub async fn process(
        &self,
        input_path: &Path,
        options: ProcessOptions,
    ) -> Result<ProcessedAsset> {
        // Read JS
        let content = tokio::fs::read_to_string(input_path).await?;
        
        // Process based on module type
        let processed = if self.is_es_module(&content) {
            self.process_es_module(&content, input_path).await?
        } else {
            content.clone()
        };
        
        // Minify if enabled
        let final_js = if options.minify.unwrap_or(self.config.minify) {
            self.minifier.minify(&processed)?
        } else {
            processed
        };
        
        // Generate fingerprint
        let fingerprint = AssetProcessor::generate_fingerprint(final_js.as_bytes());
        
        // Write processed file
        let output_path = self.get_output_path(input_path, &fingerprint);
        tokio::fs::write(&output_path, &final_js).await?;
        
        Ok(ProcessedAsset {
            original_path: input_path.to_path_buf(),
            processed_path: output_path,
            fingerprint,
            size: final_js.len() as u64,
            mime_type: "application/javascript".to_string(),
            metadata: AssetMetadata {
                width: None,
                height: None,
                format: Some("js".to_string()),
                optimized: final_js.len() < content.len(),
            },
        })
    }
    
    /// Check if ES module
    fn is_es_module(&self, content: &str) -> bool {
        content.contains("import ") || content.contains("export ")
    }
    
    /// Process ES modules
    async fn process_es_module(
        &self,
        content: &str,
        base_path: &Path,
    ) -> Result<String> {
        // Simple import resolution
        let import_regex = regex::Regex::new(
            r#"import\s+.*?from\s+["']([^"']+)["']"#
        )?;
        
        let mut processed = content.to_string();
        
        for cap in import_regex.captures_iter(content) {
            let import_path = &cap[1];
            
            // Resolve relative imports
            if import_path.starts_with('.') {
                let resolved_path = self.resolve_import_path(
                    import_path,
                    base_path,
                )?;
                
                processed = processed.replace(
                    import_path,
                    &resolved_path,
                );
            }
        }
        
        Ok(processed)
    }
}

/// JavaScript minification
struct JsMinifier {
    config: JsConfig,
}

impl JsMinifier {
    pub fn new(config: JsConfig) -> Self {
        Self { config }
    }
    
    /// Basic JS minification
    pub fn minify(&self, js: &str) -> Result<String> {
        // Very basic minification - in production use swc or terser
        let minified = js
            .lines()
            .map(|line| line.trim())
            .filter(|line| !line.is_empty())
            .filter(|line| !line.starts_with("//"))
            .collect::<Vec<_>>()
            .join(" ");
        
        Ok(minified)
    }
}
```

## Asset Manifest

```rust
/// Asset manifest for mapping original to processed files
#[derive(Debug, Serialize, Deserialize)]
pub struct AssetManifest {
    pub version: String,
    pub assets: HashMap<String, AssetEntry>,
    pub generated_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct AssetEntry {
    pub original: String,
    pub processed: String,
    pub fingerprint: String,
    pub size: u64,
    pub mime_type: String,
    pub integrity: Option<String>,
}

impl AssetManifest {
    /// Create new manifest
    pub fn new() -> Self {
        Self {
            version: "1.0".to_string(),
            assets: HashMap::new(),
            generated_at: chrono::Utc::now(),
        }
    }
    
    /// Add processed asset
    pub fn add_asset(&mut self, asset: ProcessedAsset) {
        let key = asset.original_path
            .to_string_lossy()
            .to_string();
        
        self.assets.insert(key, AssetEntry {
            original: asset.original_path.to_string_lossy().to_string(),
            processed: asset.processed_path.to_string_lossy().to_string(),
            fingerprint: asset.fingerprint,
            size: asset.size,
            mime_type: asset.mime_type,
            integrity: None, // TODO: Calculate SRI hash
        });
    }
    
    /// Save manifest to file
    pub async fn save(&self, path: &Path) -> Result<()> {
        let json = serde_json::to_string_pretty(self)?;
        tokio::fs::write(path, json).await?;
        Ok(())
    }
    
    /// Load manifest from file
    pub async fn load(path: &Path) -> Result<Self> {
        let json = tokio::fs::read_to_string(path).await?;
        let manifest = serde_json::from_str(&json)?;
        Ok(manifest)
    }
    
    /// Get processed path for original
    pub fn get_processed_path(&self, original: &str) -> Option<&str> {
        self.assets.get(original)
            .map(|entry| entry.processed.as_str())
    }
}
```

## Asset Configuration

```rust
#[derive(Debug, Clone)]
pub struct AssetConfig {
    pub cache_dir: PathBuf,
    pub manifest_path: PathBuf,
    pub image: ImageConfig,
    pub css: CssConfig,
    pub js: JsConfig,
}

#[derive(Debug, Clone)]
pub struct ImageConfig {
    pub default_quality: u8,
    pub max_width: u32,
    pub max_height: u32,
    pub formats: Vec<ImageFormat>,
    pub use_mozjpeg: bool,
    pub use_oxipng: bool,
    pub use_webp: bool,
}

#[derive(Debug, Clone)]
pub struct CssConfig {
    pub minify: bool,
    pub autoprefixer: bool,
    pub source_maps: bool,
    pub import_resolver: bool,
}

#[derive(Debug, Clone)]
pub struct JsConfig {
    pub minify: bool,
    pub source_maps: bool,
    pub module_resolver: bool,
}

#[derive(Debug, Clone, Default)]
pub struct ProcessOptions {
    pub minify: Option<bool>,
    pub resize: Option<ResizeOptions>,
    pub quality: Option<u8>,
    pub format: Option<ImageFormat>,
}
```

## Summary

Dieser Asset Processor bietet:
- **Image Optimization** - Resize, Compress, Format Conversion
- **CSS Processing** - Minification, Import Resolution, Autoprefixer
- **JS Processing** - Minification, Module Resolution
- **Asset Fingerprinting** - Cache Busting mit Hashes
- **Asset Manifest** - Mapping für Template Integration
- **EPC Integration** - Theme-aware Asset Resolution

Effiziente Asset Pipeline für optimale Performance.