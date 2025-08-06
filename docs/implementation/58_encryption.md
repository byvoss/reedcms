# Encryption

## Overview

Encryption Implementation für ReedCMS. Data-at-Rest und Data-in-Transit Encryption mit Key Management.

## Encryption Service

```rust
use aes_gcm::{
    aead::{Aead, KeyInit, OsRng},
    Aes256Gcm, Key, Nonce,
};
use argon2::{Argon2, PasswordHasher, PasswordVerifier};
use ring::rand::{SecureRandom, SystemRandom};
use std::sync::Arc;

/// Main encryption service
pub struct EncryptionService {
    key_manager: Arc<KeyManager>,
    cipher_registry: Arc<CipherRegistry>,
    config: EncryptionConfig,
}

impl EncryptionService {
    pub fn new(config: EncryptionConfig) -> Result<Self> {
        let key_manager = Arc::new(KeyManager::new(&config.key_management)?);
        let cipher_registry = Arc::new(CipherRegistry::new());
        
        Ok(Self {
            key_manager,
            cipher_registry,
            config,
        })
    }
    
    /// Encrypt data
    pub async fn encrypt(
        &self,
        data: &[u8],
        context: EncryptionContext,
    ) -> Result<EncryptedData> {
        // Get encryption key
        let key = self.key_manager
            .get_key(&context.key_id)
            .await?
            .ok_or_else(|| EncryptionError::KeyNotFound(context.key_id.clone()))?;
        
        // Get cipher
        let cipher = self.cipher_registry
            .get_cipher(&context.algorithm)
            .ok_or_else(|| EncryptionError::UnsupportedAlgorithm(context.algorithm.clone()))?;
        
        // Generate nonce
        let nonce = self.generate_nonce(cipher.nonce_size())?;
        
        // Encrypt
        let ciphertext = cipher.encrypt(&key.material, &nonce, data, context.aad.as_deref())?;
        
        Ok(EncryptedData {
            algorithm: context.algorithm,
            key_id: context.key_id,
            nonce: nonce.to_vec(),
            ciphertext,
            aad: context.aad,
            metadata: context.metadata,
        })
    }
    
    /// Decrypt data
    pub async fn decrypt(
        &self,
        encrypted: &EncryptedData,
    ) -> Result<Vec<u8>> {
        // Get decryption key
        let key = self.key_manager
            .get_key(&encrypted.key_id)
            .await?
            .ok_or_else(|| EncryptionError::KeyNotFound(encrypted.key_id.clone()))?;
        
        // Get cipher
        let cipher = self.cipher_registry
            .get_cipher(&encrypted.algorithm)
            .ok_or_else(|| EncryptionError::UnsupportedAlgorithm(encrypted.algorithm.clone()))?;
        
        // Decrypt
        let plaintext = cipher.decrypt(
            &key.material,
            &encrypted.nonce,
            &encrypted.ciphertext,
            encrypted.aad.as_deref(),
        )?;
        
        Ok(plaintext)
    }
    
    /// Encrypt field
    pub async fn encrypt_field(
        &self,
        value: &str,
        field_name: &str,
    ) -> Result<String> {
        let context = EncryptionContext {
            key_id: self.config.default_key_id.clone(),
            algorithm: Algorithm::Aes256Gcm,
            aad: Some(field_name.as_bytes().to_vec()),
            metadata: HashMap::new(),
        };
        
        let encrypted = self.encrypt(value.as_bytes(), context).await?;
        
        // Encode as base64
        Ok(format!(
            "enc:{}:{}:{}",
            encrypted.algorithm,
            base64::encode(&encrypted.nonce),
            base64::encode(&encrypted.ciphertext)
        ))
    }
    
    /// Decrypt field
    pub async fn decrypt_field(&self, encrypted_value: &str) -> Result<String> {
        if !encrypted_value.starts_with("enc:") {
            return Err(EncryptionError::InvalidFormat);
        }
        
        let parts: Vec<&str> = encrypted_value.split(':').collect();
        if parts.len() != 4 {
            return Err(EncryptionError::InvalidFormat);
        }
        
        let algorithm = Algorithm::from_str(parts[1])?;
        let nonce = base64::decode(parts[2])?;
        let ciphertext = base64::decode(parts[3])?;
        
        let encrypted = EncryptedData {
            algorithm,
            key_id: self.config.default_key_id.clone(),
            nonce,
            ciphertext,
            aad: None,
            metadata: HashMap::new(),
        };
        
        let plaintext = self.decrypt(&encrypted).await?;
        
        String::from_utf8(plaintext)
            .map_err(|_| EncryptionError::InvalidUtf8)
    }
    
    /// Generate secure nonce
    fn generate_nonce(&self, size: usize) -> Result<Vec<u8>> {
        let mut nonce = vec![0u8; size];
        let rng = SystemRandom::new();
        rng.fill(&mut nonce)
            .map_err(|_| EncryptionError::RngError)?;
        Ok(nonce)
    }
}

/// Encryption context
#[derive(Debug, Clone)]
pub struct EncryptionContext {
    pub key_id: String,
    pub algorithm: Algorithm,
    pub aad: Option<Vec<u8>>,
    pub metadata: HashMap<String, String>,
}

/// Encrypted data
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EncryptedData {
    pub algorithm: Algorithm,
    pub key_id: String,
    pub nonce: Vec<u8>,
    pub ciphertext: Vec<u8>,
    pub aad: Option<Vec<u8>>,
    pub metadata: HashMap<String, String>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum Algorithm {
    Aes256Gcm,
    ChaCha20Poly1305,
    Aes256CbcHmac,
}
```

## Key Management

```rust
use ring::aead::{UnboundKey, AES_256_GCM};
use zeroize::Zeroize;

/// Key management service
pub struct KeyManager {
    key_store: Arc<KeyStore>,
    kms_client: Option<Arc<KmsClient>>,
    cache: Arc<RwLock<HashMap<String, CachedKey>>>,
    config: KeyManagementConfig,
}

impl KeyManager {
    pub fn new(config: &KeyManagementConfig) -> Result<Self> {
        let key_store = Arc::new(KeyStore::new(&config.store)?);
        
        let kms_client = if config.use_kms {
            Some(Arc::new(KmsClient::new(&config.kms)?))
        } else {
            None
        };
        
        Ok(Self {
            key_store,
            kms_client,
            cache: Arc::new(RwLock::new(HashMap::new())),
            config: config.clone(),
        })
    }
    
    /// Get encryption key
    pub async fn get_key(&self, key_id: &str) -> Result<Option<EncryptionKey>> {
        // Check cache
        if let Some(cached) = self.get_cached_key(key_id).await {
            return Ok(Some(cached));
        }
        
        // Load from store
        let key = if let Some(kms) = &self.kms_client {
            // Use KMS
            self.get_key_from_kms(key_id, kms).await?
        } else {
            // Use local key store
            self.key_store.get_key(key_id).await?
        };
        
        if let Some(ref key) = key {
            self.cache_key(key_id, key).await?;
        }
        
        Ok(key)
    }
    
    /// Generate new key
    pub async fn generate_key(
        &self,
        key_id: &str,
        algorithm: Algorithm,
    ) -> Result<EncryptionKey> {
        let key_material = match algorithm {
            Algorithm::Aes256Gcm => {
                let mut key = vec![0u8; 32];
                OsRng.fill_bytes(&mut key);
                key
            }
            Algorithm::ChaCha20Poly1305 => {
                let mut key = vec![0u8; 32];
                OsRng.fill_bytes(&mut key);
                key
            }
            Algorithm::Aes256CbcHmac => {
                let mut key = vec![0u8; 64]; // 32 for AES + 32 for HMAC
                OsRng.fill_bytes(&mut key);
                key
            }
        };
        
        let key = EncryptionKey {
            id: key_id.to_string(),
            algorithm,
            material: key_material,
            created_at: chrono::Utc::now(),
            expires_at: None,
            metadata: HashMap::new(),
        };
        
        // Store key
        if let Some(kms) = &self.kms_client {
            self.store_key_in_kms(&key, kms).await?;
        } else {
            self.key_store.store_key(&key).await?;
        }
        
        Ok(key)
    }
    
    /// Rotate key
    pub async fn rotate_key(&self, key_id: &str) -> Result<EncryptionKey> {
        // Get current key
        let current_key = self.get_key(key_id)
            .await?
            .ok_or_else(|| KeyManagementError::KeyNotFound(key_id.to_string()))?;
        
        // Generate new version
        let new_key_id = format!("{}_v{}", key_id, chrono::Utc::now().timestamp());
        let new_key = self.generate_key(&new_key_id, current_key.algorithm).await?;
        
        // Update key mapping
        self.key_store.update_key_mapping(key_id, &new_key_id).await?;
        
        // Invalidate cache
        self.invalidate_cached_key(key_id).await;
        
        Ok(new_key)
    }
    
    /// Cache key
    async fn cache_key(&self, key_id: &str, key: &EncryptionKey) -> Result<()> {
        let cached = CachedKey {
            key: key.clone(),
            cached_at: chrono::Utc::now(),
        };
        
        self.cache.write().await.insert(key_id.to_string(), cached);
        
        Ok(())
    }
    
    /// Get cached key
    async fn get_cached_key(&self, key_id: &str) -> Option<EncryptionKey> {
        let cache = self.cache.read().await;
        
        if let Some(cached) = cache.get(key_id) {
            let age = chrono::Utc::now() - cached.cached_at;
            if age < self.config.cache_ttl {
                return Some(cached.key.clone());
            }
        }
        
        None
    }
}

/// Encryption key
#[derive(Debug, Clone)]
pub struct EncryptionKey {
    pub id: String,
    pub algorithm: Algorithm,
    pub material: Vec<u8>,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub expires_at: Option<chrono::DateTime<chrono::Utc>>,
    pub metadata: HashMap<String, String>,
}

impl Drop for EncryptionKey {
    fn drop(&mut self) {
        self.material.zeroize();
    }
}

/// Cached key
struct CachedKey {
    key: EncryptionKey,
    cached_at: chrono::DateTime<chrono::Utc>,
}
```

## Cipher Registry

```rust
/// Registry of encryption ciphers
pub struct CipherRegistry {
    ciphers: HashMap<Algorithm, Box<dyn Cipher>>,
}

impl CipherRegistry {
    pub fn new() -> Self {
        let mut registry = Self {
            ciphers: HashMap::new(),
        };
        
        // Register default ciphers
        registry.register_defaults();
        
        registry
    }
    
    /// Get cipher
    pub fn get_cipher(&self, algorithm: &Algorithm) -> Option<&dyn Cipher> {
        self.ciphers.get(algorithm).map(|c| c.as_ref())
    }
    
    /// Register cipher
    pub fn register_cipher(&mut self, algorithm: Algorithm, cipher: Box<dyn Cipher>) {
        self.ciphers.insert(algorithm, cipher);
    }
    
    /// Register default ciphers
    fn register_defaults(&mut self) {
        // AES-256-GCM
        self.ciphers.insert(
            Algorithm::Aes256Gcm,
            Box::new(Aes256GcmCipher::new()),
        );
        
        // ChaCha20-Poly1305
        self.ciphers.insert(
            Algorithm::ChaCha20Poly1305,
            Box::new(ChaCha20Poly1305Cipher::new()),
        );
        
        // AES-256-CBC with HMAC
        self.ciphers.insert(
            Algorithm::Aes256CbcHmac,
            Box::new(Aes256CbcHmacCipher::new()),
        );
    }
}

/// Cipher trait
pub trait Cipher: Send + Sync {
    fn encrypt(
        &self,
        key: &[u8],
        nonce: &[u8],
        plaintext: &[u8],
        aad: Option<&[u8]>,
    ) -> Result<Vec<u8>>;
    
    fn decrypt(
        &self,
        key: &[u8],
        nonce: &[u8],
        ciphertext: &[u8],
        aad: Option<&[u8]>,
    ) -> Result<Vec<u8>>;
    
    fn nonce_size(&self) -> usize;
}

/// AES-256-GCM cipher
struct Aes256GcmCipher;

impl Aes256GcmCipher {
    fn new() -> Self {
        Self
    }
}

impl Cipher for Aes256GcmCipher {
    fn encrypt(
        &self,
        key: &[u8],
        nonce: &[u8],
        plaintext: &[u8],
        aad: Option<&[u8]>,
    ) -> Result<Vec<u8>> {
        use aes_gcm::aead::generic_array::GenericArray;
        
        let key = Key::<Aes256Gcm>::from_slice(key);
        let cipher = Aes256Gcm::new(key);
        let nonce = GenericArray::from_slice(nonce);
        
        let ciphertext = if let Some(aad) = aad {
            cipher.encrypt_in_place_detached(nonce, aad, plaintext)
                .map_err(|_| EncryptionError::EncryptionFailed)?
        } else {
            cipher.encrypt(nonce, plaintext)
                .map_err(|_| EncryptionError::EncryptionFailed)?
        };
        
        Ok(ciphertext)
    }
    
    fn decrypt(
        &self,
        key: &[u8],
        nonce: &[u8],
        ciphertext: &[u8],
        aad: Option<&[u8]>,
    ) -> Result<Vec<u8>> {
        use aes_gcm::aead::generic_array::GenericArray;
        
        let key = Key::<Aes256Gcm>::from_slice(key);
        let cipher = Aes256Gcm::new(key);
        let nonce = GenericArray::from_slice(nonce);
        
        let plaintext = if let Some(aad) = aad {
            cipher.decrypt_in_place_detached(nonce, aad, ciphertext)
                .map_err(|_| EncryptionError::DecryptionFailed)?
        } else {
            cipher.decrypt(nonce, ciphertext)
                .map_err(|_| EncryptionError::DecryptionFailed)?
        };
        
        Ok(plaintext)
    }
    
    fn nonce_size(&self) -> usize {
        12 // 96 bits for GCM
    }
}
```

## Field-Level Encryption

```rust
/// Field-level encryption for database
pub struct FieldEncryption {
    encryption_service: Arc<EncryptionService>,
    field_config: Arc<FieldEncryptionConfig>,
}

impl FieldEncryption {
    /// Encrypt struct fields
    pub async fn encrypt_fields<T>(&self, data: &mut T) -> Result<()>
    where
        T: Serialize + for<'de> Deserialize<'de>,
    {
        let mut value = serde_json::to_value(&data)?;
        
        if let Some(obj) = value.as_object_mut() {
            for (field_name, field_value) in obj.iter_mut() {
                if self.should_encrypt_field(field_name) {
                    if let Some(str_value) = field_value.as_str() {
                        let encrypted = self.encryption_service
                            .encrypt_field(str_value, field_name)
                            .await?;
                        *field_value = serde_json::Value::String(encrypted);
                    }
                }
            }
        }
        
        *data = serde_json::from_value(value)?;
        
        Ok(())
    }
    
    /// Decrypt struct fields
    pub async fn decrypt_fields<T>(&self, data: &mut T) -> Result<()>
    where
        T: Serialize + for<'de> Deserialize<'de>,
    {
        let mut value = serde_json::to_value(&data)?;
        
        if let Some(obj) = value.as_object_mut() {
            for (field_name, field_value) in obj.iter_mut() {
                if let Some(str_value) = field_value.as_str() {
                    if str_value.starts_with("enc:") {
                        let decrypted = self.encryption_service
                            .decrypt_field(str_value)
                            .await?;
                        *field_value = serde_json::Value::String(decrypted);
                    }
                }
            }
        }
        
        *data = serde_json::from_value(value)?;
        
        Ok(())
    }
    
    /// Check if field should be encrypted
    fn should_encrypt_field(&self, field_name: &str) -> bool {
        self.field_config.encrypted_fields.contains(field_name) ||
        self.field_config.patterns.iter().any(|pattern| {
            field_name.contains(pattern)
        })
    }
}

/// Field encryption configuration
#[derive(Debug, Clone)]
pub struct FieldEncryptionConfig {
    pub encrypted_fields: HashSet<String>,
    pub patterns: Vec<String>,
}

impl Default for FieldEncryptionConfig {
    fn default() -> Self {
        let mut encrypted_fields = HashSet::new();
        encrypted_fields.insert("password".to_string());
        encrypted_fields.insert("email".to_string());
        encrypted_fields.insert("phone".to_string());
        encrypted_fields.insert("ssn".to_string());
        encrypted_fields.insert("credit_card".to_string());
        
        Self {
            encrypted_fields,
            patterns: vec![
                "secret".to_string(),
                "private".to_string(),
                "sensitive".to_string(),
            ],
        }
    }
}
```

## TLS Configuration

```rust
use rustls::{ServerConfig, Certificate, PrivateKey};
use std::sync::Arc;

/// TLS configuration service
pub struct TlsConfiguration {
    config: Arc<ServerConfig>,
    cert_manager: Arc<CertificateManager>,
}

impl TlsConfiguration {
    pub async fn new(config: TlsConfig) -> Result<Self> {
        let cert_manager = Arc::new(CertificateManager::new(&config)?);
        
        // Load certificates
        let certs = cert_manager.load_certificates().await?;
        let key = cert_manager.load_private_key().await?;
        
        // Build TLS config
        let tls_config = ServerConfig::builder()
            .with_safe_defaults()
            .with_no_client_auth()
            .with_single_cert(certs, key)?;
        
        Ok(Self {
            config: Arc::new(tls_config),
            cert_manager,
        })
    }
    
    /// Get TLS configuration
    pub fn get_config(&self) -> Arc<ServerConfig> {
        self.config.clone()
    }
    
    /// Enable ACME auto-renewal
    pub async fn enable_acme(&self, domains: Vec<String>) -> Result<()> {
        self.cert_manager.setup_acme(domains).await
    }
}

/// Certificate manager
pub struct CertificateManager {
    cert_path: PathBuf,
    key_path: PathBuf,
    acme_client: Option<Arc<AcmeClient>>,
}

impl CertificateManager {
    /// Setup ACME
    pub async fn setup_acme(&self, domains: Vec<String>) -> Result<()> {
        if let Some(acme) = &self.acme_client {
            // Order certificate
            let order = acme.new_order(domains).await?;
            
            // Complete challenges
            for auth in order.authorizations().await? {
                let challenge = auth.http_challenge()
                    .ok_or_else(|| TlsError::NoHttpChallenge)?;
                
                // Set challenge response
                acme.set_challenge_response(&challenge).await?;
                
                // Validate
                challenge.validate().await?;
            }
            
            // Get certificate
            let cert = order.certificate().await?;
            
            // Save certificate
            self.save_certificate(&cert).await?;
        }
        
        Ok(())
    }
}
```

## Database Encryption

```rust
/// Transparent database encryption
pub struct DatabaseEncryption {
    encryption_service: Arc<EncryptionService>,
    config: DatabaseEncryptionConfig,
}

impl DatabaseEncryption {
    /// Encrypt before insert
    pub async fn encrypt_row<T>(&self, row: &mut T) -> Result<()>
    where
        T: EncryptableRow,
    {
        for field in row.encrypted_fields() {
            let value = row.get_field(&field)?;
            let encrypted = self.encryption_service
                .encrypt_field(&value, &field)
                .await?;
            row.set_field(&field, encrypted)?;
        }
        
        Ok(())
    }
    
    /// Decrypt after select
    pub async fn decrypt_row<T>(&self, row: &mut T) -> Result<()>
    where
        T: EncryptableRow,
    {
        for field in row.encrypted_fields() {
            let value = row.get_field(&field)?;
            if value.starts_with("enc:") {
                let decrypted = self.encryption_service
                    .decrypt_field(&value)
                    .await?;
                row.set_field(&field, decrypted)?;
            }
        }
        
        Ok(())
    }
}

/// Trait for encryptable database rows
pub trait EncryptableRow {
    fn encrypted_fields(&self) -> Vec<String>;
    fn get_field(&self, name: &str) -> Result<String>;
    fn set_field(&mut self, name: &str, value: String) -> Result<()>;
}

/// SQLx type wrapper for encrypted fields
#[derive(Debug, Clone)]
pub struct Encrypted<T>(pub T);

impl<T> sqlx::Type<sqlx::Postgres> for Encrypted<T>
where
    T: ToString + FromStr,
{
    fn type_info() -> sqlx::postgres::PgTypeInfo {
        <String as sqlx::Type<sqlx::Postgres>>::type_info()
    }
}

impl<T> sqlx::Encode<'_, sqlx::Postgres> for Encrypted<T>
where
    T: ToString,
{
    fn encode_by_ref(&self, buf: &mut sqlx::postgres::PgArgumentBuffer) -> sqlx::encode::IsNull {
        // Encrypt value here
        let encrypted = encrypt_value(&self.0.to_string());
        encrypted.encode_by_ref(buf)
    }
}

impl<T> sqlx::Decode<'_, sqlx::Postgres> for Encrypted<T>
where
    T: FromStr,
{
    fn decode(value: sqlx::postgres::PgValueRef<'_>) -> Result<Self, sqlx::error::BoxDynError> {
        let encrypted = <String as sqlx::Decode<sqlx::Postgres>>::decode(value)?;
        // Decrypt value here
        let decrypted = decrypt_value(&encrypted)?;
        let parsed = T::from_str(&decrypted)?;
        Ok(Encrypted(parsed))
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EncryptionConfig {
    pub default_algorithm: Algorithm,
    pub default_key_id: String,
    pub key_management: KeyManagementConfig,
    pub field_encryption: FieldEncryptionConfig,
    pub database_encryption: DatabaseEncryptionConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KeyManagementConfig {
    pub store: KeyStoreConfig,
    pub use_kms: bool,
    pub kms: Option<KmsConfig>,
    pub cache_ttl: chrono::Duration,
    pub rotation_interval: chrono::Duration,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KeyStoreConfig {
    pub path: PathBuf,
    pub master_key_env: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KmsConfig {
    pub provider: KmsProvider,
    pub region: String,
    pub key_id: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum KmsProvider {
    Aws,
    Azure,
    Google,
    HashiCorp,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TlsConfig {
    pub cert_path: PathBuf,
    pub key_path: PathBuf,
    pub enable_acme: bool,
    pub acme_email: String,
    pub acme_directory: String,
}

impl Default for EncryptionConfig {
    fn default() -> Self {
        Self {
            default_algorithm: Algorithm::Aes256Gcm,
            default_key_id: "default".to_string(),
            key_management: KeyManagementConfig {
                store: KeyStoreConfig {
                    path: PathBuf::from("data/keys"),
                    master_key_env: "REED_MASTER_KEY".to_string(),
                },
                use_kms: false,
                kms: None,
                cache_ttl: chrono::Duration::hours(1),
                rotation_interval: chrono::Duration::days(90),
            },
            field_encryption: FieldEncryptionConfig::default(),
            database_encryption: DatabaseEncryptionConfig {
                enabled: true,
                encrypt_pii: true,
            },
        }
    }
}
```

## Summary

Diese Encryption Implementation bietet:
- **Data-at-Rest Encryption** - AES-256-GCM, ChaCha20-Poly1305
- **Key Management** - Local und KMS Support
- **Field-Level Encryption** - Selective Field Encryption
- **Database Encryption** - Transparent Row Encryption
- **TLS Configuration** - ACME Support
- **Secure Key Storage** - Zeroization, Rotation

Comprehensive Encryption für Data Security.