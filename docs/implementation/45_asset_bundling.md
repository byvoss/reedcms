# Asset Bundling

## Overview

Asset Bundling System für ReedCMS. Module Bundling, Code Splitting und Tree Shaking für optimierte JavaScript und CSS Delivery.

## Bundle Manager Core

```rust
use std::collections::{HashMap, HashSet};
use petgraph::graph::{DiGraph, NodeIndex};
use swc::{config::Options, Compiler};

/// Asset bundle manager
pub struct BundleManager {
    config: BundleConfig,
    dependency_graph: DependencyGraph,
    compiler: Arc<Compiler>,
    cache: Arc<BundleCache>,
}

impl BundleManager {
    pub fn new(config: BundleConfig) -> Result<Self> {
        let compiler = Arc::new(Compiler::new(Default::default()));
        let cache = Arc::new(BundleCache::new(&config.cache_dir)?);
        
        Ok(Self {
            config,
            dependency_graph: DependencyGraph::new(),
            compiler,
            cache,
        })
    }
    
    /// Build bundle from entry points
    pub async fn build_bundle(
        &mut self,
        entry_points: Vec<String>,
        options: BundleOptions,
    ) -> Result<Bundle> {
        // Check cache first
        let cache_key = self.generate_cache_key(&entry_points, &options);
        if let Some(cached) = self.cache.get(&cache_key).await? {
            if !options.force_rebuild {
                return Ok(cached);
            }
        }
        
        // Build dependency graph
        self.build_dependency_graph(&entry_points).await?;
        
        // Analyze and split chunks
        let chunks = self.analyze_chunks(&options)?;
        
        // Bundle each chunk
        let mut bundle = Bundle::new();
        
        for chunk in chunks {
            let bundled_chunk = self.bundle_chunk(chunk, &options).await?;
            bundle.add_chunk(bundled_chunk);
        }
        
        // Generate manifest
        bundle.generate_manifest();
        
        // Cache result
        self.cache.set(&cache_key, &bundle).await?;
        
        Ok(bundle)
    }
    
    /// Build dependency graph
    async fn build_dependency_graph(
        &mut self,
        entry_points: &[String],
    ) -> Result<()> {
        self.dependency_graph.clear();
        
        // Process each entry point
        for entry in entry_points {
            self.process_module(entry, None).await?;
        }
        
        Ok(())
    }
    
    /// Process single module
    async fn process_module(
        &mut self,
        module_path: &str,
        parent: Option<NodeIndex>,
    ) -> Result<NodeIndex> {
        // Check if already processed
        if let Some(node) = self.dependency_graph.get_node(module_path) {
            if let Some(parent_node) = parent {
                self.dependency_graph.add_edge(parent_node, node);
            }
            return Ok(node);
        }
        
        // Read module
        let content = self.read_module(module_path).await?;
        
        // Parse imports
        let module_info = self.parse_module(&content, module_path)?;
        
        // Add to graph
        let node = self.dependency_graph.add_module(module_info.clone());
        
        if let Some(parent_node) = parent {
            self.dependency_graph.add_edge(parent_node, node);
        }
        
        // Process dependencies
        for dep in &module_info.dependencies {
            let dep_path = self.resolve_import(dep, module_path)?;
            Box::pin(self.process_module(&dep_path, Some(node))).await?;
        }
        
        Ok(node)
    }
}

/// Bundle output
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Bundle {
    pub id: String,
    pub chunks: Vec<BundledChunk>,
    pub manifest: BundleManifest,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BundledChunk {
    pub id: String,
    pub name: String,
    pub content: String,
    pub source_map: Option<String>,
    pub modules: Vec<String>,
    pub size: usize,
    pub hash: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BundleManifest {
    pub entry_points: HashMap<String, String>,
    pub chunks: HashMap<String, ChunkInfo>,
    pub assets: Vec<String>,
}
```

## Dependency Graph

```rust
/// Module dependency graph
pub struct DependencyGraph {
    graph: DiGraph<ModuleInfo, EdgeWeight>,
    module_map: HashMap<String, NodeIndex>,
}

impl DependencyGraph {
    pub fn new() -> Self {
        Self {
            graph: DiGraph::new(),
            module_map: HashMap::new(),
        }
    }
    
    /// Add module to graph
    pub fn add_module(&mut self, module: ModuleInfo) -> NodeIndex {
        let path = module.path.clone();
        let node = self.graph.add_node(module);
        self.module_map.insert(path, node);
        node
    }
    
    /// Get node by path
    pub fn get_node(&self, path: &str) -> Option<NodeIndex> {
        self.module_map.get(path).copied()
    }
    
    /// Add dependency edge
    pub fn add_edge(&mut self, from: NodeIndex, to: NodeIndex) {
        self.graph.add_edge(from, to, EdgeWeight::Static);
    }
    
    /// Find common chunks
    pub fn find_common_chunks(
        &self,
        min_shared: usize,
    ) -> Vec<CommonChunk> {
        let mut module_usage: HashMap<NodeIndex, HashSet<NodeIndex>> = HashMap::new();
        
        // Track which entry points use each module
        for entry in self.get_entry_points() {
            let reachable = self.get_reachable_modules(entry);
            for module in reachable {
                module_usage.entry(module)
                    .or_insert_with(HashSet::new)
                    .insert(entry);
            }
        }
        
        // Find modules shared by multiple entry points
        let mut common_chunks = Vec::new();
        let mut processed = HashSet::new();
        
        for (module, entries) in module_usage {
            if entries.len() >= min_shared && !processed.contains(&module) {
                // Build chunk from this module and its exclusive deps
                let chunk_modules = self.build_chunk_from(module, &module_usage);
                
                for m in &chunk_modules {
                    processed.insert(*m);
                }
                
                common_chunks.push(CommonChunk {
                    modules: chunk_modules,
                    used_by: entries.into_iter().collect(),
                });
            }
        }
        
        common_chunks
    }
    
    /// Get reachable modules from node
    fn get_reachable_modules(&self, start: NodeIndex) -> HashSet<NodeIndex> {
        use petgraph::visit::Dfs;
        
        let mut visited = HashSet::new();
        let mut dfs = Dfs::new(&self.graph, start);
        
        while let Some(node) = dfs.next(&self.graph) {
            visited.insert(node);
        }
        
        visited
    }
}

#[derive(Debug, Clone)]
pub struct ModuleInfo {
    pub path: String,
    pub content: String,
    pub dependencies: Vec<String>,
    pub exports: Vec<String>,
    pub size: usize,
    pub is_dynamic: bool,
}

#[derive(Debug, Clone)]
enum EdgeWeight {
    Static,
    Dynamic,
}

#[derive(Debug)]
struct CommonChunk {
    modules: Vec<NodeIndex>,
    used_by: Vec<NodeIndex>,
}
```

## Module Parsing

```rust
impl BundleManager {
    /// Parse module for imports/exports
    fn parse_module(
        &self,
        content: &str,
        path: &str,
    ) -> Result<ModuleInfo> {
        use swc::{
            ecmascript::{ast::*, parser::*, visit::*},
            common::{FileName, SourceMap},
        };
        
        let source_map = Arc::new(SourceMap::default());
        let source_file = source_map.new_source_file(
            FileName::Real(path.into()),
            content.to_string(),
        );
        
        // Parse module
        let mut parser = Parser::new_from(
            Syntax::Es(Default::default()),
            StringInput::from(&*source_file),
            None,
        );
        
        let module = parser.parse_module()
            .map_err(|e| ReedError::Parse(format!("Failed to parse {}: {:?}", path, e)))?;
        
        // Extract imports and exports
        let mut visitor = ModuleVisitor::new();
        module.visit_with(&mut visitor);
        
        Ok(ModuleInfo {
            path: path.to_string(),
            content: content.to_string(),
            dependencies: visitor.imports,
            exports: visitor.exports,
            size: content.len(),
            is_dynamic: visitor.has_dynamic_imports,
        })
    }
    
    /// Resolve import path
    fn resolve_import(
        &self,
        import_path: &str,
        from_module: &str,
    ) -> Result<String> {
        // Handle different import types
        if import_path.starts_with('.') {
            // Relative import
            let base_dir = PathBuf::from(from_module)
                .parent()
                .ok_or_else(|| ReedError::InvalidPath("No parent dir".into()))?;
            
            let resolved = base_dir.join(import_path);
            
            // Add extension if missing
            if resolved.extension().is_none() {
                for ext in &[".js", ".ts", ".jsx", ".tsx"] {
                    let with_ext = resolved.with_extension(&ext[1..]);
                    if with_ext.exists() {
                        return Ok(with_ext.to_string_lossy().to_string());
                    }
                }
            }
            
            Ok(resolved.to_string_lossy().to_string())
        } else if import_path.starts_with('/') {
            // Absolute import
            Ok(self.config.root_dir.join(&import_path[1..]).to_string_lossy().to_string())
        } else {
            // Node module
            self.resolve_node_module(import_path, from_module)
        }
    }
}

/// Module visitor for extracting imports/exports
struct ModuleVisitor {
    imports: Vec<String>,
    exports: Vec<String>,
    has_dynamic_imports: bool,
}

impl ModuleVisitor {
    fn new() -> Self {
        Self {
            imports: Vec::new(),
            exports: Vec::new(),
            has_dynamic_imports: false,
        }
    }
}

impl Visit for ModuleVisitor {
    fn visit_import_decl(&mut self, node: &ImportDecl) {
        self.imports.push(node.src.value.to_string());
    }
    
    fn visit_export_decl(&mut self, node: &ExportDecl) {
        match &node.decl {
            Decl::Fn(f) => self.exports.push(f.ident.sym.to_string()),
            Decl::Class(c) => self.exports.push(c.ident.sym.to_string()),
            Decl::Var(v) => {
                for decl in &v.decls {
                    if let Pat::Ident(ident) = &decl.name {
                        self.exports.push(ident.id.sym.to_string());
                    }
                }
            }
            _ => {}
        }
    }
    
    fn visit_call_expr(&mut self, node: &CallExpr) {
        if let Callee::Import(_) = &node.callee {
            self.has_dynamic_imports = true;
        }
    }
}
```

## Code Splitting

```rust
impl BundleManager {
    /// Analyze and split chunks
    fn analyze_chunks(
        &self,
        options: &BundleOptions,
    ) -> Result<Vec<Chunk>> {
        let mut chunks = Vec::new();
        
        // Create entry chunks
        let entry_points = self.dependency_graph.get_entry_points();
        for entry in entry_points {
            chunks.push(Chunk {
                id: format!("entry-{}", chunks.len()),
                name: self.dependency_graph.graph[entry].path.clone(),
                modules: vec![entry],
                chunk_type: ChunkType::Entry,
            });
        }
        
        // Find common chunks
        if options.split_chunks {
            let common_chunks = self.dependency_graph
                .find_common_chunks(options.min_chunk_size);
            
            for (i, common) in common_chunks.into_iter().enumerate() {
                chunks.push(Chunk {
                    id: format!("common-{}", i),
                    name: format!("common-{}", i),
                    modules: common.modules,
                    chunk_type: ChunkType::Common,
                });
            }
        }
        
        // Handle dynamic imports
        if options.code_splitting {
            let dynamic_chunks = self.find_dynamic_chunks();
            
            for (i, dynamic) in dynamic_chunks.into_iter().enumerate() {
                chunks.push(Chunk {
                    id: format!("dynamic-{}", i),
                    name: format!("dynamic-{}", i),
                    modules: dynamic,
                    chunk_type: ChunkType::Dynamic,
                });
            }
        }
        
        Ok(chunks)
    }
    
    /// Bundle single chunk
    async fn bundle_chunk(
        &self,
        chunk: Chunk,
        options: &BundleOptions,
    ) -> Result<BundledChunk> {
        let mut bundled_content = String::new();
        let mut source_map = if options.source_maps {
            Some(SourceMapBuilder::new())
        } else {
            None
        };
        
        // Add runtime if needed
        if chunk.chunk_type == ChunkType::Entry {
            bundled_content.push_str(&self.get_runtime_code());
        }
        
        // Bundle modules in dependency order
        let ordered_modules = self.order_modules(&chunk.modules)?;
        
        for module_idx in ordered_modules {
            let module = &self.dependency_graph.graph[module_idx];
            
            // Transform module
            let transformed = self.transform_module(module, options).await?;
            
            // Wrap in module definition
            let wrapped = self.wrap_module(&module.path, &transformed.code);
            bundled_content.push_str(&wrapped);
            bundled_content.push('\n');
            
            // Add to source map
            if let Some(ref mut sm) = source_map {
                sm.add_module(&module.path, &transformed.source_map);
            }
        }
        
        // Minify if enabled
        let final_content = if options.minify {
            self.minify_js(&bundled_content)?
        } else {
            bundled_content
        };
        
        // Generate hash
        let hash = self.generate_content_hash(&final_content);
        
        Ok(BundledChunk {
            id: chunk.id,
            name: format!("{}-{}", chunk.name, &hash[..8]),
            content: final_content.clone(),
            source_map: source_map.map(|sm| sm.build()),
            modules: chunk.modules.into_iter()
                .map(|idx| self.dependency_graph.graph[idx].path.clone())
                .collect(),
            size: final_content.len(),
            hash,
        })
    }
    
    /// Wrap module in definition
    fn wrap_module(&self, path: &str, code: &str) -> String {
        format!(
            r#"__modules__[{}] = function(module, exports, require) {{
{}
}};
"#,
            serde_json::to_string(path).unwrap(),
            code
        )
    }
    
    /// Get runtime code
    fn get_runtime_code(&self) -> &'static str {
        r#"
const __modules__ = {};
const __cache__ = {};

function require(id) {
    if (__cache__[id]) {
        return __cache__[id].exports;
    }
    
    const module = { exports: {} };
    __cache__[id] = module;
    
    if (__modules__[id]) {
        __modules__[id].call(module.exports, module, module.exports, require);
    } else {
        throw new Error(`Module ${id} not found`);
    }
    
    return module.exports;
}
"#
    }
}

#[derive(Debug)]
struct Chunk {
    id: String,
    name: String,
    modules: Vec<NodeIndex>,
    chunk_type: ChunkType,
}

#[derive(Debug, PartialEq)]
enum ChunkType {
    Entry,
    Common,
    Dynamic,
}
```

## CSS Bundling

```rust
/// CSS bundle manager
pub struct CssBundler {
    config: BundleConfig,
    processor: CssProcessor,
}

impl CssBundler {
    /// Bundle CSS files
    pub async fn bundle_css(
        &self,
        entry_points: Vec<String>,
        options: &BundleOptions,
    ) -> Result<CssBundle> {
        let mut bundled = String::new();
        let mut processed_files = HashSet::new();
        
        // Process each entry point
        for entry in entry_points {
            self.process_css_file(&entry, &mut bundled, &mut processed_files).await?;
        }
        
        // Post-process
        let final_css = if options.minify {
            self.processor.minify(&bundled)?
        } else {
            bundled
        };
        
        // Extract critical CSS if enabled
        let critical_css = if options.extract_critical {
            Some(self.extract_critical_css(&final_css)?)
        } else {
            None
        };
        
        Ok(CssBundle {
            content: final_css,
            critical: critical_css,
            hash: self.generate_content_hash(&final_css),
        })
    }
    
    /// Process CSS file and imports
    async fn process_css_file(
        &self,
        path: &str,
        output: &mut String,
        processed: &mut HashSet<String>,
    ) -> Result<()> {
        if processed.contains(path) {
            return Ok(());
        }
        
        processed.insert(path.to_string());
        
        let content = tokio::fs::read_to_string(path).await?;
        
        // Process imports first
        let import_regex = regex::Regex::new(r#"@import\s+["']([^"']+)["'];"#)?;
        let base_dir = PathBuf::from(path).parent().unwrap_or(Path::new(""));
        
        for cap in import_regex.captures_iter(&content) {
            let import_path = &cap[1];
            let full_path = base_dir.join(import_path);
            
            if full_path.exists() {
                Box::pin(self.process_css_file(
                    &full_path.to_string_lossy(),
                    output,
                    processed,
                )).await?;
            }
        }
        
        // Add file content (without imports)
        let content_without_imports = import_regex.replace_all(&content, "");
        output.push_str(&content_without_imports);
        output.push_str("\n");
        
        Ok(())
    }
    
    /// Extract critical CSS
    fn extract_critical_css(&self, css: &str) -> Result<String> {
        // Simple critical CSS extraction
        // In production, use tools like critical or penthouse
        let mut critical = String::new();
        
        // Extract reset/normalize rules
        let reset_regex = regex::Regex::new(r"(\*|html|body)\s*\{[^}]+\}")?;
        for mat in reset_regex.find_iter(css) {
            critical.push_str(mat.as_str());
            critical.push('\n');
        }
        
        // Extract above-the-fold selectors
        let critical_selectors = [
            "header", "nav", ".hero", "h1", "h2", ".container",
        ];
        
        for selector in &critical_selectors {
            let regex = regex::Regex::new(&format!(r"{}\s*\{{[^}}]+\}}", selector))?;
            for mat in regex.find_iter(css) {
                critical.push_str(mat.as_str());
                critical.push('\n');
            }
        }
        
        Ok(critical)
    }
}

#[derive(Debug, Clone)]
pub struct CssBundle {
    pub content: String,
    pub critical: Option<String>,
    pub hash: String,
}
```

## Bundle Configuration

```rust
#[derive(Debug, Clone)]
pub struct BundleConfig {
    pub root_dir: PathBuf,
    pub output_dir: PathBuf,
    pub cache_dir: PathBuf,
    pub public_path: String,
    pub chunk_naming: ChunkNaming,
    pub externals: Vec<String>,
}

#[derive(Debug, Clone)]
pub struct BundleOptions {
    pub minify: bool,
    pub source_maps: bool,
    pub split_chunks: bool,
    pub code_splitting: bool,
    pub tree_shaking: bool,
    pub min_chunk_size: usize,
    pub extract_critical: bool,
    pub force_rebuild: bool,
}

#[derive(Debug, Clone)]
pub enum ChunkNaming {
    Hash,
    Name,
    NameHash,
}

impl Default for BundleOptions {
    fn default() -> Self {
        Self {
            minify: true,
            source_maps: true,
            split_chunks: true,
            code_splitting: true,
            tree_shaking: true,
            min_chunk_size: 2,
            extract_critical: false,
            force_rebuild: false,
        }
    }
}
```

## Summary

Dieses Asset Bundling System bietet:
- **Module Bundling** - ES6 Module Support mit Dependency Graph
- **Code Splitting** - Automatic Chunk Splitting
- **Tree Shaking** - Unused Code Elimination
- **CSS Bundling** - Import Resolution und Critical CSS
- **Source Maps** - Für Development Debugging
- **Caching** - Build Cache für schnelle Rebuilds

Modernes Bundling für optimale Load Performance.