# Authorization

## Overview

Authorization Implementation für ReedCMS. Role-Based Access Control (RBAC) mit Permission System und Policy Engine.

## Authorization Core

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

/// Main authorization service
pub struct AuthorizationService {
    permission_store: Arc<PermissionStore>,
    role_manager: Arc<RoleManager>,
    policy_engine: Arc<PolicyEngine>,
    resource_registry: Arc<ResourceRegistry>,
    cache: Arc<AuthzCache>,
    config: AuthzConfig,
}

impl AuthorizationService {
    pub fn new(
        db: Arc<Database>,
        cache: Arc<RedisCache>,
        config: AuthzConfig,
    ) -> Result<Self> {
        let permission_store = Arc::new(PermissionStore::new(db.clone()));
        let role_manager = Arc::new(RoleManager::new(db.clone(), cache.clone()));
        let policy_engine = Arc::new(PolicyEngine::new(config.policies.clone()));
        let resource_registry = Arc::new(ResourceRegistry::new());
        let authz_cache = Arc::new(AuthzCache::new(cache.clone()));
        
        Ok(Self {
            permission_store,
            role_manager,
            policy_engine,
            resource_registry,
            cache: authz_cache,
            config,
        })
    }
    
    /// Check if user has permission
    pub async fn has_permission(
        &self,
        user_id: &Uuid,
        permission: &str,
        context: Option<AuthzContext>,
    ) -> Result<bool> {
        // Check cache first
        let cache_key = format!("authz:{}:{}", user_id, permission);
        if let Some(cached) = self.cache.get::<bool>(&cache_key).await? {
            return Ok(cached);
        }
        
        // Get user permissions
        let user_permissions = self.get_user_permissions(user_id).await?;
        
        // Direct permission check
        if user_permissions.contains(permission) {
            self.cache.set(&cache_key, &true, Some(300)).await?;
            return Ok(true);
        }
        
        // Wildcard permission check
        if self.check_wildcard_permission(&user_permissions, permission) {
            self.cache.set(&cache_key, &true, Some(300)).await?;
            return Ok(true);
        }
        
        // Policy-based check
        if let Some(ctx) = context {
            if self.policy_engine.evaluate(user_id, permission, &ctx).await? {
                self.cache.set(&cache_key, &true, Some(300)).await?;
                return Ok(true);
            }
        }
        
        self.cache.set(&cache_key, &false, Some(300)).await?;
        Ok(false)
    }
    
    /// Check resource access
    pub async fn can_access_resource(
        &self,
        user_id: &Uuid,
        resource: &Resource,
        action: &str,
    ) -> Result<bool> {
        // Build permission string
        let permission = format!("{}:{}:{}", resource.resource_type, action, resource.id);
        
        // Build context
        let context = AuthzContext {
            user_id: *user_id,
            resource: Some(resource.clone()),
            action: action.to_string(),
            environment: self.build_environment().await?,
        };
        
        self.has_permission(user_id, &permission, Some(context)).await
    }
    
    /// Get user permissions
    async fn get_user_permissions(&self, user_id: &Uuid) -> Result<Vec<String>> {
        // Get from cache
        let cache_key = format!("user:{}:permissions", user_id);
        if let Some(cached) = self.cache.get::<Vec<String>>(&cache_key).await? {
            return Ok(cached);
        }
        
        // Get user roles
        let roles = self.role_manager.get_user_roles(user_id).await?;
        
        // Collect all permissions
        let mut permissions = Vec::new();
        
        // Direct user permissions
        let direct_perms = self.permission_store
            .get_user_permissions(user_id)
            .await?;
        permissions.extend(direct_perms);
        
        // Role permissions
        for role in roles {
            let role_perms = self.permission_store
                .get_role_permissions(&role.id)
                .await?;
            permissions.extend(role_perms);
        }
        
        // Deduplicate
        permissions.sort();
        permissions.dedup();
        
        // Cache
        self.cache.set(&cache_key, &permissions, Some(600)).await?;
        
        Ok(permissions)
    }
    
    /// Check wildcard permission
    fn check_wildcard_permission(
        &self,
        permissions: &[String],
        required: &str,
    ) -> bool {
        let required_parts: Vec<&str> = required.split(':').collect();
        
        for perm in permissions {
            let perm_parts: Vec<&str> = perm.split(':').collect();
            
            if perm_parts.len() <= required_parts.len() {
                let mut matches = true;
                
                for (i, part) in perm_parts.iter().enumerate() {
                    if *part != "*" && *part != required_parts[i] {
                        matches = false;
                        break;
                    }
                }
                
                if matches {
                    return true;
                }
            }
        }
        
        false
    }
}

/// Authorization context
#[derive(Debug, Clone)]
pub struct AuthzContext {
    pub user_id: Uuid,
    pub resource: Option<Resource>,
    pub action: String,
    pub environment: HashMap<String, String>,
}

/// Resource representation
#[derive(Debug, Clone)]
pub struct Resource {
    pub id: String,
    pub resource_type: String,
    pub owner_id: Option<Uuid>,
    pub attributes: HashMap<String, serde_json::Value>,
}
```

## Role Management

```rust
/// Role manager
pub struct RoleManager {
    db: Arc<Database>,
    cache: Arc<RedisCache>,
}

impl RoleManager {
    /// Create new role
    pub async fn create_role(&self, data: CreateRoleData) -> Result<Role> {
        let role = Role {
            id: Uuid::new_v4(),
            name: data.name,
            display_name: data.display_name,
            description: data.description,
            permissions: vec![],
            is_system: false,
            created_at: chrono::Utc::now(),
            updated_at: chrono::Utc::now(),
        };
        
        // Insert into database
        sqlx::query!(
            r#"
            INSERT INTO roles (id, name, display_name, description, is_system, created_at, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            "#,
            role.id,
            role.name,
            role.display_name,
            role.description,
            role.is_system,
            role.created_at,
            role.updated_at,
        )
        .execute(&*self.db.pool)
        .await?;
        
        // Assign permissions
        if !data.permissions.is_empty() {
            self.assign_permissions_to_role(&role.id, &data.permissions).await?;
        }
        
        Ok(role)
    }
    
    /// Get user roles
    pub async fn get_user_roles(&self, user_id: &Uuid) -> Result<Vec<Role>> {
        let cache_key = format!("user:{}:roles", user_id);
        
        if let Some(cached) = self.cache.get::<Vec<Role>>(&cache_key).await? {
            return Ok(cached);
        }
        
        let roles = sqlx::query_as!(
            Role,
            r#"
            SELECT r.* FROM roles r
            JOIN user_roles ur ON r.id = ur.role_id
            WHERE ur.user_id = $1
            ORDER BY r.name
            "#,
            user_id
        )
        .fetch_all(&*self.db.pool)
        .await?;
        
        self.cache.set(&cache_key, &roles, Some(600)).await?;
        
        Ok(roles)
    }
    
    /// Assign role to user
    pub async fn assign_role_to_user(
        &self,
        user_id: &Uuid,
        role_id: &Uuid,
    ) -> Result<()> {
        sqlx::query!(
            r#"
            INSERT INTO user_roles (user_id, role_id, assigned_at)
            VALUES ($1, $2, $3)
            ON CONFLICT (user_id, role_id) DO NOTHING
            "#,
            user_id,
            role_id,
            chrono::Utc::now(),
        )
        .execute(&*self.db.pool)
        .await?;
        
        // Invalidate caches
        self.invalidate_user_caches(user_id).await?;
        
        Ok(())
    }
    
    /// Remove role from user
    pub async fn remove_role_from_user(
        &self,
        user_id: &Uuid,
        role_id: &Uuid,
    ) -> Result<()> {
        sqlx::query!(
            r#"
            DELETE FROM user_roles
            WHERE user_id = $1 AND role_id = $2
            "#,
            user_id,
            role_id,
        )
        .execute(&*self.db.pool)
        .await?;
        
        // Invalidate caches
        self.invalidate_user_caches(user_id).await?;
        
        Ok(())
    }
    
    /// Create system roles
    pub async fn create_system_roles(&self) -> Result<()> {
        let system_roles = vec![
            ("admin", "Administrator", "Full system access"),
            ("editor", "Editor", "Content management access"),
            ("author", "Author", "Create and edit own content"),
            ("viewer", "Viewer", "Read-only access"),
        ];
        
        for (name, display_name, description) in system_roles {
            let existing = sqlx::query!(
                "SELECT id FROM roles WHERE name = $1",
                name
            )
            .fetch_optional(&*self.db.pool)
            .await?;
            
            if existing.is_none() {
                sqlx::query!(
                    r#"
                    INSERT INTO roles (id, name, display_name, description, is_system, created_at, updated_at)
                    VALUES ($1, $2, $3, $4, true, $5, $6)
                    "#,
                    Uuid::new_v4(),
                    name,
                    display_name,
                    description,
                    chrono::Utc::now(),
                    chrono::Utc::now(),
                )
                .execute(&*self.db.pool)
                .await?;
            }
        }
        
        Ok(())
    }
}

/// Role model
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Role {
    pub id: Uuid,
    pub name: String,
    pub display_name: String,
    pub description: Option<String>,
    pub permissions: Vec<String>,
    pub is_system: bool,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub updated_at: chrono::DateTime<chrono::Utc>,
}
```

## Permission System

```rust
/// Permission store
pub struct PermissionStore {
    db: Arc<Database>,
}

impl PermissionStore {
    /// Register permission
    pub async fn register_permission(&self, perm: PermissionDefinition) -> Result<()> {
        sqlx::query!(
            r#"
            INSERT INTO permissions (name, resource_type, action, description)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (name) DO UPDATE
            SET description = EXCLUDED.description,
                updated_at = NOW()
            "#,
            perm.name,
            perm.resource_type,
            perm.action,
            perm.description,
        )
        .execute(&*self.db.pool)
        .await?;
        
        Ok(())
    }
    
    /// Get user permissions
    pub async fn get_user_permissions(&self, user_id: &Uuid) -> Result<Vec<String>> {
        let perms = sqlx::query!(
            r#"
            SELECT p.name FROM permissions p
            JOIN user_permissions up ON p.id = up.permission_id
            WHERE up.user_id = $1
            "#,
            user_id
        )
        .fetch_all(&*self.db.pool)
        .await?;
        
        Ok(perms.into_iter().map(|r| r.name).collect())
    }
    
    /// Get role permissions
    pub async fn get_role_permissions(&self, role_id: &Uuid) -> Result<Vec<String>> {
        let perms = sqlx::query!(
            r#"
            SELECT p.name FROM permissions p
            JOIN role_permissions rp ON p.id = rp.permission_id
            WHERE rp.role_id = $1
            "#,
            role_id
        )
        .fetch_all(&*self.db.pool)
        .await?;
        
        Ok(perms.into_iter().map(|r| r.name).collect())
    }
    
    /// Grant permission to user
    pub async fn grant_permission_to_user(
        &self,
        user_id: &Uuid,
        permission: &str,
    ) -> Result<()> {
        let perm_id = sqlx::query!(
            "SELECT id FROM permissions WHERE name = $1",
            permission
        )
        .fetch_one(&*self.db.pool)
        .await?
        .id;
        
        sqlx::query!(
            r#"
            INSERT INTO user_permissions (user_id, permission_id, granted_at)
            VALUES ($1, $2, $3)
            ON CONFLICT (user_id, permission_id) DO NOTHING
            "#,
            user_id,
            perm_id,
            chrono::Utc::now(),
        )
        .execute(&*self.db.pool)
        .await?;
        
        Ok(())
    }
    
    /// Register default permissions
    pub async fn register_default_permissions(&self) -> Result<()> {
        let permissions = vec![
            // Content permissions
            PermissionDefinition {
                name: "content:create".to_string(),
                resource_type: "content".to_string(),
                action: "create".to_string(),
                description: Some("Create new content".to_string()),
            },
            PermissionDefinition {
                name: "content:read".to_string(),
                resource_type: "content".to_string(),
                action: "read".to_string(),
                description: Some("View content".to_string()),
            },
            PermissionDefinition {
                name: "content:update".to_string(),
                resource_type: "content".to_string(),
                action: "update".to_string(),
                description: Some("Update content".to_string()),
            },
            PermissionDefinition {
                name: "content:delete".to_string(),
                resource_type: "content".to_string(),
                action: "delete".to_string(),
                description: Some("Delete content".to_string()),
            },
            PermissionDefinition {
                name: "content:publish".to_string(),
                resource_type: "content".to_string(),
                action: "publish".to_string(),
                description: Some("Publish content".to_string()),
            },
            
            // User permissions
            PermissionDefinition {
                name: "user:create".to_string(),
                resource_type: "user".to_string(),
                action: "create".to_string(),
                description: Some("Create users".to_string()),
            },
            PermissionDefinition {
                name: "user:manage".to_string(),
                resource_type: "user".to_string(),
                action: "manage".to_string(),
                description: Some("Manage users".to_string()),
            },
            
            // System permissions
            PermissionDefinition {
                name: "system:admin".to_string(),
                resource_type: "system".to_string(),
                action: "admin".to_string(),
                description: Some("System administration".to_string()),
            },
        ];
        
        for perm in permissions {
            self.register_permission(perm).await?;
        }
        
        Ok(())
    }
}

/// Permission definition
#[derive(Debug, Clone)]
pub struct PermissionDefinition {
    pub name: String,
    pub resource_type: String,
    pub action: String,
    pub description: Option<String>,
}
```

## Policy Engine

```rust
use rego::{Runtime, Value};

/// Policy-based authorization engine
pub struct PolicyEngine {
    runtime: Arc<Runtime>,
    policies: Arc<RwLock<HashMap<String, Policy>>>,
    config: PolicyConfig,
}

impl PolicyEngine {
    pub fn new(config: PolicyConfig) -> Self {
        let runtime = Runtime::new().expect("Failed to create Rego runtime");
        
        let engine = Self {
            runtime: Arc::new(runtime),
            policies: Arc::new(RwLock::new(HashMap::new())),
            config,
        };
        
        // Load default policies
        engine.load_default_policies();
        
        engine
    }
    
    /// Evaluate policy
    pub async fn evaluate(
        &self,
        user_id: &Uuid,
        action: &str,
        context: &AuthzContext,
    ) -> Result<bool> {
        // Build input data
        let input = self.build_policy_input(user_id, action, context).await?;
        
        // Get applicable policies
        let policies = self.policies.read().await;
        let applicable_policies: Vec<_> = policies
            .values()
            .filter(|p| p.applies_to(action, &context.resource))
            .collect();
        
        // Evaluate each policy
        for policy in applicable_policies {
            match self.evaluate_policy(policy, &input).await? {
                PolicyResult::Allow => return Ok(true),
                PolicyResult::Deny => return Ok(false),
                PolicyResult::Continue => continue,
            }
        }
        
        // Default deny
        Ok(false)
    }
    
    /// Evaluate single policy
    async fn evaluate_policy(
        &self,
        policy: &Policy,
        input: &serde_json::Value,
    ) -> Result<PolicyResult> {
        let query = format!("data.{}.allow", policy.package);
        
        let result = self.runtime
            .query(&query)
            .with_input(input.clone())
            .execute()?;
        
        if let Some(Value::Bool(allowed)) = result.get(0) {
            if *allowed {
                Ok(PolicyResult::Allow)
            } else {
                Ok(PolicyResult::Deny)
            }
        } else {
            Ok(PolicyResult::Continue)
        }
    }
    
    /// Build policy input
    async fn build_policy_input(
        &self,
        user_id: &Uuid,
        action: &str,
        context: &AuthzContext,
    ) -> Result<serde_json::Value> {
        let mut input = serde_json::json!({
            "user": {
                "id": user_id.to_string(),
            },
            "action": action,
            "environment": context.environment,
        });
        
        if let Some(ref resource) = context.resource {
            input["resource"] = serde_json::json!({
                "id": resource.id,
                "type": resource.resource_type,
                "owner_id": resource.owner_id,
                "attributes": resource.attributes,
            });
        }
        
        Ok(input)
    }
    
    /// Load default policies
    fn load_default_policies(&self) {
        // Owner policy
        let owner_policy = Policy {
            id: "owner-policy".to_string(),
            name: "Resource Owner Policy".to_string(),
            package: "authz.owner".to_string(),
            content: r#"
                package authz.owner
                
                default allow = false
                
                allow {
                    input.resource.owner_id == input.user.id
                }
            "#.to_string(),
            resource_types: vec!["*".to_string()],
            priority: 100,
        };
        
        // Time-based policy
        let time_policy = Policy {
            id: "time-policy".to_string(),
            name: "Business Hours Policy".to_string(),
            package: "authz.time".to_string(),
            content: r#"
                package authz.time
                
                default allow = false
                
                allow {
                    current_hour := time.hour(time.now_ns())
                    current_hour >= 9
                    current_hour < 17
                }
            "#.to_string(),
            resource_types: vec!["admin".to_string()],
            priority: 50,
        };
        
        let mut policies = self.policies.blocking_write();
        policies.insert(owner_policy.id.clone(), owner_policy);
        policies.insert(time_policy.id.clone(), time_policy);
    }
}

/// Policy definition
#[derive(Debug, Clone)]
pub struct Policy {
    pub id: String,
    pub name: String,
    pub package: String,
    pub content: String,
    pub resource_types: Vec<String>,
    pub priority: i32,
}

impl Policy {
    fn applies_to(&self, _action: &str, resource: &Option<Resource>) -> bool {
        if self.resource_types.contains(&"*".to_string()) {
            return true;
        }
        
        if let Some(res) = resource {
            self.resource_types.contains(&res.resource_type)
        } else {
            false
        }
    }
}

#[derive(Debug)]
enum PolicyResult {
    Allow,
    Deny,
    Continue,
}
```

## Resource Registry

```rust
/// Resource type registry
pub struct ResourceRegistry {
    resources: Arc<RwLock<HashMap<String, ResourceType>>>,
}

impl ResourceRegistry {
    pub fn new() -> Self {
        let registry = Self {
            resources: Arc::new(RwLock::new(HashMap::new())),
        };
        
        // Register default resources
        registry.register_defaults();
        
        registry
    }
    
    /// Register resource type
    pub async fn register_resource_type(&self, resource_type: ResourceType) -> Result<()> {
        self.resources.write().await.insert(
            resource_type.name.clone(),
            resource_type,
        );
        
        Ok(())
    }
    
    /// Get resource type
    pub async fn get_resource_type(&self, name: &str) -> Option<ResourceType> {
        self.resources.read().await.get(name).cloned()
    }
    
    /// Register default resource types
    fn register_defaults(&self) {
        let defaults = vec![
            ResourceType {
                name: "content".to_string(),
                display_name: "Content".to_string(),
                actions: vec![
                    "create".to_string(),
                    "read".to_string(),
                    "update".to_string(),
                    "delete".to_string(),
                    "publish".to_string(),
                ],
                attributes: vec![
                    ResourceAttribute {
                        name: "status".to_string(),
                        attribute_type: "string".to_string(),
                        required: true,
                    },
                    ResourceAttribute {
                        name: "author_id".to_string(),
                        attribute_type: "uuid".to_string(),
                        required: true,
                    },
                ],
            },
            ResourceType {
                name: "user".to_string(),
                display_name: "User".to_string(),
                actions: vec![
                    "create".to_string(),
                    "read".to_string(),
                    "update".to_string(),
                    "delete".to_string(),
                    "manage".to_string(),
                ],
                attributes: vec![],
            },
            ResourceType {
                name: "plugin".to_string(),
                display_name: "Plugin".to_string(),
                actions: vec![
                    "install".to_string(),
                    "activate".to_string(),
                    "deactivate".to_string(),
                    "configure".to_string(),
                    "uninstall".to_string(),
                ],
                attributes: vec![],
            },
        ];
        
        let mut resources = self.resources.blocking_write();
        for resource_type in defaults {
            resources.insert(resource_type.name.clone(), resource_type);
        }
    }
}

/// Resource type definition
#[derive(Debug, Clone)]
pub struct ResourceType {
    pub name: String,
    pub display_name: String,
    pub actions: Vec<String>,
    pub attributes: Vec<ResourceAttribute>,
}

#[derive(Debug, Clone)]
pub struct ResourceAttribute {
    pub name: String,
    pub attribute_type: String,
    pub required: bool,
}
```

## Authorization Guards

```rust
use axum::extract::FromRequestParts;

/// Permission guard for routes
pub struct RequirePermission(pub String);

#[async_trait]
impl<S> FromRequestParts<S> for RequirePermission
where
    S: Send + Sync,
{
    type Rejection = AuthzError;
    
    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        // Get auth context
        let auth_context = parts.extensions
            .get::<AuthContext>()
            .ok_or(AuthzError::Unauthorized)?;
        
        // Get required permission from route
        let permission = parts.extensions
            .get::<String>()
            .ok_or(AuthzError::MissingPermission)?;
        
        // Check permission
        let authz_service = state.get::<Arc<AuthorizationService>>()
            .ok_or(AuthzError::ServiceError)?;
        
        if !authz_service.has_permission(&auth_context.user_id, permission, None).await? {
            return Err(AuthzError::Forbidden);
        }
        
        Ok(RequirePermission(permission.clone()))
    }
}

/// Role guard
pub struct RequireRole(pub String);

#[async_trait]
impl<S> FromRequestParts<S> for RequireRole
where
    S: Send + Sync,
{
    type Rejection = AuthzError;
    
    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        let auth_context = parts.extensions
            .get::<AuthContext>()
            .ok_or(AuthzError::Unauthorized)?;
        
        let required_role = parts.extensions
            .get::<String>()
            .ok_or(AuthzError::MissingRole)?;
        
        let role_manager = state.get::<Arc<RoleManager>>()
            .ok_or(AuthzError::ServiceError)?;
        
        let user_roles = role_manager.get_user_roles(&auth_context.user_id).await?;
        
        if !user_roles.iter().any(|r| &r.name == required_role) {
            return Err(AuthzError::Forbidden);
        }
        
        Ok(RequireRole(required_role.clone()))
    }
}

/// Resource owner guard
pub struct RequireOwner;

#[async_trait]
impl<S> FromRequestParts<S> for RequireOwner
where
    S: Send + Sync,
{
    type Rejection = AuthzError;
    
    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        let auth_context = parts.extensions
            .get::<AuthContext>()
            .ok_or(AuthzError::Unauthorized)?;
        
        let resource = parts.extensions
            .get::<Resource>()
            .ok_or(AuthzError::MissingResource)?;
        
        if resource.owner_id != Some(auth_context.user_id) {
            return Err(AuthzError::Forbidden);
        }
        
        Ok(RequireOwner)
    }
}
```

## Configuration

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AuthzConfig {
    pub cache_ttl: u64,
    pub enable_policies: bool,
    pub policies: PolicyConfig,
    pub default_deny: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PolicyConfig {
    pub policy_dir: PathBuf,
    pub enable_hot_reload: bool,
    pub evaluation_timeout: u64,
}

impl Default for AuthzConfig {
    fn default() -> Self {
        Self {
            cache_ttl: 300,
            enable_policies: true,
            policies: PolicyConfig {
                policy_dir: PathBuf::from("policies"),
                enable_hot_reload: true,
                evaluation_timeout: 1000,
            },
            default_deny: true,
        }
    }
}
```

## Summary

Diese Authorization Implementation bietet:
- **RBAC System** - Roles, Permissions, User Assignment
- **Policy Engine** - Rego-based Policy Evaluation
- **Resource Registry** - Typed Resource Management
- **Authorization Guards** - Route Protection
- **Caching** - Performance Optimization
- **Wildcard Permissions** - Flexible Permission Matching

Comprehensive Authorization für ReedCMS.