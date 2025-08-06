# Authentication

## Overview

Authentication Implementation für ReedCMS. Multi-Provider Authentication mit Session Management und Token-based Auth.

## Authentication Core

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use argon2::{Argon2, PasswordHash, PasswordHasher, PasswordVerifier};

/// Main authentication service
pub struct AuthenticationService {
    user_store: Arc<UserStore>,
    session_manager: Arc<SessionManager>,
    token_service: Arc<TokenService>,
    providers: Arc<RwLock<HashMap<String, Box<dyn AuthProvider>>>>,
    config: AuthConfig,
}

impl AuthenticationService {
    pub fn new(
        user_store: Arc<UserStore>,
        config: AuthConfig,
    ) -> Result<Self> {
        let session_manager = Arc::new(SessionManager::new(&config.session)?);
        let token_service = Arc::new(TokenService::new(&config.token)?);
        
        let mut service = Self {
            user_store,
            session_manager,
            token_service,
            providers: Arc::new(RwLock::new(HashMap::new())),
            config,
        };
        
        // Register default providers
        service.register_default_providers()?;
        
        Ok(service)
    }
    
    /// Authenticate user with credentials
    pub async fn authenticate(
        &self,
        credentials: AuthCredentials,
    ) -> Result<AuthResult> {
        match credentials {
            AuthCredentials::Password { username, password } => {
                self.authenticate_password(&username, &password).await
            }
            AuthCredentials::Token { token } => {
                self.authenticate_token(&token).await
            }
            AuthCredentials::OAuth { provider, code } => {
                self.authenticate_oauth(&provider, &code).await
            }
            AuthCredentials::ApiKey { key } => {
                self.authenticate_api_key(&key).await
            }
        }
    }
    
    /// Password authentication
    async fn authenticate_password(
        &self,
        username: &str,
        password: &str,
    ) -> Result<AuthResult> {
        // Get user by username or email
        let user = self.user_store
            .find_by_username_or_email(username)
            .await?
            .ok_or_else(|| AuthError::InvalidCredentials)?;
        
        // Check if user is active
        if !user.is_active {
            return Err(AuthError::UserInactive);
        }
        
        // Verify password
        let parsed_hash = PasswordHash::new(&user.password_hash)
            .map_err(|_| AuthError::Internal("Invalid password hash".into()))?;
        
        Argon2::default()
            .verify_password(password.as_bytes(), &parsed_hash)
            .map_err(|_| AuthError::InvalidCredentials)?;
        
        // Check if password needs rehashing
        if self.needs_rehash(&parsed_hash) {
            self.rehash_password(&user.id, password).await?;
        }
        
        // Update last login
        self.user_store.update_last_login(&user.id).await?;
        
        // Create session
        let session = self.session_manager
            .create_session(&user.id, SessionType::Web)
            .await?;
        
        Ok(AuthResult {
            user_id: user.id,
            session_id: Some(session.id),
            token: Some(session.token),
            expires_at: session.expires_at,
            permissions: user.permissions,
        })
    }
    
    /// Token authentication
    async fn authenticate_token(&self, token: &str) -> Result<AuthResult> {
        // Validate token
        let claims = self.token_service.verify_token(token)?;
        
        // Get user
        let user = self.user_store
            .get_by_id(&claims.sub)
            .await?
            .ok_or_else(|| AuthError::InvalidToken)?;
        
        if !user.is_active {
            return Err(AuthError::UserInactive);
        }
        
        Ok(AuthResult {
            user_id: user.id,
            session_id: None,
            token: Some(token.to_string()),
            expires_at: claims.exp,
            permissions: user.permissions,
        })
    }
    
    /// Register authentication provider
    pub async fn register_provider(
        &self,
        name: String,
        provider: Box<dyn AuthProvider>,
    ) -> Result<()> {
        self.providers.write().await.insert(name, provider);
        Ok(())
    }
}

/// Authentication credentials
#[derive(Debug, Clone)]
pub enum AuthCredentials {
    Password {
        username: String,
        password: String,
    },
    Token {
        token: String,
    },
    OAuth {
        provider: String,
        code: String,
    },
    ApiKey {
        key: String,
    },
}

/// Authentication result
#[derive(Debug, Clone)]
pub struct AuthResult {
    pub user_id: Uuid,
    pub session_id: Option<Uuid>,
    pub token: Option<String>,
    pub expires_at: chrono::DateTime<chrono::Utc>,
    pub permissions: Vec<String>,
}

/// Authentication error
#[derive(Debug, thiserror::Error)]
pub enum AuthError {
    #[error("Invalid credentials")]
    InvalidCredentials,
    
    #[error("Invalid token")]
    InvalidToken,
    
    #[error("Token expired")]
    TokenExpired,
    
    #[error("User inactive")]
    UserInactive,
    
    #[error("User locked")]
    UserLocked,
    
    #[error("Provider not found: {0}")]
    ProviderNotFound(String),
    
    #[error("Internal error: {0}")]
    Internal(String),
}
```

## Session Management

```rust
use redis::AsyncCommands;

/// Session manager
pub struct SessionManager {
    redis: Arc<RedisConnection>,
    config: SessionConfig,
}

impl SessionManager {
    /// Create new session
    pub async fn create_session(
        &self,
        user_id: &Uuid,
        session_type: SessionType,
    ) -> Result<Session> {
        let session = Session {
            id: Uuid::new_v4(),
            user_id: *user_id,
            token: self.generate_session_token(),
            session_type,
            created_at: chrono::Utc::now(),
            last_activity: chrono::Utc::now(),
            expires_at: chrono::Utc::now() + self.config.ttl,
            ip_address: None,
            user_agent: None,
            data: HashMap::new(),
        };
        
        // Store in Redis
        let key = format!("session:{}", session.id);
        let value = serde_json::to_string(&session)?;
        
        self.redis
            .set_ex(&key, value, self.config.ttl.num_seconds() as usize)
            .await?;
        
        // Add to user's session list
        let user_sessions_key = format!("user:{}:sessions", user_id);
        self.redis
            .sadd(&user_sessions_key, session.id.to_string())
            .await?;
        
        Ok(session)
    }
    
    /// Get session by token
    pub async fn get_session(&self, token: &str) -> Result<Option<Session>> {
        // Find session by token
        let pattern = "session:*";
        let keys: Vec<String> = self.redis.keys(pattern).await?;
        
        for key in keys {
            let value: String = self.redis.get(&key).await?;
            if let Ok(session) = serde_json::from_str::<Session>(&value) {
                if session.token == token {
                    // Check expiration
                    if session.expires_at < chrono::Utc::now() {
                        self.destroy_session(&session.id).await?;
                        return Ok(None);
                    }
                    
                    // Update last activity
                    self.touch_session(&session.id).await?;
                    
                    return Ok(Some(session));
                }
            }
        }
        
        Ok(None)
    }
    
    /// Update session activity
    pub async fn touch_session(&self, session_id: &Uuid) -> Result<()> {
        let key = format!("session:{}", session_id);
        
        if let Some(value) = self.redis.get::<_, Option<String>>(&key).await? {
            if let Ok(mut session) = serde_json::from_str::<Session>(&value) {
                session.last_activity = chrono::Utc::now();
                
                let updated_value = serde_json::to_string(&session)?;
                let ttl = (session.expires_at - chrono::Utc::now())
                    .num_seconds()
                    .max(0) as usize;
                
                self.redis.set_ex(&key, updated_value, ttl).await?;
            }
        }
        
        Ok(())
    }
    
    /// Destroy session
    pub async fn destroy_session(&self, session_id: &Uuid) -> Result<()> {
        let key = format!("session:{}", session_id);
        
        // Get session to find user
        if let Some(value) = self.redis.get::<_, Option<String>>(&key).await? {
            if let Ok(session) = serde_json::from_str::<Session>(&value) {
                // Remove from user's session list
                let user_sessions_key = format!("user:{}:sessions", session.user_id);
                self.redis
                    .srem(&user_sessions_key, session_id.to_string())
                    .await?;
            }
        }
        
        // Delete session
        self.redis.del(&key).await?;
        
        Ok(())
    }
    
    /// Get all sessions for user
    pub async fn get_user_sessions(&self, user_id: &Uuid) -> Result<Vec<Session>> {
        let user_sessions_key = format!("user:{}:sessions", user_id);
        let session_ids: Vec<String> = self.redis.smembers(&user_sessions_key).await?;
        
        let mut sessions = Vec::new();
        
        for session_id in session_ids {
            let key = format!("session:{}", session_id);
            if let Some(value) = self.redis.get::<_, Option<String>>(&key).await? {
                if let Ok(session) = serde_json::from_str::<Session>(&value) {
                    sessions.push(session);
                }
            }
        }
        
        Ok(sessions)
    }
    
    /// Generate secure session token
    fn generate_session_token(&self) -> String {
        use rand::Rng;
        let token_bytes: Vec<u8> = (0..32)
            .map(|_| rand::thread_rng().gen())
            .collect();
        
        base64::encode_config(&token_bytes, base64::URL_SAFE_NO_PAD)
    }
}

/// Session data
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Session {
    pub id: Uuid,
    pub user_id: Uuid,
    pub token: String,
    pub session_type: SessionType,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub last_activity: chrono::DateTime<chrono::Utc>,
    pub expires_at: chrono::DateTime<chrono::Utc>,
    pub ip_address: Option<String>,
    pub user_agent: Option<String>,
    pub data: HashMap<String, serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SessionType {
    Web,
    Api,
    Mobile,
}
```

## JWT Token Service

```rust
use jsonwebtoken::{encode, decode, Header, Algorithm, Validation, EncodingKey, DecodingKey};

/// JWT token service
pub struct TokenService {
    encoding_key: EncodingKey,
    decoding_key: DecodingKey,
    config: TokenConfig,
}

impl TokenService {
    pub fn new(config: &TokenConfig) -> Result<Self> {
        let encoding_key = EncodingKey::from_secret(config.secret.as_bytes());
        let decoding_key = DecodingKey::from_secret(config.secret.as_bytes());
        
        Ok(Self {
            encoding_key,
            decoding_key,
            config: config.clone(),
        })
    }
    
    /// Generate JWT token
    pub fn generate_token(&self, user_id: &Uuid, scope: TokenScope) -> Result<String> {
        let now = chrono::Utc::now();
        let exp = match scope {
            TokenScope::Access => now + self.config.access_token_ttl,
            TokenScope::Refresh => now + self.config.refresh_token_ttl,
            TokenScope::ApiKey => now + chrono::Duration::days(365),
        };
        
        let claims = TokenClaims {
            sub: user_id.to_string(),
            exp: exp.timestamp() as usize,
            iat: now.timestamp() as usize,
            scope: scope.to_string(),
            jti: Uuid::new_v4().to_string(),
        };
        
        let header = Header::new(Algorithm::HS256);
        
        encode(&header, &claims, &self.encoding_key)
            .map_err(|e| AuthError::Internal(e.to_string()))
    }
    
    /// Verify JWT token
    pub fn verify_token(&self, token: &str) -> Result<TokenClaims> {
        let validation = Validation::new(Algorithm::HS256);
        
        let token_data = decode::<TokenClaims>(token, &self.decoding_key, &validation)
            .map_err(|e| match e.kind() {
                jsonwebtoken::errors::ErrorKind::ExpiredSignature => AuthError::TokenExpired,
                _ => AuthError::InvalidToken,
            })?;
        
        Ok(token_data.claims)
    }
    
    /// Refresh access token
    pub async fn refresh_token(&self, refresh_token: &str) -> Result<TokenPair> {
        // Verify refresh token
        let claims = self.verify_token(refresh_token)?;
        
        if claims.scope != "refresh" {
            return Err(AuthError::InvalidToken);
        }
        
        let user_id = Uuid::parse_str(&claims.sub)
            .map_err(|_| AuthError::InvalidToken)?;
        
        // Generate new token pair
        let access_token = self.generate_token(&user_id, TokenScope::Access)?;
        let new_refresh_token = self.generate_token(&user_id, TokenScope::Refresh)?;
        
        Ok(TokenPair {
            access_token,
            refresh_token: new_refresh_token,
            expires_in: self.config.access_token_ttl.num_seconds(),
        })
    }
}

/// Token claims
#[derive(Debug, Serialize, Deserialize)]
pub struct TokenClaims {
    pub sub: String,
    pub exp: usize,
    pub iat: usize,
    pub scope: String,
    pub jti: String,
}

#[derive(Debug, Clone)]
pub enum TokenScope {
    Access,
    Refresh,
    ApiKey,
}

impl ToString for TokenScope {
    fn to_string(&self) -> String {
        match self {
            TokenScope::Access => "access".to_string(),
            TokenScope::Refresh => "refresh".to_string(),
            TokenScope::ApiKey => "api_key".to_string(),
        }
    }
}

#[derive(Debug, Clone)]
pub struct TokenPair {
    pub access_token: String,
    pub refresh_token: String,
    pub expires_in: i64,
}
```

## OAuth Providers

```rust
/// OAuth authentication provider trait
#[async_trait]
pub trait AuthProvider: Send + Sync {
    /// Get provider name
    fn name(&self) -> &str;
    
    /// Get authorization URL
    async fn get_auth_url(&self, state: &str) -> Result<String>;
    
    /// Exchange code for user info
    async fn exchange_code(&self, code: &str) -> Result<OAuthUserInfo>;
}

/// OAuth user info
#[derive(Debug, Clone)]
pub struct OAuthUserInfo {
    pub provider: String,
    pub provider_id: String,
    pub email: String,
    pub name: Option<String>,
    pub picture: Option<String>,
}

/// GitHub OAuth provider
pub struct GitHubProvider {
    client_id: String,
    client_secret: String,
    redirect_uri: String,
}

#[async_trait]
impl AuthProvider for GitHubProvider {
    fn name(&self) -> &str {
        "github"
    }
    
    async fn get_auth_url(&self, state: &str) -> Result<String> {
        let params = [
            ("client_id", &self.client_id),
            ("redirect_uri", &self.redirect_uri),
            ("scope", &"user:email".to_string()),
            ("state", &state.to_string()),
        ];
        
        let url = format!(
            "https://github.com/login/oauth/authorize?{}",
            serde_urlencoded::to_string(&params)?
        );
        
        Ok(url)
    }
    
    async fn exchange_code(&self, code: &str) -> Result<OAuthUserInfo> {
        // Exchange code for token
        let token_params = [
            ("client_id", &self.client_id),
            ("client_secret", &self.client_secret),
            ("code", &code.to_string()),
        ];
        
        let client = reqwest::Client::new();
        let token_response = client
            .post("https://github.com/login/oauth/access_token")
            .header("Accept", "application/json")
            .form(&token_params)
            .send()
            .await?;
        
        let token_data: serde_json::Value = token_response.json().await?;
        let access_token = token_data["access_token"]
            .as_str()
            .ok_or_else(|| AuthError::Internal("No access token".into()))?;
        
        // Get user info
        let user_response = client
            .get("https://api.github.com/user")
            .header("Authorization", format!("Bearer {}", access_token))
            .header("User-Agent", "ReedCMS")
            .send()
            .await?;
        
        let user_data: serde_json::Value = user_response.json().await?;
        
        // Get primary email
        let email_response = client
            .get("https://api.github.com/user/emails")
            .header("Authorization", format!("Bearer {}", access_token))
            .header("User-Agent", "ReedCMS")
            .send()
            .await?;
        
        let emails: Vec<serde_json::Value> = email_response.json().await?;
        let primary_email = emails
            .iter()
            .find(|e| e["primary"].as_bool().unwrap_or(false))
            .and_then(|e| e["email"].as_str())
            .ok_or_else(|| AuthError::Internal("No primary email".into()))?;
        
        Ok(OAuthUserInfo {
            provider: "github".to_string(),
            provider_id: user_data["id"].to_string(),
            email: primary_email.to_string(),
            name: user_data["name"].as_str().map(|s| s.to_string()),
            picture: user_data["avatar_url"].as_str().map(|s| s.to_string()),
        })
    }
}
```

## User Store

```rust
/// User storage service
pub struct UserStore {
    db: Arc<Database>,
    cache: Arc<RedisCache>,
}

impl UserStore {
    /// Create new user
    pub async fn create_user(&self, data: CreateUserData) -> Result<User> {
        // Hash password
        let salt = SaltString::generate(&mut OsRng);
        let argon2 = Argon2::default();
        let password_hash = argon2
            .hash_password(data.password.as_bytes(), &salt)?
            .to_string();
        
        let user = User {
            id: Uuid::new_v4(),
            username: data.username,
            email: data.email,
            password_hash,
            is_active: true,
            is_verified: false,
            created_at: chrono::Utc::now(),
            updated_at: chrono::Utc::now(),
            last_login: None,
            permissions: vec![],
            metadata: HashMap::new(),
        };
        
        // Insert into database
        sqlx::query!(
            r#"
            INSERT INTO users (id, username, email, password_hash, is_active, is_verified, created_at, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            "#,
            user.id,
            user.username,
            user.email,
            user.password_hash,
            user.is_active,
            user.is_verified,
            user.created_at,
            user.updated_at,
        )
        .execute(&*self.db.pool)
        .await?;
        
        // Cache user
        self.cache_user(&user).await?;
        
        Ok(user)
    }
    
    /// Find user by username or email
    pub async fn find_by_username_or_email(&self, identifier: &str) -> Result<Option<User>> {
        // Check cache first
        let cache_key = format!("user:by_identifier:{}", identifier);
        if let Some(cached) = self.cache.get::<User>(&cache_key).await? {
            return Ok(Some(cached));
        }
        
        // Query database
        let user = sqlx::query_as!(
            User,
            r#"
            SELECT * FROM users
            WHERE username = $1 OR email = $1
            "#,
            identifier
        )
        .fetch_optional(&*self.db.pool)
        .await?;
        
        if let Some(ref user) = user {
            self.cache_user(user).await?;
        }
        
        Ok(user)
    }
    
    /// Update last login
    pub async fn update_last_login(&self, user_id: &Uuid) -> Result<()> {
        let now = chrono::Utc::now();
        
        sqlx::query!(
            r#"
            UPDATE users
            SET last_login = $1
            WHERE id = $2
            "#,
            now,
            user_id
        )
        .execute(&*self.db.pool)
        .await?;
        
        // Invalidate cache
        self.invalidate_user_cache(user_id).await?;
        
        Ok(())
    }
    
    /// Cache user
    async fn cache_user(&self, user: &User) -> Result<()> {
        let ttl = 3600; // 1 hour
        
        // Cache by ID
        let id_key = format!("user:{}", user.id);
        self.cache.set(&id_key, user, Some(ttl)).await?;
        
        // Cache by username
        let username_key = format!("user:by_identifier:{}", user.username);
        self.cache.set(&username_key, user, Some(ttl)).await?;
        
        // Cache by email
        let email_key = format!("user:by_identifier:{}", user.email);
        self.cache.set(&email_key, user, Some(ttl)).await?;
        
        Ok(())
    }
}

/// User model
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub password_hash: String,
    pub is_active: bool,
    pub is_verified: bool,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub updated_at: chrono::DateTime<chrono::Utc>,
    pub last_login: Option<chrono::DateTime<chrono::Utc>>,
    pub permissions: Vec<String>,
    pub metadata: HashMap<String, serde_json::Value>,
}
```

## Authentication Middleware

```rust
use axum::{
    extract::{Request, State},
    middleware::Next,
    response::Response,
};

/// Authentication middleware
pub async fn auth_middleware(
    State(state): State<AppState>,
    mut req: Request,
    next: Next,
) -> Result<Response, AuthError> {
    // Extract token from header
    let token = extract_token(&req)?;
    
    // Authenticate
    let auth_result = state.auth_service
        .authenticate(AuthCredentials::Token { token })
        .await?;
    
    // Insert auth context
    req.extensions_mut().insert(AuthContext {
        user_id: auth_result.user_id,
        permissions: auth_result.permissions,
    });
    
    Ok(next.run(req).await)
}

/// Extract token from request
fn extract_token(req: &Request) -> Result<String, AuthError> {
    // Check Authorization header
    if let Some(auth_header) = req.headers().get("Authorization") {
        let auth_str = auth_header
            .to_str()
            .map_err(|_| AuthError::InvalidToken)?;
        
        if auth_str.starts_with("Bearer ") {
            return Ok(auth_str[7..].to_string());
        }
    }
    
    // Check cookie
    if let Some(cookie_header) = req.headers().get("Cookie") {
        let cookie_str = cookie_header
            .to_str()
            .map_err(|_| AuthError::InvalidToken)?;
        
        for cookie in cookie_str.split(';') {
            let parts: Vec<&str> = cookie.trim().split('=').collect();
            if parts.len() == 2 && parts[0] == "auth_token" {
                return Ok(parts[1].to_string());
            }
        }
    }
    
    Err(AuthError::InvalidToken)
}

/// Authentication context
#[derive(Debug, Clone)]
pub struct AuthContext {
    pub user_id: Uuid,
    pub permissions: Vec<String>,
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AuthConfig {
    pub session: SessionConfig,
    pub token: TokenConfig,
    pub password: PasswordConfig,
    pub oauth: OAuthConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SessionConfig {
    pub ttl: chrono::Duration,
    pub max_sessions_per_user: usize,
    pub secure_cookie: bool,
    pub same_site: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TokenConfig {
    pub secret: String,
    pub access_token_ttl: chrono::Duration,
    pub refresh_token_ttl: chrono::Duration,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PasswordConfig {
    pub min_length: usize,
    pub require_uppercase: bool,
    pub require_lowercase: bool,
    pub require_numbers: bool,
    pub require_special: bool,
    pub argon2_memory_cost: u32,
    pub argon2_time_cost: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OAuthConfig {
    pub providers: HashMap<String, OAuthProviderConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OAuthProviderConfig {
    pub client_id: String,
    pub client_secret: String,
    pub redirect_uri: String,
}
```

## Summary

Diese Authentication Implementation bietet:
- **Multi-Provider Auth** - Password, Token, OAuth, API Keys
- **Session Management** - Redis-based Session Storage
- **JWT Tokens** - Access und Refresh Tokens
- **OAuth Integration** - GitHub, Google, etc.
- **Password Security** - Argon2 Hashing mit Rehashing
- **Auth Middleware** - Request Authentication

Secure und flexible Authentication für ReedCMS.