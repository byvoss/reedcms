# Testing Framework

## Overview

Testing Framework Implementation für ReedCMS. Comprehensive Testing mit Unit, Integration und E2E Tests.

## Test Suite Manager

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

/// Main test suite management
pub struct TestSuiteManager {
    suites: Arc<RwLock<HashMap<String, TestSuite>>>,
    runner: Arc<TestRunner>,
    reporter: Arc<TestReporter>,
    config: TestConfig,
}

impl TestSuiteManager {
    pub fn new(config: TestConfig) -> Self {
        Self {
            suites: Arc::new(RwLock::new(HashMap::new())),
            runner: Arc::new(TestRunner::new(config.runner.clone())),
            reporter: Arc::new(TestReporter::new(config.reporter.clone())),
            config,
        }
    }
    
    /// Register test suite
    pub async fn register_suite(&self, suite: TestSuite) -> Result<()> {
        self.suites.write().await.insert(suite.name.clone(), suite);
        Ok(())
    }
    
    /// Run all tests
    pub async fn run_all(&self) -> Result<TestReport> {
        let suites = self.suites.read().await;
        let mut results = Vec::new();
        
        for suite in suites.values() {
            let result = self.runner.run_suite(suite).await?;
            results.push(result);
        }
        
        let report = self.reporter.generate_report(&results).await?;
        Ok(report)
    }
    
    /// Run specific test suite
    pub async fn run_suite(&self, suite_name: &str) -> Result<SuiteResult> {
        let suites = self.suites.read().await;
        let suite = suites.get(suite_name)
            .ok_or_else(|| TestError::SuiteNotFound(suite_name.to_string()))?;
        
        self.runner.run_suite(suite).await
    }
    
    /// Run tests matching pattern
    pub async fn run_pattern(&self, pattern: &str) -> Result<TestReport> {
        let regex = regex::Regex::new(pattern)?;
        let suites = self.suites.read().await;
        let mut results = Vec::new();
        
        for suite in suites.values() {
            let filtered_suite = self.filter_suite(suite, &regex)?;
            if !filtered_suite.tests.is_empty() {
                let result = self.runner.run_suite(&filtered_suite).await?;
                results.push(result);
            }
        }
        
        let report = self.reporter.generate_report(&results).await?;
        Ok(report)
    }
}

/// Test suite definition
#[derive(Debug, Clone)]
pub struct TestSuite {
    pub name: String,
    pub tests: Vec<TestCase>,
    pub setup: Option<SetupFn>,
    pub teardown: Option<TeardownFn>,
    pub config: SuiteConfig,
}

/// Test case
#[derive(Debug, Clone)]
pub struct TestCase {
    pub name: String,
    pub test_fn: TestFn,
    pub timeout: Duration,
    pub tags: Vec<String>,
    pub skip: Option<String>,
}

/// Test function types
type TestFn = Arc<dyn Fn() -> BoxFuture<'static, Result<()>> + Send + Sync>;
type SetupFn = Arc<dyn Fn() -> BoxFuture<'static, Result<TestContext>> + Send + Sync>;
type TeardownFn = Arc<dyn Fn(TestContext) -> BoxFuture<'static, Result<()>> + Send + Sync>;

/// Test context
#[derive(Clone)]
pub struct TestContext {
    pub fixtures: Arc<RwLock<HashMap<String, Box<dyn Any + Send + Sync>>>>,
    pub temp_dir: PathBuf,
    pub database: Option<TestDatabase>,
    pub server: Option<TestServer>,
}

impl TestContext {
    /// Get fixture
    pub async fn fixture<T: 'static + Send + Sync + Clone>(&self, name: &str) -> Option<T> {
        self.fixtures.read().await
            .get(name)
            .and_then(|f| f.downcast_ref::<T>())
            .cloned()
    }
    
    /// Set fixture
    pub async fn set_fixture<T: 'static + Send + Sync>(&self, name: String, value: T) {
        self.fixtures.write().await
            .insert(name, Box::new(value));
    }
}
```

## Test Runner

```rust
/// Test execution engine
pub struct TestRunner {
    executor: Arc<TestExecutor>,
    hooks: Arc<TestHooks>,
    config: RunnerConfig,
}

impl TestRunner {
    /// Run test suite
    pub async fn run_suite(&self, suite: &TestSuite) -> Result<SuiteResult> {
        let start = Instant::now();
        let mut results = Vec::new();
        
        // Setup
        let context = if let Some(setup) = &suite.setup {
            self.hooks.before_suite_setup(suite).await?;
            let ctx = setup().await?;
            self.hooks.after_suite_setup(suite, &ctx).await?;
            Some(ctx)
        } else {
            None
        };
        
        // Run tests
        for test in &suite.tests {
            if test.skip.is_some() {
                results.push(TestResult::skipped(test, test.skip.as_ref().unwrap()));
                continue;
            }
            
            let result = self.run_test(test, context.as_ref()).await;
            results.push(result);
        }
        
        // Teardown
        if let (Some(teardown), Some(ctx)) = (&suite.teardown, context) {
            self.hooks.before_suite_teardown(suite).await?;
            teardown(ctx).await?;
            self.hooks.after_suite_teardown(suite).await?;
        }
        
        Ok(SuiteResult {
            suite_name: suite.name.clone(),
            test_results: results,
            duration: start.elapsed(),
        })
    }
    
    /// Run single test
    async fn run_test(
        &self,
        test: &TestCase,
        context: Option<&TestContext>,
    ) -> TestResult {
        let start = Instant::now();
        
        // Before test
        if let Err(e) = self.hooks.before_test(test).await {
            return TestResult::failed(test, e, start.elapsed());
        }
        
        // Execute test with timeout
        let result = match timeout(test.timeout, (test.test_fn)()).await {
            Ok(Ok(())) => TestResult::passed(test, start.elapsed()),
            Ok(Err(e)) => TestResult::failed(test, e, start.elapsed()),
            Err(_) => TestResult::timeout(test, test.timeout),
        };
        
        // After test
        let _ = self.hooks.after_test(test, &result).await;
        
        result
    }
}

/// Test result
#[derive(Debug, Clone)]
pub struct TestResult {
    pub test_name: String,
    pub status: TestStatus,
    pub duration: Duration,
    pub error: Option<String>,
    pub stdout: Vec<String>,
    pub stderr: Vec<String>,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum TestStatus {
    Passed,
    Failed,
    Skipped,
    Timeout,
}

impl TestResult {
    fn passed(test: &TestCase, duration: Duration) -> Self {
        Self {
            test_name: test.name.clone(),
            status: TestStatus::Passed,
            duration,
            error: None,
            stdout: Vec::new(),
            stderr: Vec::new(),
        }
    }
    
    fn failed(test: &TestCase, error: Error, duration: Duration) -> Self {
        Self {
            test_name: test.name.clone(),
            status: TestStatus::Failed,
            duration,
            error: Some(error.to_string()),
            stdout: Vec::new(),
            stderr: Vec::new(),
        }
    }
}
```

## Test Macros

```rust
/// Test definition macros
#[macro_export]
macro_rules! test_suite {
    ($name:expr, $($test:tt)*) => {
        pub fn suite() -> TestSuite {
            TestSuite {
                name: $name.to_string(),
                tests: vec![$($test)*],
                setup: None,
                teardown: None,
                config: Default::default(),
            }
        }
    };
}

#[macro_export]
macro_rules! test_case {
    ($name:expr, $test_fn:expr) => {
        TestCase {
            name: $name.to_string(),
            test_fn: Arc::new(|| Box::pin($test_fn())),
            timeout: Duration::from_secs(30),
            tags: Vec::new(),
            skip: None,
        }
    };
    
    ($name:expr, $test_fn:expr, timeout = $timeout:expr) => {
        TestCase {
            name: $name.to_string(),
            test_fn: Arc::new(|| Box::pin($test_fn())),
            timeout: $timeout,
            tags: Vec::new(),
            skip: None,
        }
    };
}

#[macro_export]
macro_rules! assert_matches {
    ($left:expr, $pattern:pat) => {
        match $left {
            $pattern => {},
            ref left_val => {
                panic!(
                    "assertion failed: `(left matches pattern)`\n  left: `{:?}`\n pattern: `{}`",
                    left_val,
                    stringify!($pattern)
                )
            }
        }
    };
    
    ($left:expr, $pattern:pat, $msg:expr) => {
        match $left {
            $pattern => {},
            ref left_val => {
                panic!(
                    "assertion failed: `(left matches pattern)`\n  left: `{:?}`\n pattern: `{}`\n message: {}",
                    left_val,
                    stringify!($pattern),
                    $msg
                )
            }
        }
    };
}

#[macro_export]
macro_rules! assert_eventually {
    ($condition:expr, timeout = $timeout:expr) => {
        let start = std::time::Instant::now();
        loop {
            if $condition {
                break;
            }
            if start.elapsed() > $timeout {
                panic!("assertion failed: condition not met within {:?}", $timeout);
            }
            tokio::time::sleep(Duration::from_millis(10)).await;
        }
    };
}
```

## Integration Test Helpers

```rust
/// Test database helper
pub struct TestDatabase {
    pool: Arc<PgPool>,
    database_name: String,
    cleanup_on_drop: bool,
}

impl TestDatabase {
    /// Create test database
    pub async fn create() -> Result<Self> {
        let database_name = format!("test_{}_{}", 
            chrono::Utc::now().timestamp(),
            rand::random::<u32>()
        );
        
        // Create database
        let admin_pool = PgPoolOptions::new()
            .connect(&env::var("TEST_DATABASE_URL")?)
            .await?;
        
        sqlx::query(&format!("CREATE DATABASE {}", database_name))
            .execute(&admin_pool)
            .await?;
        
        // Connect to test database
        let test_url = format!("{}/{}", 
            env::var("TEST_DATABASE_URL")?,
            database_name
        );
        
        let pool = PgPoolOptions::new()
            .connect(&test_url)
            .await?;
        
        // Run migrations
        sqlx::migrate!("./migrations")
            .run(&pool)
            .await?;
        
        Ok(Self {
            pool: Arc::new(pool),
            database_name,
            cleanup_on_drop: true,
        })
    }
    
    /// Get connection pool
    pub fn pool(&self) -> Arc<PgPool> {
        self.pool.clone()
    }
    
    /// Seed test data
    pub async fn seed<F, Fut>(&self, seeder: F) -> Result<()>
    where
        F: FnOnce(Arc<PgPool>) -> Fut,
        Fut: Future<Output = Result<()>>,
    {
        seeder(self.pool.clone()).await
    }
}

impl Drop for TestDatabase {
    fn drop(&mut self) {
        if self.cleanup_on_drop {
            let database_name = self.database_name.clone();
            tokio::spawn(async move {
                // Drop test database
                if let Ok(admin_pool) = PgPoolOptions::new()
                    .connect(&env::var("TEST_DATABASE_URL").unwrap())
                    .await
                {
                    let _ = sqlx::query(&format!("DROP DATABASE IF EXISTS {}", database_name))
                        .execute(&admin_pool)
                        .await;
                }
            });
        }
    }
}

/// Test server helper
pub struct TestServer {
    addr: SocketAddr,
    shutdown_tx: Option<oneshot::Sender<()>>,
}

impl TestServer {
    /// Start test server
    pub async fn start(app: Router) -> Result<Self> {
        let listener = TcpListener::bind("127.0.0.1:0").await?;
        let addr = listener.local_addr()?;
        
        let (shutdown_tx, shutdown_rx) = oneshot::channel();
        
        tokio::spawn(async move {
            axum::serve(listener, app)
                .with_graceful_shutdown(async {
                    shutdown_rx.await.ok();
                })
                .await
                .unwrap();
        });
        
        // Wait for server to start
        tokio::time::sleep(Duration::from_millis(100)).await;
        
        Ok(Self {
            addr,
            shutdown_tx: Some(shutdown_tx),
        })
    }
    
    /// Get server URL
    pub fn url(&self) -> String {
        format!("http://{}", self.addr)
    }
    
    /// Create test client
    pub fn client(&self) -> TestClient {
        TestClient::new(self.url())
    }
}

impl Drop for TestServer {
    fn drop(&mut self) {
        if let Some(tx) = self.shutdown_tx.take() {
            let _ = tx.send(());
        }
    }
}
```

## Test Client

```rust
/// HTTP test client
pub struct TestClient {
    client: reqwest::Client,
    base_url: String,
    default_headers: HeaderMap,
}

impl TestClient {
    pub fn new(base_url: String) -> Self {
        Self {
            client: reqwest::Client::new(),
            base_url,
            default_headers: HeaderMap::new(),
        }
    }
    
    /// Set authentication
    pub fn with_auth(mut self, token: &str) -> Self {
        self.default_headers.insert(
            "Authorization",
            HeaderValue::from_str(&format!("Bearer {}", token)).unwrap(),
        );
        self
    }
    
    /// GET request
    pub async fn get(&self, path: &str) -> TestResponse {
        self.request(Method::GET, path, None).await
    }
    
    /// POST request
    pub async fn post<T: Serialize>(&self, path: &str, body: &T) -> TestResponse {
        self.request(Method::POST, path, Some(body)).await
    }
    
    /// Generic request
    async fn request<T: Serialize>(
        &self,
        method: Method,
        path: &str,
        body: Option<&T>,
    ) -> TestResponse {
        let url = format!("{}{}", self.base_url, path);
        let mut request = self.client.request(method, &url);
        
        // Add default headers
        for (key, value) in &self.default_headers {
            request = request.header(key, value);
        }
        
        // Add body
        if let Some(body) = body {
            request = request.json(body);
        }
        
        let response = request.send().await.unwrap();
        
        TestResponse::from_response(response).await
    }
}

/// Test response wrapper
pub struct TestResponse {
    pub status: StatusCode,
    pub headers: HeaderMap,
    pub body: Bytes,
}

impl TestResponse {
    async fn from_response(response: reqwest::Response) -> Self {
        Self {
            status: response.status(),
            headers: response.headers().clone(),
            body: response.bytes().await.unwrap(),
        }
    }
    
    /// Assert status code
    pub fn assert_status(&self, expected: StatusCode) -> &Self {
        assert_eq!(self.status, expected, "Unexpected status code");
        self
    }
    
    /// Get JSON body
    pub fn json<T: DeserializeOwned>(&self) -> T {
        serde_json::from_slice(&self.body).unwrap()
    }
    
    /// Get text body
    pub fn text(&self) -> String {
        String::from_utf8(self.body.to_vec()).unwrap()
    }
}
```

## Mock Helpers

```rust
/// Mock service builder
pub struct MockBuilder<T> {
    expectations: Vec<Expectation<T>>,
}

impl<T> MockBuilder<T> {
    pub fn new() -> Self {
        Self {
            expectations: Vec::new(),
        }
    }
    
    /// Expect method call
    pub fn expect<F, R>(mut self, matcher: F) -> Self
    where
        F: Fn(&T) -> R + Send + Sync + 'static,
        R: Future<Output = Result<()>> + Send + 'static,
    {
        self.expectations.push(Expectation {
            matcher: Box::new(matcher),
            times: Times::Any,
        });
        self
    }
    
    /// Build mock
    pub fn build(self) -> Mock<T> {
        Mock {
            expectations: Arc::new(Mutex::new(self.expectations)),
            calls: Arc::new(Mutex::new(Vec::new())),
        }
    }
}

/// Mock service
pub struct Mock<T> {
    expectations: Arc<Mutex<Vec<Expectation<T>>>>,
    calls: Arc<Mutex<Vec<T>>>,
}

impl<T: Clone> Mock<T> {
    /// Record call
    pub async fn call(&self, value: T) -> Result<()> {
        self.calls.lock().await.push(value.clone());
        
        let expectations = self.expectations.lock().await;
        for expectation in expectations.iter() {
            if (expectation.matcher)(&value).await.is_ok() {
                return Ok(());
            }
        }
        
        Err(TestError::UnexpectedCall)
    }
    
    /// Verify expectations
    pub async fn verify(&self) -> Result<()> {
        let calls = self.calls.lock().await;
        let expectations = self.expectations.lock().await;
        
        // Check all expectations were met
        for expectation in expectations.iter() {
            let matching_calls = calls.iter()
                .filter(|call| (expectation.matcher)(call).await.is_ok())
                .count();
            
            expectation.times.verify(matching_calls)?;
        }
        
        Ok(())
    }
}

/// Expectation
struct Expectation<T> {
    matcher: Box<dyn Fn(&T) -> BoxFuture<'static, Result<()>> + Send + Sync>,
    times: Times,
}

/// Expected call count
enum Times {
    Exactly(usize),
    AtLeast(usize),
    AtMost(usize),
    Any,
}

impl Times {
    fn verify(&self, actual: usize) -> Result<()> {
        match self {
            Times::Exactly(n) => {
                if actual != *n {
                    return Err(TestError::WrongCallCount(*n, actual));
                }
            }
            Times::AtLeast(n) => {
                if actual < *n {
                    return Err(TestError::TooFewCalls(*n, actual));
                }
            }
            Times::AtMost(n) => {
                if actual > *n {
                    return Err(TestError::TooManyCalls(*n, actual));
                }
            }
            Times::Any => {}
        }
        Ok(())
    }
}
```

## E2E Test Framework

```rust
/// End-to-end test framework
pub struct E2ETestFramework {
    browser: Arc<Browser>,
    server: TestServer,
    database: TestDatabase,
}

impl E2ETestFramework {
    /// Setup E2E test environment
    pub async fn setup() -> Result<Self> {
        // Start test database
        let database = TestDatabase::create().await?;
        
        // Build app with test database
        let app = build_app(AppConfig {
            database_url: database.connection_string(),
            ..Default::default()
        })?;
        
        // Start test server
        let server = TestServer::start(app).await?;
        
        // Launch browser
        let browser = Browser::launch(BrowserConfig {
            headless: true,
            ..Default::default()
        }).await?;
        
        Ok(Self {
            browser: Arc::new(browser),
            server,
            database,
        })
    }
    
    /// Create new page
    pub async fn new_page(&self) -> Result<Page> {
        let page = self.browser.new_page().await?;
        page.set_default_timeout(Duration::from_secs(30)).await?;
        Ok(page)
    }
    
    /// Login as user
    pub async fn login(&self, page: &Page, email: &str, password: &str) -> Result<()> {
        page.goto(&format!("{}/login", self.server.url())).await?;
        page.fill("input[name=email]", email).await?;
        page.fill("input[name=password]", password).await?;
        page.click("button[type=submit]").await?;
        page.wait_for_navigation().await?;
        Ok(())
    }
}

/// Page object pattern
pub struct PageObject {
    page: Page,
    selectors: HashMap<String, String>,
}

impl PageObject {
    /// Click element
    pub async fn click(&self, element: &str) -> Result<()> {
        let selector = self.selectors.get(element)
            .ok_or_else(|| TestError::ElementNotFound(element.to_string()))?;
        
        self.page.click(selector).await?;
        Ok(())
    }
    
    /// Fill input
    pub async fn fill(&self, element: &str, value: &str) -> Result<()> {
        let selector = self.selectors.get(element)
            .ok_or_else(|| TestError::ElementNotFound(element.to_string()))?;
        
        self.page.fill(selector, value).await?;
        Ok(())
    }
    
    /// Assert element visible
    pub async fn assert_visible(&self, element: &str) -> Result<()> {
        let selector = self.selectors.get(element)
            .ok_or_else(|| TestError::ElementNotFound(element.to_string()))?;
        
        self.page.wait_for_selector(selector).await?;
        Ok(())
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TestConfig {
    pub runner: RunnerConfig,
    pub reporter: ReporterConfig,
    pub coverage: CoverageConfig,
    pub fixtures: FixtureConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RunnerConfig {
    pub parallel: bool,
    pub max_threads: usize,
    pub timeout: Duration,
    pub retry_failed: bool,
    pub fail_fast: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReporterConfig {
    pub format: ReportFormat,
    pub output_path: Option<PathBuf>,
    pub verbose: bool,
    pub show_output: bool,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ReportFormat {
    Pretty,
    Json,
    Junit,
    Html,
}

impl Default for TestConfig {
    fn default() -> Self {
        Self {
            runner: RunnerConfig {
                parallel: true,
                max_threads: num_cpus::get(),
                timeout: Duration::from_secs(300),
                retry_failed: false,
                fail_fast: false,
            },
            reporter: ReporterConfig {
                format: ReportFormat::Pretty,
                output_path: None,
                verbose: false,
                show_output: false,
            },
            coverage: CoverageConfig {
                enabled: true,
                min_coverage: 80.0,
                exclude: vec!["tests/**", "benches/**"],
            },
            fixtures: FixtureConfig {
                path: PathBuf::from("tests/fixtures"),
                cleanup: true,
            },
        }
    }
}
```

## Summary

Diese Testing Framework Implementation bietet:
- **Test Suite Management** - Organized Test Execution
- **Multiple Test Types** - Unit, Integration, E2E
- **Test Helpers** - Database, Server, Client Utilities
- **Mock Framework** - Expectation-based Mocking
- **Assertions** - Extended Assertion Macros
- **Reporting** - Multiple Output Formats

Comprehensive Testing Infrastructure für Quality Assurance.