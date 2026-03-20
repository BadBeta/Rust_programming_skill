# Architecture Examples & Extended Patterns

Complete code examples for architecture patterns. For concepts, decision tables, and rules, see [architecture.md](architecture.md).

## DI Containers

For most projects, **manual DI via constructor injection is sufficient**. Use DI containers when you have 20+ services with complex dependency graphs.

### Simple DI Container

Build a basic container using `TypeId` and `Any`:

```rust
use std::any::{Any, TypeId};
use std::collections::HashMap;

pub trait Service: Send + Sync + 'static {}

// Macro to implement Service for types
macro_rules! impl_service {
    ($t:ty) => {
        impl Service for $t {}
    };
}

pub struct DiContainer {
    services: HashMap<TypeId, Box<dyn Any + Send + Sync>>,
}

impl DiContainer {
    pub fn new() -> Self {
        Self { services: HashMap::new() }
    }

    pub fn register<T: Service + 'static>(&mut self, service: T) {
        let type_id = TypeId::of::<T>();
        self.services.insert(type_id, Box::new(service));
    }

    pub fn resolve<T: Service + 'static>(&self) -> Option<&T> {
        let type_id = TypeId::of::<T>();
        self.services
            .get(&type_id)
            .and_then(|boxed| boxed.downcast_ref::<T>())
    }
}

// Usage
struct Logger;
impl_service!(Logger);

impl Logger {
    fn log(&self, msg: &str) { println!("[LOG] {}", msg); }
}

fn main() {
    let mut container = DiContainer::new();
    container.register(Logger);

    if let Some(logger) = container.resolve::<Logger>() {
        logger.log("Hello from DI container");
    }
}
```

### Factory-Based Container

For services with dependencies:

```rust
use std::any::{Any, TypeId};
use std::collections::HashMap;
use std::sync::Arc;

pub struct AdvancedContainer {
    factories: HashMap<TypeId, Box<dyn Fn(&Self) -> Box<dyn Any + Send + Sync>>>,
    singletons: HashMap<TypeId, Arc<dyn Any + Send + Sync>>,
}

impl AdvancedContainer {
    pub fn new() -> Self {
        Self {
            factories: HashMap::new(),
            singletons: HashMap::new(),
        }
    }

    // Register a factory that can resolve its own dependencies
    pub fn register_factory<T, F>(&mut self, factory: F)
    where
        T: Service + 'static,
        F: Fn(&Self) -> T + 'static,
    {
        let type_id = TypeId::of::<T>();
        self.factories.insert(
            type_id,
            Box::new(move |container| Box::new(factory(container))),
        );
    }

    // Register a singleton
    pub fn register_singleton<T: Service + 'static>(&mut self, service: T) {
        let type_id = TypeId::of::<T>();
        self.singletons.insert(type_id, Arc::new(service));
    }

    pub fn resolve<T: Service + 'static>(&self) -> Option<T>
    where
        T: Clone,
    {
        let type_id = TypeId::of::<T>();

        // Check singletons first
        if let Some(arc) = self.singletons.get(&type_id) {
            return arc.downcast_ref::<T>().cloned();
        }

        // Otherwise use factory
        self.factories.get(&type_id).and_then(|factory| {
            let boxed = factory(self);
            boxed.downcast::<T>().ok().map(|b| *b)
        })
    }
}
```

### Shaku Framework Patterns

For larger applications, `shaku` provides a structured approach to compile-time dependency injection:

```rust
// Cargo.toml:
// shaku = { version = "0.6", features = ["derive"] }
// shaku_actix = "0.2"  # For Actix-web integration

use shaku::{module, Component, Interface, HasComponent, Provider};
use std::sync::Arc;

// --- Define Interfaces (Traits) ---

pub trait UserRepository: Interface {
    fn find_by_id(&self, id: u32) -> Option<User>;
    fn save(&self, user: User) -> Result<(), String>;
}

pub trait EmailService: Interface {
    fn send_welcome_email(&self, email: &str) -> Result<(), String>;
}

// --- Implement Components ---

#[derive(Component)]
#[shaku(interface = UserRepository)]
pub struct InMemoryUserRepository {
    // Component parameters can be injected during build
    #[shaku(default)]
    initial_capacity: usize,
}

impl UserRepository for InMemoryUserRepository {
    fn find_by_id(&self, id: u32) -> Option<User> {
        // Implementation...
        None
    }

    fn save(&self, user: User) -> Result<(), String> {
        println!("Saving user to in-memory store");
        Ok(())
    }
}

#[derive(Component)]
#[shaku(interface = EmailService)]
pub struct MockEmailService;

impl EmailService for MockEmailService {
    fn send_welcome_email(&self, email: &str) -> Result<(), String> {
        println!("Mock sending email to: {}", email);
        Ok(())
    }
}

// --- Component with Dependencies ---

pub trait CreateUserUseCase: Interface {
    fn execute(&self, username: String, email: String) -> Result<User, String>;
}

#[derive(Component)]
#[shaku(interface = CreateUserUseCase)]
pub struct CreateUserUseCaseImpl {
    #[shaku(inject)]
    user_repository: Arc<dyn UserRepository>,

    #[shaku(inject)]
    email_service: Arc<dyn EmailService>,
}

impl CreateUserUseCase for CreateUserUseCaseImpl {
    fn execute(&self, username: String, email: String) -> Result<User, String> {
        let user = User { id: 1, username, email: email.clone() };
        self.user_repository.save(user.clone())?;
        self.email_service.send_welcome_email(&email)?;
        Ok(user)
    }
}

// --- Define Module (Composition Root) ---

module! {
    pub AppModule {
        components = [
            InMemoryUserRepository,
            MockEmailService,
            CreateUserUseCaseImpl,
        ],
        providers = []
    }
}

// --- Usage ---

fn main() {
    // Build the module with all dependencies wired
    let module = AppModule::builder()
        // Override default parameters if needed
        .with_component_parameters::<InMemoryUserRepository>(
            InMemoryUserRepositoryParameters { initial_capacity: 100 }
        )
        .build();

    // Resolve dependencies
    let use_case: &dyn CreateUserUseCase = module.resolve_ref();

    match use_case.execute("alice".into(), "alice@example.com".into()) {
        Ok(user) => println!("Created user: {:?}", user),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

### Shaku with Actix-web

```rust
use actix_web::{web, App, HttpServer, HttpResponse};
use shaku_actix::Inject;

// Handler with injected dependencies
async fn create_user_handler(
    use_case: Inject<AppModule, dyn CreateUserUseCase>,
    req: web::Json<CreateUserRequest>,
) -> HttpResponse {
    match use_case.execute(req.username.clone(), req.email.clone()) {
        Ok(user) => HttpResponse::Created().json(user),
        Err(e) => HttpResponse::InternalServerError().body(e),
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Build module once at startup
    let module = Arc::new(AppModule::builder().build());

    HttpServer::new(move || {
        App::new()
            // Share module across all requests
            .app_data(module.clone())
            .route("/users", web::post().to(create_user_handler))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### Shaku Providers for Transient Dependencies

Providers create new instances per resolution (transient scope):

```rust
use shaku::{Provider, Interface};

// Provider trait - creates new instance each time
pub trait DatabaseConnection: Interface {
    fn query(&self, sql: &str) -> Vec<String>;
}

pub struct PooledConnection {
    connection_id: u32,
}

impl DatabaseConnection for PooledConnection {
    fn query(&self, sql: &str) -> Vec<String> {
        println!("Connection {} executing: {}", self.connection_id, sql);
        vec![]
    }
}

// Provider implementation
impl<M: HasComponent<dyn ConnectionPool>> Provider<M> for PooledConnection {
    type Interface = dyn DatabaseConnection;

    fn provide(module: &M) -> Result<Box<Self::Interface>, Box<dyn std::error::Error>> {
        let pool: &dyn ConnectionPool = module.resolve_ref();
        let conn = pool.get_connection()?;
        Ok(Box::new(conn))
    }
}

module! {
    pub DbModule {
        components = [PostgresConnectionPool],
        providers = [PooledConnection]
    }
}

// Each resolution creates a new connection from pool
fn use_database(module: &DbModule) {
    let conn: Box<dyn DatabaseConnection> = module.provide().unwrap();
    conn.query("SELECT * FROM users");
    // Connection returned to pool when dropped
}
```

### Lifetime and Scope Management

**Singleton Scope with `Arc`:**

```rust
use std::sync::Arc;

pub struct AppServices {
    pub config: Arc<Config>,
    pub database: Arc<DatabasePool>,
    pub cache: Arc<CacheService>,
}

impl AppServices {
    pub fn new(config: Config) -> Self {
        let config = Arc::new(config);
        let database = Arc::new(DatabasePool::new(&config.database_url));
        let cache = Arc::new(CacheService::new(&config.cache_url));

        Self { config, database, cache }
    }
}

// Services share the same instances
let services = AppServices::new(config);
let user_service = UserService::new(
    Arc::clone(&services.database),
    Arc::clone(&services.cache),
);
```

**Scoped Lifetime with References:**

```rust
// Service manager with explicit lifetime
struct ServiceManager<'a> {
    logger: &'a dyn Logger,
    processors: Vec<Processor<'a>>,
}

impl<'a> ServiceManager<'a> {
    fn new(logger: &'a dyn Logger) -> Self {
        Self { logger, processors: Vec::new() }
    }

    fn add_processor(&mut self, name: &str) {
        // Processors borrow the same logger
        self.processors.push(Processor::new(self.logger, name));
    }
}

struct Processor<'a> {
    logger: &'a dyn Logger,
    name: String,
}

impl<'a> Processor<'a> {
    fn new(logger: &'a dyn Logger, name: &str) -> Self {
        Self { logger, name: name.to_string() }
    }

    fn process(&self, data: &str) {
        self.logger.log(&format!("{} processing: {}", self.name, data));
    }
}

// Usage - logger must outlive manager
fn main() {
    let console_logger = ConsoleLogger;

    {
        let mut manager = ServiceManager::new(&console_logger);
        manager.add_processor("DataFetcher");
        manager.add_processor("DataTransformer");
        // manager dropped here, but console_logger still valid
    }
}
```

**Interior Mutability for `&self` Methods:**

When trait methods use `&self` but you need to track state:

```rust
use std::cell::RefCell;
use std::sync::{Arc, Mutex};

// Single-threaded: RefCell
pub struct CallTracker {
    calls: RefCell<Vec<String>>,
}

impl CallTracker {
    pub fn new() -> Self {
        Self { calls: RefCell::new(Vec::new()) }
    }

    pub fn record(&self, method: &str) {
        self.calls.borrow_mut().push(method.to_string());
    }

    pub fn get_calls(&self) -> Vec<String> {
        self.calls.borrow().clone()
    }
}

// Multi-threaded: Arc<Mutex<T>>
pub struct ThreadSafeTracker {
    calls: Arc<Mutex<Vec<String>>>,
}

impl ThreadSafeTracker {
    pub fn new() -> Self {
        Self { calls: Arc::new(Mutex::new(Vec::new())) }
    }

    pub fn record(&self, method: &str) {
        self.calls.lock().unwrap().push(method.to_string());
    }
}
```

**Shared Ownership with `Rc<RefCell<T>>`:**

For `&self` trait methods that need mutable state:

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct ConnectionPool {
    connections: Rc<RefCell<Vec<Connection>>>,
}

impl ConnectionPool {
    fn new(size: usize) -> Self {
        let connections = (0..size)
            .map(|_| Connection::new())
            .collect();
        Self {
            connections: Rc::new(RefCell::new(connections)),
        }
    }

    fn get_connection(&self) -> Rc<RefCell<Connection>> {
        // Clone Rc increases reference count, not the data
        Rc::new(RefCell::new(
            self.connections.borrow_mut().pop().unwrap()
        ))
    }
}

// Repository using shared pool
struct UserRepository {
    pool: Rc<ConnectionPool>,
}

impl UserRepository {
    fn new(pool: Rc<ConnectionPool>) -> Self {
        Self { pool }
    }
}
```

### Composition Root Pattern

Wire all dependencies at application startup:

```rust
pub struct AppContainer {
    config: Arc<Config>,
    database: Arc<DatabasePool>,
    user_repository: Arc<dyn UserRepository>,
    order_repository: Arc<dyn OrderRepository>,
    notification_service: Arc<dyn NotificationService>,
    user_service: Arc<UserService>,
    order_service: Arc<OrderService>,
}

impl AppContainer {
    pub fn new(config: Config) -> Self {
        let config = Arc::new(config);

        // Infrastructure layer
        let database = Arc::new(DatabasePool::new(&config.database_url));
        let email_client = Arc::new(SmtpClient::new(&config.smtp_url));

        // Repository implementations
        let user_repository: Arc<dyn UserRepository> =
            Arc::new(PostgresUserRepository::new(Arc::clone(&database)));
        let order_repository: Arc<dyn OrderRepository> =
            Arc::new(PostgresOrderRepository::new(Arc::clone(&database)));

        // Services
        let notification_service: Arc<dyn NotificationService> =
            Arc::new(EmailNotificationService::new(email_client));

        let user_service = Arc::new(UserService::new(
            Arc::clone(&user_repository),
            Arc::clone(&notification_service),
        ));

        let order_service = Arc::new(OrderService::new(
            Arc::clone(&order_repository),
            Arc::clone(&user_repository),
        ));

        Self {
            config,
            database,
            user_repository,
            order_repository,
            notification_service,
            user_service,
            order_service,
        }
    }

    pub fn user_service(&self) -> Arc<UserService> {
        Arc::clone(&self.user_service)
    }

    pub fn order_service(&self) -> Arc<OrderService> {
        Arc::clone(&self.order_service)
    }
}

// Application entry point
#[tokio::main]
async fn main() {
    let config = Config::from_env();
    let container = AppContainer::new(config);

    // Pass services to web handlers, CLI, etc.
    start_web_server(container).await;
}
```


## Domain Modeling Patterns

### Rich Domain Entity with State Machine

```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;

// Value Objects - immutable, identity-less, compared by value
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(pub u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Money {
    amount: Decimal,
    currency: Currency,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Currency { USD, EUR, GBP }

impl Money {
    pub fn new(amount: Decimal, currency: Currency) -> Self {
        Self { amount, currency }
    }

    pub fn add(&self, other: &Money) -> Result<Money, DomainError> {
        if self.currency != other.currency {
            return Err(DomainError::CurrencyMismatch);
        }
        Ok(Money::new(self.amount + other.amount, self.currency))
    }
}

// Entity State Machine - enforces valid transitions
#[derive(Debug, Clone, PartialEq)]
pub enum OrderStatus {
    Pending,
    Processing { started_at: DateTime<Utc> },
    Shipped { tracking_number: String, shipped_at: DateTime<Utc> },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String, cancelled_at: DateTime<Utc> },
}

// Rich Domain Entity
#[derive(Debug, Clone)]
pub struct Order {
    id: OrderId,
    customer_id: u64,
    items: Vec<OrderItem>,
    status: OrderStatus,
    created_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct OrderItem {
    product_id: u64,
    quantity: u32,
    unit_price: Money,
}

impl Order {
    // Factory method with validation
    pub fn new(id: OrderId, customer_id: u64, items: Vec<OrderItem>) -> Result<Self, DomainError> {
        if items.is_empty() {
            return Err(DomainError::EmptyOrder);
        }
        Ok(Self {
            id,
            customer_id,
            items,
            status: OrderStatus::Pending,
            created_at: Utc::now(),
        })
    }

    // Domain logic: calculated property
    pub fn total_price(&self) -> Money {
        self.items.iter().fold(
            Money::new(Decimal::ZERO, Currency::USD),
            |acc, item| {
                let item_total = Money::new(
                    item.unit_price.amount * Decimal::from(item.quantity),
                    item.unit_price.currency,
                );
                acc.add(&item_total).unwrap_or(acc)
            },
        )
    }

    // State transition with validation
    pub fn start_processing(&mut self) -> Result<(), DomainError> {
        match &self.status {
            OrderStatus::Pending => {
                self.status = OrderStatus::Processing { started_at: Utc::now() };
                Ok(())
            }
            _ => Err(DomainError::InvalidStateTransition {
                from: format!("{:?}", self.status),
                to: "Processing".to_string(),
            }),
        }
    }

    pub fn ship(&mut self, tracking_number: String) -> Result<(), DomainError> {
        match &self.status {
            OrderStatus::Processing { .. } => {
                self.status = OrderStatus::Shipped {
                    tracking_number,
                    shipped_at: Utc::now(),
                };
                Ok(())
            }
            _ => Err(DomainError::InvalidStateTransition {
                from: format!("{:?}", self.status),
                to: "Shipped".to_string(),
            }),
        }
    }

    pub fn deliver(&mut self) -> Result<(), DomainError> {
        match &self.status {
            OrderStatus::Shipped { .. } => {
                self.status = OrderStatus::Delivered { delivered_at: Utc::now() };
                Ok(())
            }
            _ => Err(DomainError::InvalidStateTransition {
                from: format!("{:?}", self.status),
                to: "Delivered".to_string(),
            }),
        }
    }

    pub fn cancel(&mut self, reason: String) -> Result<(), DomainError> {
        match &self.status {
            OrderStatus::Pending | OrderStatus::Processing { .. } => {
                self.status = OrderStatus::Cancelled {
                    reason,
                    cancelled_at: Utc::now(),
                };
                Ok(())
            }
            OrderStatus::Shipped { .. } | OrderStatus::Delivered { .. } => {
                Err(DomainError::CannotCancelShippedOrder)
            }
            OrderStatus::Cancelled { .. } => {
                Err(DomainError::OrderAlreadyCancelled)
            }
        }
    }

    // Getters preserve encapsulation
    pub fn id(&self) -> OrderId { self.id }
    pub fn status(&self) -> &OrderStatus { &self.status }
    pub fn items(&self) -> &[OrderItem] { &self.items }
}

#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("Order cannot be empty")]
    EmptyOrder,
    #[error("Currency mismatch")]
    CurrencyMismatch,
    #[error("Invalid state transition from {from} to {to}")]
    InvalidStateTransition { from: String, to: String },
    #[error("Cannot cancel shipped order")]
    CannotCancelShippedOrder,
    #[error("Order already cancelled")]
    OrderAlreadyCancelled,
}
```

### Immutable Entity Pattern (Copy-on-Write)

```rust
// Entities that return new instances instead of mutating
#[derive(Debug, Clone, PartialEq)]
pub struct User {
    id: UserId,
    username: String,
    email: String,
}

impl User {
    pub fn new(id: UserId, username: String, email: String) -> Self {
        Self { id, username, email }
    }

    // Returns new instance - original unchanged
    pub fn change_email(&self, new_email: String) -> Result<User, ValidationError> {
        if !new_email.contains('@') {
            return Err(ValidationError::InvalidEmail);
        }
        Ok(User {
            email: new_email,
            ..self.clone()  // Copy all other fields
        })
    }

    pub fn change_username(&self, new_username: String) -> Result<User, ValidationError> {
        if new_username.is_empty() {
            return Err(ValidationError::EmptyUsername);
        }
        Ok(User {
            username: new_username,
            ..self.clone()
        })
    }

    pub fn id(&self) -> &UserId { &self.id }
    pub fn email(&self) -> &str { &self.email }
    pub fn username(&self) -> &str { &self.username }
}

// Usage: original preserved for audit trail
fn update_user_email(user: User, new_email: String) -> Result<(User, User), ValidationError> {
    let original = user.clone();
    let updated = user.change_email(new_email)?;
    Ok((original, updated))  // Both versions available
}
```

### Document State Machine Pattern

```rust
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, PartialEq)]
pub enum DocumentState {
    Draft,
    Published { published_at: DateTime<Utc> },
    Archived { archived_at: DateTime<Utc> },
}

#[derive(Debug, Clone)]
pub struct Document {
    id: u32,
    title: String,
    content: String,
    state: DocumentState,
}

impl Document {
    pub fn new(id: u32, title: String) -> Self {
        Self {
            id,
            title,
            content: String::new(),
            state: DocumentState::Draft,
        }
    }

    // Can only edit drafts
    pub fn update_content(&mut self, content: String) -> Result<(), DocumentError> {
        match &self.state {
            DocumentState::Draft => {
                self.content = content;
                Ok(())
            }
            _ => Err(DocumentError::CannotEditNonDraft),
        }
    }

    pub fn publish(&mut self) -> Result<(), DocumentError> {
        match &self.state {
            DocumentState::Draft => {
                if self.content.is_empty() {
                    return Err(DocumentError::CannotPublishEmpty);
                }
                self.state = DocumentState::Published { published_at: Utc::now() };
                Ok(())
            }
            DocumentState::Published { .. } => Err(DocumentError::AlreadyPublished),
            DocumentState::Archived { .. } => Err(DocumentError::CannotPublishArchived),
        }
    }

    pub fn archive(&mut self) -> Result<(), DocumentError> {
        match &self.state {
            DocumentState::Archived { .. } => Err(DocumentError::AlreadyArchived),
            _ => {
                self.state = DocumentState::Archived { archived_at: Utc::now() };
                Ok(())
            }
        }
    }

    pub fn state(&self) -> &DocumentState { &self.state }
    pub fn content(&self) -> &str { &self.content }
}

#[derive(Debug, thiserror::Error)]
pub enum DocumentError {
    #[error("Cannot edit non-draft document")]
    CannotEditNonDraft,
    #[error("Cannot publish empty document")]
    CannotPublishEmpty,
    #[error("Document already published")]
    AlreadyPublished,
    #[error("Cannot publish archived document")]
    CannotPublishArchived,
    #[error("Document already archived")]
    AlreadyArchived,
}
```

### Use Case with DTOs (Data Transfer Objects)

```rust
use async_trait::async_trait;

// Input DTO - what the use case receives
#[derive(Debug, Clone)]
pub struct DepositInput {
    pub account_id: String,
    pub amount: u64,
}

// Output DTO - what the use case returns
#[derive(Debug, Clone)]
pub struct DepositOutput {
    pub account_id: String,
    pub new_balance: u64,
    pub transaction_id: String,
}

// Use case error enum
#[derive(Debug, thiserror::Error)]
pub enum DepositError {
    #[error("Invalid account ID format")]
    InvalidAccountId,
    #[error("Account not found: {0}")]
    AccountNotFound(String),
    #[error("Deposit amount cannot be zero")]
    ZeroAmount,
    #[error("Balance overflow")]
    BalanceOverflow,
    #[error("Repository error: {0}")]
    Repository(#[from] RepositoryError),
}

// Use Case / Interactor
pub struct DepositUseCase<R: AccountRepository> {
    repository: R,
}

impl<R: AccountRepository> DepositUseCase<R> {
    pub fn new(repository: R) -> Self {
        Self { repository }
    }

    pub async fn execute(&mut self, input: DepositInput) -> Result<DepositOutput, DepositError> {
        // Input validation
        if input.amount == 0 {
            return Err(DepositError::ZeroAmount);
        }

        let account_id = input.account_id.parse::<u64>()
            .map_err(|_| DepositError::InvalidAccountId)?;

        // Fetch entity through repository
        let mut account = self.repository
            .find_by_id(account_id)
            .await?
            .ok_or_else(|| DepositError::AccountNotFound(input.account_id.clone()))?;

        // Domain logic on entity
        account.deposit(input.amount)
            .map_err(|_| DepositError::BalanceOverflow)?;

        // Persist changes
        self.repository.save(&account).await?;

        // Return output DTO
        Ok(DepositOutput {
            account_id: input.account_id,
            new_balance: account.balance(),
            transaction_id: uuid::Uuid::new_v4().to_string(),
        })
    }
}

// Repository trait (Port)
#[async_trait]
pub trait AccountRepository: Send + Sync {
    async fn find_by_id(&self, id: u64) -> Result<Option<Account>, RepositoryError>;
    async fn save(&mut self, account: &Account) -> Result<(), RepositoryError>;
}
```

### Sensitive Data in DTOs

Protect sensitive data from accidental logging or serialization:

```rust
use serde::{Serialize, Serializer, Deserialize, Deserializer};

/// Password newtype that redacts on serialization
#[derive(Clone)]
pub struct Password(String);

impl Password {
    pub fn new(value: String) -> Self {
        Self(value)
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

// Redact when serializing (for logs, API responses)
impl Serialize for Password {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str("[REDACTED]")
    }
}

// Allow deserializing from JSON input
impl<'de> Deserialize<'de> for Password {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        Ok(Password(s))
    }
}

// Redact in Debug output
impl std::fmt::Debug for Password {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str("Password([REDACTED])")
    }
}

// Usage in DTOs
#[derive(Debug, Deserialize)]
pub struct LoginInput {
    pub username: String,
    pub password: Password,  // Safe to log this struct
}

#[derive(Debug, Serialize)]
pub struct UserProfile {
    pub id: u64,
    pub username: String,
    #[serde(skip_serializing)]  // Alternative: skip entirely
    pub api_key: Option<String>,
}

// Safe logging - password is redacted
fn handle_login(input: LoginInput) {
    tracing::info!("Login attempt: {:?}", input);
    // Logs: Login attempt: LoginInput { username: "alice", password: Password([REDACTED]) }

    // Access actual value when needed
    let actual_password = input.password.as_str();
}
```

### Mock Repository for Testing

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use async_trait::async_trait;

// Mock implementation for unit testing
pub struct MockAccountRepository {
    accounts: Arc<Mutex<HashMap<u64, Account>>>,
    save_calls: Arc<Mutex<Vec<Account>>>,  // Track save calls for assertions
}

impl MockAccountRepository {
    pub fn new() -> Self {
        Self {
            accounts: Arc::new(Mutex::new(HashMap::new())),
            save_calls: Arc::new(Mutex::new(Vec::new())),
        }
    }

    // Setup helper for tests
    pub fn with_account(self, account: Account) -> Self {
        self.accounts.lock().unwrap().insert(account.id(), account);
        self
    }

    // Assertion helper
    pub fn save_was_called_with(&self, account_id: u64) -> bool {
        self.save_calls.lock().unwrap()
            .iter()
            .any(|a| a.id() == account_id)
    }

    pub fn get_saved_account(&self, account_id: u64) -> Option<Account> {
        self.save_calls.lock().unwrap()
            .iter()
            .find(|a| a.id() == account_id)
            .cloned()
    }
}

#[async_trait]
impl AccountRepository for MockAccountRepository {
    async fn find_by_id(&self, id: u64) -> Result<Option<Account>, RepositoryError> {
        Ok(self.accounts.lock().unwrap().get(&id).cloned())
    }

    async fn save(&mut self, account: &Account) -> Result<(), RepositoryError> {
        self.save_calls.lock().unwrap().push(account.clone());
        self.accounts.lock().unwrap().insert(account.id(), account.clone());
        Ok(())
    }
}

// Example test
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn deposit_increases_balance() {
        // Arrange
        let initial_account = Account::new(1, 100);
        let mut repo = MockAccountRepository::new().with_account(initial_account);
        let mut use_case = DepositUseCase::new(repo);

        // Act
        let result = use_case.execute(DepositInput {
            account_id: "1".to_string(),
            amount: 50,
        }).await;

        // Assert
        assert!(result.is_ok());
        let output = result.unwrap();
        assert_eq!(output.new_balance, 150);
    }

    #[tokio::test]
    async fn deposit_zero_returns_error() {
        let repo = MockAccountRepository::new();
        let mut use_case = DepositUseCase::new(repo);

        let result = use_case.execute(DepositInput {
            account_id: "1".to_string(),
            amount: 0,
        }).await;

        assert!(matches!(result, Err(DepositError::ZeroAmount)));
    }
}
```

### Presenter and View Model Pattern

The presentation layer transforms domain data into UI-specific formats. This keeps domain entities clean and provides UI-optimized data structures.

```rust
use serde::Serialize;
use chrono::{DateTime, Utc};

// Domain entity (from domain layer)
pub struct User {
    id: UserId,
    first_name: String,
    last_name: String,
    email: String,
    created_at: DateTime<Utc>,
    last_login: Option<DateTime<Utc>>,
    role: UserRole,
}

// View Model - optimized for UI display
#[derive(Debug, Clone, Serialize)]
pub struct UserProfileViewModel {
    pub id: String,
    pub display_name: String,
    pub email: String,
    pub member_since: String,
    pub last_active: String,
    pub role_badge: RoleBadge,
    pub initials: String,
}

#[derive(Debug, Clone, Serialize)]
pub struct RoleBadge {
    pub label: String,
    pub color: String,
}

impl From<&User> for UserProfileViewModel {
    fn from(user: &User) -> Self {
        Self {
            id: user.id.0.to_string(),
            display_name: format!("{} {}", user.first_name, user.last_name),
            email: user.email.clone(),
            member_since: user.created_at.format("%B %Y").to_string(),
            last_active: format_last_active(user.last_login),
            role_badge: RoleBadge::from(&user.role),
            initials: format!(
                "{}{}",
                user.first_name.chars().next().unwrap_or_default(),
                user.last_name.chars().next().unwrap_or_default()
            ),
        }
    }
}

fn format_last_active(last_login: Option<DateTime<Utc>>) -> String {
    match last_login {
        Some(dt) => {
            let duration = Utc::now().signed_duration_since(dt);
            if duration.num_minutes() < 5 {
                "Online now".to_string()
            } else if duration.num_hours() < 1 {
                format!("{} minutes ago", duration.num_minutes())
            } else if duration.num_hours() < 24 {
                format!("{} hours ago", duration.num_hours())
            } else {
                format!("{} days ago", duration.num_days())
            }
        }
        None => "Never".to_string(),
    }
}

impl From<&UserRole> for RoleBadge {
    fn from(role: &UserRole) -> Self {
        match role {
            UserRole::Admin => RoleBadge {
                label: "Admin".to_string(),
                color: "red".to_string(),
            },
            UserRole::Moderator => RoleBadge {
                label: "Mod".to_string(),
                color: "blue".to_string(),
            },
            UserRole::User => RoleBadge {
                label: "Member".to_string(),
                color: "gray".to_string(),
            },
        }
    }
}
```

**Presenter Pattern (MVP):**

```rust
// Presenter trait - transforms domain to view
pub trait UserPresenter {
    fn present_profile(&self, user: &User) -> UserProfileViewModel;
    fn present_list(&self, users: &[User]) -> UserListViewModel;
    fn present_error(&self, error: &UserError) -> ErrorViewModel;
}

// Concrete presenter implementation
pub struct WebUserPresenter;

impl UserPresenter for WebUserPresenter {
    fn present_profile(&self, user: &User) -> UserProfileViewModel {
        UserProfileViewModel::from(user)
    }

    fn present_list(&self, users: &[User]) -> UserListViewModel {
        UserListViewModel {
            users: users.iter().map(UserSummaryViewModel::from).collect(),
            total_count: users.len(),
            empty_message: if users.is_empty() {
                Some("No users found".to_string())
            } else {
                None
            },
        }
    }

    fn present_error(&self, error: &UserError) -> ErrorViewModel {
        match error {
            UserError::NotFound => ErrorViewModel {
                title: "User Not Found".to_string(),
                message: "The requested user could not be found.".to_string(),
                code: "USER_NOT_FOUND".to_string(),
                recoverable: false,
            },
            UserError::ValidationFailed(msg) => ErrorViewModel {
                title: "Validation Error".to_string(),
                message: msg.clone(),
                code: "VALIDATION_ERROR".to_string(),
                recoverable: true,
            },
            _ => ErrorViewModel {
                title: "Error".to_string(),
                message: "An unexpected error occurred.".to_string(),
                code: "INTERNAL_ERROR".to_string(),
                recoverable: false,
            },
        }
    }
}

#[derive(Debug, Serialize)]
pub struct UserListViewModel {
    pub users: Vec<UserSummaryViewModel>,
    pub total_count: usize,
    pub empty_message: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct UserSummaryViewModel {
    pub id: String,
    pub name: String,
    pub avatar_url: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct ErrorViewModel {
    pub title: String,
    pub message: String,
    pub code: String,
    pub recoverable: bool,
}
```

**Using Presenters in Handlers:**

```rust
use axum::{extract::Path, Json};

async fn get_user_profile(
    Path(user_id): Path<String>,
    service: Extension<Arc<dyn UserService>>,
    presenter: Extension<Arc<dyn UserPresenter>>,
) -> Result<Json<UserProfileViewModel>, ApiError> {
    let user = service.get_user(&user_id).await?;

    // Presenter transforms domain to view model
    let view_model = presenter.present_profile(&user);

    Ok(Json(view_model))
}

async fn list_users(
    service: Extension<Arc<dyn UserService>>,
    presenter: Extension<Arc<dyn UserPresenter>>,
) -> Result<Json<UserListViewModel>, ApiError> {
    let users = service.list_users().await?;

    // Presenter handles the transformation
    let view_model = presenter.present_list(&users);

    Ok(Json(view_model))
}
```

## Resilience Patterns

### Retry with Exponential Backoff and Jitter

```rust
use std::time::Duration;
use tokio::time::sleep;
use rand::Rng;

pub struct RetryConfig {
    pub max_retries: u32,
    pub initial_delay: Duration,
    pub backoff_factor: f64,
    pub jitter_factor: f64,  // 0.0 to 1.0
}

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_retries: 3,
            initial_delay: Duration::from_millis(100),
            backoff_factor: 2.0,
            jitter_factor: 0.5,
        }
    }
}

pub async fn retry_with_backoff<T, E, F, Fut>(
    config: &RetryConfig,
    should_retry: impl Fn(&E) -> bool,
    mut operation: F,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut current_delay = config.initial_delay;
    let mut rng = rand::thread_rng();

    for attempt in 0..=config.max_retries {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(err) if attempt < config.max_retries && should_retry(&err) => {
                // Calculate delay with jitter
                let jitter = current_delay.as_secs_f64()
                    * config.jitter_factor
                    * rng.gen::<f64>();
                let actual_delay = Duration::from_secs_f64(
                    current_delay.as_secs_f64() + jitter
                );

                eprintln!(
                    "Attempt {} failed, retrying in {:?}...",
                    attempt + 1, actual_delay
                );
                sleep(actual_delay).await;

                // Apply exponential backoff
                current_delay = Duration::from_secs_f64(
                    current_delay.as_secs_f64() * config.backoff_factor
                );
            }
            Err(err) => return Err(err),
        }
    }
    unreachable!()
}

// Usage
async fn fetch_with_retry(url: &str) -> Result<String, reqwest::Error> {
    let config = RetryConfig::default();

    retry_with_backoff(
        &config,
        |err: &reqwest::Error| err.is_timeout() || err.is_connect(),
        || async { reqwest::get(url).await?.text().await }
    ).await
}
```

### Retryable Trait Abstraction

For more structured retry logic, use a trait-based approach:

```rust
use std::future::Future;
use std::time::Duration;
use rand::Rng;

/// Trait for operations that can be retried
pub trait Retryable {
    type Output;
    type Error;

    fn perform(&self) -> impl Future<Output = Result<Self::Output, Self::Error>> + Send;

    fn should_retry(&self, error: &Self::Error) -> bool;
}

/// Executor that handles retry logic with configurable strategies
pub struct RetryExecutor {
    max_retries: u32,
    initial_delay: Duration,
    max_delay: Duration,
    strategy: BackoffStrategy,
}

#[derive(Clone, Copy)]
pub enum BackoffStrategy {
    /// delay = initial * factor^attempt + jitter
    Exponential { factor: f64, jitter: f64 },
    /// delay = random(0, min(cap, initial * 2^attempt))
    FullJitter,
    /// delay = min(cap, random(initial, previous * 3))
    DecorrelatedJitter,
}

impl RetryExecutor {
    pub fn new(max_retries: u32) -> Self {
        Self {
            max_retries,
            initial_delay: Duration::from_millis(100),
            max_delay: Duration::from_secs(30),
            strategy: BackoffStrategy::Exponential { factor: 2.0, jitter: 0.5 },
        }
    }

    pub fn with_decorrelated_jitter(mut self) -> Self {
        self.strategy = BackoffStrategy::DecorrelatedJitter;
        self
    }

    pub async fn execute<R: Retryable>(&self, retryable: R) -> Result<R::Output, R::Error> {
        let mut rng = rand::thread_rng();
        let mut previous_delay = self.initial_delay;

        for attempt in 0..=self.max_retries {
            match retryable.perform().await {
                Ok(result) => return Ok(result),
                Err(err) if attempt < self.max_retries && retryable.should_retry(&err) => {
                    let delay = self.calculate_delay(attempt, previous_delay, &mut rng);
                    previous_delay = delay;
                    tokio::time::sleep(delay).await;
                }
                Err(err) => return Err(err),
            }
        }
        unreachable!()
    }

    fn calculate_delay(&self, attempt: u32, previous: Duration, rng: &mut impl Rng) -> Duration {
        let delay = match self.strategy {
            BackoffStrategy::Exponential { factor, jitter } => {
                let base = self.initial_delay.as_secs_f64() * factor.powi(attempt as i32);
                let jittered = base + (base * jitter * rng.gen::<f64>());
                Duration::from_secs_f64(jittered)
            }
            BackoffStrategy::FullJitter => {
                let cap = self.initial_delay.as_secs_f64() * 2.0_f64.powi(attempt as i32);
                let capped = cap.min(self.max_delay.as_secs_f64());
                Duration::from_secs_f64(rng.gen::<f64>() * capped)
            }
            BackoffStrategy::DecorrelatedJitter => {
                let next = previous.as_secs_f64() * 3.0;
                let capped = next.min(self.max_delay.as_secs_f64());
                let delay = self.initial_delay.as_secs_f64()
                    + rng.gen::<f64>() * (capped - self.initial_delay.as_secs_f64());
                Duration::from_secs_f64(delay)
            }
        };
        delay.min(self.max_delay)
    }
}

// Example: HTTP fetch as Retryable
struct HttpFetch {
    client: reqwest::Client,
    url: String,
}

impl Retryable for HttpFetch {
    type Output = String;
    type Error = reqwest::Error;

    async fn perform(&self) -> Result<Self::Output, Self::Error> {
        self.client.get(&self.url).send().await?.text().await
    }

    fn should_retry(&self, error: &Self::Error) -> bool {
        error.is_timeout() || error.is_connect()
    }
}

// Usage
async fn fetch_with_retryable(url: &str) -> Result<String, reqwest::Error> {
    let executor = RetryExecutor::new(3).with_decorrelated_jitter();
    let fetch = HttpFetch {
        client: reqwest::Client::new(),
        url: url.to_string(),
    };
    executor.execute(fetch).await
}
```

### Circuit Breaker Pattern

```rust
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};

#[derive(Debug, Clone, PartialEq)]
pub enum CircuitState {
    Closed,
    Open,
    HalfOpen,
}

pub struct CircuitBreaker {
    state: Mutex<CircuitState>,
    failure_count: Mutex<usize>,
    last_failure: Mutex<Option<Instant>>,
    failure_threshold: usize,
    success_threshold: usize,
    timeout: Duration,
}

impl CircuitBreaker {
    pub fn new(failure_threshold: usize, success_threshold: usize, timeout: Duration) -> Self {
        Self {
            state: Mutex::new(CircuitState::Closed),
            failure_count: Mutex::new(0),
            last_failure: Mutex::new(None),
            failure_threshold,
            success_threshold,
            timeout,
        }
    }

    pub fn call<T, E>(&self, operation: impl FnOnce() -> Result<T, E>) -> Result<T, CircuitError<E>> {
        // Check if circuit should transition from Open to HalfOpen
        {
            let state = self.state.lock().unwrap();
            if *state == CircuitState::Open {
                let last = self.last_failure.lock().unwrap();
                if let Some(last_failure) = *last {
                    if last_failure.elapsed() < self.timeout {
                        return Err(CircuitError::Open);
                    }
                }
                drop(state);
                *self.state.lock().unwrap() = CircuitState::HalfOpen;
            }
        }

        // Execute the operation
        match operation() {
            Ok(result) => {
                self.on_success();
                Ok(result)
            }
            Err(err) => {
                self.on_failure();
                Err(CircuitError::OperationFailed(err))
            }
        }
    }

    fn on_success(&self) {
        let mut state = self.state.lock().unwrap();
        if *state == CircuitState::HalfOpen {
            *state = CircuitState::Closed;
        }
        *self.failure_count.lock().unwrap() = 0;
    }

    fn on_failure(&self) {
        let mut count = self.failure_count.lock().unwrap();
        *count += 1;
        *self.last_failure.lock().unwrap() = Some(Instant::now());

        if *count >= self.failure_threshold {
            *self.state.lock().unwrap() = CircuitState::Open;
        }
    }

    pub fn state(&self) -> CircuitState {
        self.state.lock().unwrap().clone()
    }
}

#[derive(Debug)]
pub enum CircuitError<E> {
    Open,
    OperationFailed(E),
}

// Usage
fn main() {
    let breaker = Arc::new(CircuitBreaker::new(3, 1, Duration::from_secs(30)));

    for i in 0..10 {
        let result = breaker.call(|| {
            if i % 2 == 0 { Ok("success") }
            else { Err("simulated failure") }
        });

        println!("Call {}: {:?}, State: {:?}", i, result, breaker.state());
    }
}
```

### Graceful Degradation

```rust
// Design services with optional dependencies
pub struct UserService<Repo, ImageSvc>
where
    Repo: UserRepository,
    ImageSvc: ImageService,
{
    user_repo: Repo,
    image_service: ImageSvc,
}

impl<Repo, ImageSvc> UserService<Repo, ImageSvc>
where
    Repo: UserRepository,
    ImageSvc: ImageService,
{
    pub async fn get_user_profile(&self, id: UserId) -> Result<UserProfile, UserError> {
        // Core functionality - must succeed
        let user = self.user_repo.find_by_id(id)?;

        // Optional enhancement - degrade gracefully
        let avatar_url = match self.image_service.get_avatar(&user.id).await {
            Ok(url) => Some(url),
            Err(ImageError::ServiceUnavailable) => {
                // Log warning but continue
                eprintln!("Image service unavailable, using default avatar");
                None
            }
            Err(e) => return Err(UserError::from(e)),
        };

        Ok(UserProfile {
            user,
            avatar_url,
        })
    }
}
```


## Nanoservices Architecture

Structure Rust projects so modules can work as independent microservices OR be compiled into a single binary ("nanoservices"). This provides deployment flexibility while maintaining code isolation.

### Project Structure

```
my_app/
├── Cargo.toml              # Workspace root
├── services/
│   ├── api_gateway/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs      # Library for embedding
│   │       └── main.rs     # Standalone binary
│   ├── user_service/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       └── main.rs
│   └── order_service/
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           └── main.rs
├── shared/
│   ├── domain/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs      # Shared domain types
│   └── dal/
│       ├── Cargo.toml
│       └── src/lib.rs      # Data access layer
└── monolith/
    ├── Cargo.toml          # Combines all services
    └── src/main.rs
```

### Workspace Configuration

```toml
# Root Cargo.toml
[workspace]
resolver = "2"
members = [
    "services/api_gateway",
    "services/user_service",
    "services/order_service",
    "shared/domain",
    "shared/dal",
    "monolith",
]

# Shared dependencies for consistency
[workspace.dependencies]
tokio = { version = "1.36", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
tracing = "0.1"
```

### Service as Library + Binary

```toml
# services/user_service/Cargo.toml
[package]
name = "user-service"
version = "0.1.0"
edition = "2021"

[lib]
name = "user_service"
path = "src/lib.rs"

[[bin]]
name = "user-service"
path = "src/main.rs"

[dependencies]
domain = { path = "../../shared/domain" }
dal = { path = "../../shared/dal" }
tokio.workspace = true
serde.workspace = true
```

```rust
// services/user_service/src/lib.rs
use domain::User;

pub struct UserService {
    // Service state
}

impl UserService {
    pub fn new() -> Self {
        Self {}
    }

    pub async fn get_user(&self, id: u64) -> Option<User> {
        // Business logic
        None
    }

    pub async fn create_user(&self, name: &str) -> User {
        User { id: 1, name: name.to_string() }
    }
}

// Optional: HTTP handlers for standalone mode
#[cfg(feature = "http")]
pub mod http {
    use super::*;

    pub async fn run_server(service: UserService, port: u16) {
        // Start HTTP server
    }
}
```

```rust
// services/user_service/src/main.rs
use user_service::UserService;

#[tokio::main]
async fn main() {
    let service = UserService::new();
    // Run as standalone microservice
    user_service::http::run_server(service, 8081).await;
}
```

### Monolith Composition

```toml
# monolith/Cargo.toml
[package]
name = "monolith"
version = "0.1.0"
edition = "2021"

[dependencies]
user-service = { path = "../services/user_service" }
order-service = { path = "../services/order_service" }
api-gateway = { path = "../services/api_gateway" }
tokio.workspace = true
```

```rust
// monolith/src/main.rs
use user_service::UserService;
use order_service::OrderService;

#[tokio::main]
async fn main() {
    // All services in one process, communicating in-memory
    let user_svc = UserService::new();
    let order_svc = OrderService::new();

    // Run combined server with all routes
    run_combined_server(user_svc, order_svc, 8080).await;
}

async fn run_combined_server(
    user_svc: UserService,
    order_svc: OrderService,
    port: u16,
) {
    // Single HTTP server handling all service routes
}
```

### Feature-Gated Data Access Layer

Use Cargo features to swap storage backends without changing application code:

```toml
# shared/dal/Cargo.toml
[package]
name = "dal"
version = "0.1.0"
edition = "2021"

[features]
default = ["json-file"]
json-file = ["serde_json"]
postgres = ["sqlx", "sqlx/postgres"]
sqlite = ["sqlx", "sqlx/sqlite"]

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = { version = "1.0", optional = true }
sqlx = { version = "0.7", optional = true }
```

```rust
// shared/dal/src/lib.rs
use serde::{de::DeserializeOwned, Serialize};

// Common trait for all storage backends
pub trait Storage: Send + Sync {
    type Error: std::error::Error + Send + Sync + 'static;

    fn get<T: DeserializeOwned>(&self, key: &str) -> Result<Option<T>, Self::Error>;
    fn set<T: Serialize>(&self, key: &str, value: &T) -> Result<(), Self::Error>;
    fn delete(&self, key: &str) -> Result<(), Self::Error>;
    fn list_keys(&self, prefix: &str) -> Result<Vec<String>, Self::Error>;
}

// Feature-gated implementations
#[cfg(feature = "json-file")]
pub mod json_file;

#[cfg(feature = "postgres")]
pub mod postgres;

#[cfg(feature = "sqlite")]
pub mod sqlite;

// Re-export based on features
#[cfg(feature = "json-file")]
pub use json_file::JsonFileStorage;

#[cfg(feature = "postgres")]
pub use postgres::PostgresStorage;

#[cfg(feature = "sqlite")]
pub use sqlite::SqliteStorage;
```

### JSON File Storage Implementation

```rust
// shared/dal/src/json_file.rs
use super::Storage;
use serde::{de::DeserializeOwned, Serialize};
use std::collections::HashMap;
use std::fs::{self, File};
use std::io::{Read, Write};
use std::path::PathBuf;

#[derive(Debug)]
pub struct JsonFileError(String);

impl std::fmt::Display for JsonFileError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::error::Error for JsonFileError {}

pub struct JsonFileStorage {
    path: PathBuf,
}

impl JsonFileStorage {
    pub fn new(path: impl Into<PathBuf>) -> Self {
        Self { path: path.into() }
    }

    fn read_store(&self) -> Result<HashMap<String, serde_json::Value>, JsonFileError> {
        if !self.path.exists() {
            return Ok(HashMap::new());
        }
        let mut file = File::open(&self.path)
            .map_err(|e| JsonFileError(e.to_string()))?;
        let mut contents = String::new();
        file.read_to_string(&mut contents)
            .map_err(|e| JsonFileError(e.to_string()))?;
        serde_json::from_str(&contents)
            .map_err(|e| JsonFileError(e.to_string()))
    }

    fn write_store(&self, store: &HashMap<String, serde_json::Value>) -> Result<(), JsonFileError> {
        let json = serde_json::to_string_pretty(store)
            .map_err(|e| JsonFileError(e.to_string()))?;
        let mut file = File::create(&self.path)
            .map_err(|e| JsonFileError(e.to_string()))?;
        file.write_all(json.as_bytes())
            .map_err(|e| JsonFileError(e.to_string()))
    }
}

impl Storage for JsonFileStorage {
    type Error = JsonFileError;

    fn get<T: DeserializeOwned>(&self, key: &str) -> Result<Option<T>, Self::Error> {
        let store = self.read_store()?;
        match store.get(key) {
            Some(value) => {
                let typed: T = serde_json::from_value(value.clone())
                    .map_err(|e| JsonFileError(e.to_string()))?;
                Ok(Some(typed))
            }
            None => Ok(None),
        }
    }

    fn set<T: Serialize>(&self, key: &str, value: &T) -> Result<(), Self::Error> {
        let mut store = self.read_store()?;
        let json_value = serde_json::to_value(value)
            .map_err(|e| JsonFileError(e.to_string()))?;
        store.insert(key.to_string(), json_value);
        self.write_store(&store)
    }

    fn delete(&self, key: &str) -> Result<(), Self::Error> {
        let mut store = self.read_store()?;
        store.remove(key);
        self.write_store(&store)
    }

    fn list_keys(&self, prefix: &str) -> Result<Vec<String>, Self::Error> {
        let store = self.read_store()?;
        Ok(store.keys()
            .filter(|k| k.starts_with(prefix))
            .cloned()
            .collect())
    }
}
```

### Application Using Feature-Gated DAL

```toml
# services/user_service/Cargo.toml
[features]
default = ["json-storage"]
json-storage = ["dal/json-file"]
postgres-storage = ["dal/postgres"]

[dependencies]
dal = { path = "../../shared/dal", default-features = false }
```

```rust
// services/user_service/src/lib.rs
use dal::Storage;
use domain::User;

pub struct UserService<S: Storage> {
    storage: S,
}

impl<S: Storage> UserService<S> {
    pub fn new(storage: S) -> Self {
        Self { storage }
    }

    pub fn get_user(&self, id: u64) -> Result<Option<User>, S::Error> {
        self.storage.get(&format!("user:{}", id))
    }

    pub fn save_user(&self, user: &User) -> Result<(), S::Error> {
        self.storage.set(&format!("user:{}", user.id), user)
    }
}

// Factory function selects backend based on features
#[cfg(feature = "json-storage")]
pub fn create_default_storage() -> dal::JsonFileStorage {
    dal::JsonFileStorage::new("./data/users.json")
}

#[cfg(feature = "postgres-storage")]
pub async fn create_default_storage() -> dal::PostgresStorage {
    dal::PostgresStorage::connect("postgres://localhost/mydb").await.unwrap()
}
```

### Build Commands

```bash
# Build with JSON file storage (default)
cargo build -p user-service

# Build with Postgres storage
cargo build -p user-service --no-default-features --features postgres-storage

# Build monolith with all services using Postgres
cargo build -p monolith --features "user-service/postgres-storage order-service/postgres-storage"

# Run tests with specific storage backend
cargo test -p dal --features json-file
cargo test -p dal --features postgres
```

### Glue Module Pattern

The glue module provides shared types used across workspace boundaries:

```rust
// glue/src/errors.rs
use thiserror::Error;

/// Error status codes that map to HTTP status
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum NanoServiceErrorStatus {
    NotFound,
    BadRequest,
    Unauthorized,
    Forbidden,
    Conflict,
    InternalError,
}

/// Unified error type for cross-workspace communication
#[derive(Debug, Error)]
#[error("{message}")]
pub struct NanoServiceError {
    pub message: String,
    pub status: NanoServiceErrorStatus,
}

impl NanoServiceError {
    pub fn new(message: String, status: NanoServiceErrorStatus) -> Self {
        Self { message, status }
    }

    pub fn not_found(entity: &str) -> Self {
        Self::new(
            format!("{} not found", entity),
            NanoServiceErrorStatus::NotFound,
        )
    }

    pub fn bad_request(message: impl Into<String>) -> Self {
        Self::new(message.into(), NanoServiceErrorStatus::BadRequest)
    }

    pub fn unauthorized(message: impl Into<String>) -> Self {
        Self::new(message.into(), NanoServiceErrorStatus::Unauthorized)
    }
}

// Convert from domain errors
impl From<std::io::Error> for NanoServiceError {
    fn from(err: std::io::Error) -> Self {
        Self::new(err.to_string(), NanoServiceErrorStatus::InternalError)
    }
}

impl From<serde_json::Error> for NanoServiceError {
    fn from(err: serde_json::Error) -> Self {
        Self::new(err.to_string(), NanoServiceErrorStatus::BadRequest)
    }
}
```

### Glue Token for Authentication

```rust
// glue/src/token.rs

/// Authentication token shared across layers
#[derive(Debug, Clone)]
pub struct HeaderToken {
    pub token: String,
}

impl HeaderToken {
    pub fn new(token: String) -> Self {
        Self { token }
    }

    /// Validate token format (basic validation)
    pub fn validate(&self) -> Result<(), NanoServiceError> {
        if self.token.is_empty() {
            return Err(NanoServiceError::unauthorized("Empty token"));
        }
        if self.token.len() < 10 {
            return Err(NanoServiceError::unauthorized("Invalid token format"));
        }
        Ok(())
    }
}
```

### safe_eject! Macro

Helper macro for consistent error mapping:

```rust
// glue/src/lib.rs
pub mod errors;
pub mod token;

pub use errors::{NanoServiceError, NanoServiceErrorStatus};
pub use token::HeaderToken;

/// Maps errors to NanoServiceError with consistent formatting
#[macro_export]
macro_rules! safe_eject {
    // Basic: map error to status
    ($e:expr, $err_status:expr) => {
        $e.map_err(|x| $crate::NanoServiceError::new(
            x.to_string(),
            $err_status
        ))
    };
    // With context: add prefix to error message
    ($e:expr, $err_status:expr, $context:expr) => {
        $e.map_err(|x| $crate::NanoServiceError::new(
            format!("{}: {}", $context, x),
            $err_status
        ))
    };
}

// Usage in dal layer:
// safe_eject!(
//     serde_json::from_str::<Vec<Item>>(&data),
//     NanoServiceErrorStatus::InternalError,
//     "Failed to parse items"
// )?;
```

### Core Layer (Business Logic)

```rust
// core/src/api.rs
use glue::NanoServiceError;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Item {
    pub name: String,
    pub status: ItemStatus,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ItemStatus {
    Pending,
    Active,
    Completed,
}

/// Core API trait - implemented by DAL, used by networking
pub trait ItemStore: Send + Sync {
    fn get_all(&self) -> Result<Vec<Item>, NanoServiceError>;
    fn get_by_name(&self, name: &str) -> Result<Option<Item>, NanoServiceError>;
    fn create(&mut self, item: Item) -> Result<(), NanoServiceError>;
    fn delete(&mut self, name: &str) -> Result<(), NanoServiceError>;
    fn update(&mut self, item: Item) -> Result<(), NanoServiceError>;
}
```

### DAL Layer (Storage Implementations)

```rust
// dal/src/json_file.rs
use core::{Item, ItemStore};
use glue::{NanoServiceError, NanoServiceErrorStatus, safe_eject};
use std::fs::{File, OpenOptions};
use std::io::{Read, Write};

pub struct JsonFileStore {
    path: String,
}

impl JsonFileStore {
    pub fn new(path: &str) -> Self {
        Self { path: path.to_string() }
    }

    fn read_items(&self) -> Result<Vec<Item>, NanoServiceError> {
        let mut file = match File::open(&self.path) {
            Ok(f) => f,
            Err(e) if e.kind() == std::io::ErrorKind::NotFound => {
                return Ok(Vec::new());
            }
            Err(e) => return Err(e.into()),
        };

        let mut data = String::new();
        file.read_to_string(&mut data)?;

        if data.is_empty() {
            return Ok(Vec::new());
        }

        safe_eject!(
            serde_json::from_str(&data),
            NanoServiceErrorStatus::InternalError,
            "Failed to parse items file"
        )
    }

    fn write_items(&self, items: &[Item]) -> Result<(), NanoServiceError> {
        let data = serde_json::to_string_pretty(items)?;

        // Truncate and write (important: truncate first!)
        let mut file = OpenOptions::new()
            .write(true)
            .create(true)
            .truncate(true)  // Critical: truncate to avoid leftover data
            .open(&self.path)?;

        file.write_all(data.as_bytes())?;
        Ok(())
    }
}

impl ItemStore for JsonFileStore {
    fn get_all(&self) -> Result<Vec<Item>, NanoServiceError> {
        self.read_items()
    }

    fn get_by_name(&self, name: &str) -> Result<Option<Item>, NanoServiceError> {
        let items = self.read_items()?;
        Ok(items.into_iter().find(|i| i.name == name))
    }

    fn create(&mut self, item: Item) -> Result<(), NanoServiceError> {
        let mut items = self.read_items()?;
        if items.iter().any(|i| i.name == item.name) {
            return Err(NanoServiceError::new(
                format!("Item '{}' already exists", item.name),
                NanoServiceErrorStatus::Conflict,
            ));
        }
        items.push(item);
        self.write_items(&items)
    }

    fn delete(&mut self, name: &str) -> Result<(), NanoServiceError> {
        let mut items = self.read_items()?;
        let len_before = items.len();
        items.retain(|i| i.name != name);
        if items.len() == len_before {
            return Err(NanoServiceError::not_found("Item"));
        }
        self.write_items(&items)
    }

    fn update(&mut self, item: Item) -> Result<(), NanoServiceError> {
        let mut items = self.read_items()?;
        let pos = items.iter().position(|i| i.name == item.name)
            .ok_or_else(|| NanoServiceError::not_found("Item"))?;
        items[pos] = item;
        self.write_items(&items)
    }
}
```

### Networking Layer (Framework Adapters)

```rust
// networking/src/actix.rs
#[cfg(feature = "actix")]
use actix_web::{web, HttpResponse, http::StatusCode, error::ResponseError};
use core::ItemStore;
use glue::{NanoServiceError, NanoServiceErrorStatus, HeaderToken};
use std::sync::{Arc, Mutex};

// ResponseError implementation for framework integration
#[cfg(feature = "actix")]
impl ResponseError for NanoServiceError {
    fn status_code(&self) -> StatusCode {
        match self.status {
            NanoServiceErrorStatus::NotFound => StatusCode::NOT_FOUND,
            NanoServiceErrorStatus::BadRequest => StatusCode::BAD_REQUEST,
            NanoServiceErrorStatus::Unauthorized => StatusCode::UNAUTHORIZED,
            NanoServiceErrorStatus::Forbidden => StatusCode::FORBIDDEN,
            NanoServiceErrorStatus::Conflict => StatusCode::CONFLICT,
            NanoServiceErrorStatus::InternalError => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }

    fn error_response(&self) -> HttpResponse {
        HttpResponse::build(self.status_code())
            .json(&self.message)
    }
}

// FromRequest for HeaderToken extraction
#[cfg(feature = "actix")]
impl actix_web::FromRequest for HeaderToken {
    type Error = NanoServiceError;
    type Future = std::future::Ready<Result<Self, Self::Error>>;

    fn from_request(req: &actix_web::HttpRequest, _: &mut actix_web::dev::Payload) -> Self::Future {
        let result = req.headers()
            .get("Authorization")
            .and_then(|v| v.to_str().ok())
            .map(|s| HeaderToken::new(s.trim_start_matches("Bearer ").to_string()))
            .ok_or_else(|| NanoServiceError::unauthorized("Missing Authorization header"));

        std::future::ready(result)
    }
}

// Handler using shared store
#[cfg(feature = "actix")]
pub async fn get_all<S: ItemStore + 'static>(
    store: web::Data<Arc<Mutex<S>>>,
) -> Result<HttpResponse, NanoServiceError> {
    let store = store.lock().unwrap();
    let items = store.get_all()?;
    Ok(HttpResponse::Ok().json(items))
}

#[cfg(feature = "actix")]
pub async fn create<S: ItemStore + 'static>(
    store: web::Data<Arc<Mutex<S>>>,
    body: web::Json<core::Item>,
    _token: HeaderToken,  // Requires authentication
) -> Result<HttpResponse, NanoServiceError> {
    let mut store = store.lock().unwrap();
    store.create(body.into_inner())?;
    Ok(HttpResponse::Created().finish())
}
```

### Assembling the Server

```rust
// server/src/main.rs
use dal::JsonFileStore;
use networking::actix::{get_all, create};
use std::sync::{Arc, Mutex};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let store = Arc::new(Mutex::new(JsonFileStore::new("./data/items.json")));

    actix_web::HttpServer::new(move || {
        actix_web::App::new()
            .app_data(actix_web::web::Data::new(store.clone()))
            .route("/items", actix_web::web::get().to(get_all::<JsonFileStore>))
            .route("/items", actix_web::web::post().to(create::<JsonFileStore>))
    })
    .workers(4)
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

## Actor-Based Async Logging

### Why println! Blocks Multi-Threaded Servers

```rust
// What println! does internally:
use std::io::{stdout, Write};
let mut lock = stdout().lock();  // Blocks all threads
write!(lock, "hello world").unwrap();
```

Even with 4+ threads processing requests, all must wait for the stdout lock. Use the `tracing` crate which provides a global logger that efficiently handles concurrent log events.

### Basic Tracing Setup

```rust
// glue/src/logger/logger.rs
use tracing::Level;
use tracing_subscriber::FmtSubscriber;

pub fn init_logger() {
    let subscriber = FmtSubscriber::builder()
        .with_max_level(Level::INFO)
        .finish();
    tracing::subscriber::set_global_default(subscriber)
        .expect("Failed to set up logger");
}

// Wrapper functions to avoid spreading tracing dependency everywhere
pub fn log_info(message: &str) {
    tracing::info!("{}", message);
}

pub fn log_warn(message: &str) {
    tracing::warn!("{}", message);
}

pub fn log_error(message: &str) {
    tracing::error!("{}", message);
}
```

### Custom Actix Web Logging Middleware

Implement the `Transform` trait to log all HTTP requests automatically:

```rust
// glue/src/logger/network_wrappers/actix_web.rs
use actix_web::{dev::ServiceRequest, dev::ServiceResponse, Error};
use actix_web::dev::{Transform, Service};
use futures_util::future::{ok, Ready};
use std::task::{Context, Poll};
use std::pin::Pin;

pub struct ActixLogger;

impl<S, B> Transform<S, ServiceRequest> for ActixLogger
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = LoggingMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(LoggingMiddleware { service })
    }
}

pub struct LoggingMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for LoggingMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = Pin<Box<dyn futures_util::Future<Output = Result<Self::Response, Self::Error>>>>;

    fn poll_ready(&self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.service.poll_ready(cx)
    }

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let fut = self.service.call(req);
        Box::pin(async move {
            let res = fut.await?;
            let req_info = format!(
                "{} {} {}",
                res.request().method(),
                res.request().uri(),
                res.status().as_str()
            );
            tracing::info!("Request: {}", req_info);
            Ok(res)
        })
    }
}

// Usage in server
use glue::logger::{logger::init_logger, network_wrappers::actix_web::ActixLogger};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    init_logger();

    HttpServer::new(|| {
        App::new()
            .wrap(ActixLogger)  // Log all requests
            .wrap(Cors::default().allow_any_origin())
            .configure(api::views_factory)
    })
    .workers(4)
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### Actor-Based Remote Logging

For logging to external services (Elasticsearch, etc.) without blocking request processing, use a background actor that receives log messages via a channel:

```rust
// glue/src/logger/elastic_actor.rs
use tokio::sync::mpsc;
use tokio::sync::mpsc::{Receiver, Sender};
use serde_json::json;
use reqwest::{Client, Body};
use chrono::Utc;
use std::sync::LazyLock as Lazy;
use serde::Serialize;

#[derive(Debug, Serialize)]
struct LogMessage {
    level: String,
    message: String,
}

/// Send a log message to the background actor.
/// The actor is lazily initialized on first call.
pub async fn send_log(level: &str, message: &str) {
    static LOG_CHANNEL: Lazy<Sender<LogMessage>> = Lazy::new(|| {
        let (tx, rx) = mpsc::channel(100);  // Buffer 100 messages
        tokio::spawn(async move {
            elastic_actor(rx).await;
        });
        tx
    });

    // Non-blocking send - if channel is full, message is dropped
    let _ = LOG_CHANNEL.send(LogMessage {
        level: level.to_string(),
        message: message.to_string(),
    }).await;
}

/// Background actor that consumes log messages and sends to Elasticsearch
async fn elastic_actor(mut rx: Receiver<LogMessage>) {
    let elastic_url = std::env::var("ELASTICSEARCH_URL")
        .expect("ELASTICSEARCH_URL must be set");
    let client = Client::new();

    while let Some(log) = rx.recv().await {
        let body = json!({
            "level": log.level,
            "message": log.message,
            "timestamp": Utc::now().to_rfc3339()
        });

        let body = Body::from(serde_json::to_string(&body).unwrap());

        match client.post(&elastic_url)
            .header("Content-Type", "application/json")
            .header("Accept", "application/json")
            .body(body)
            .send()
            .await
        {
            Ok(_) => {},
            Err(e) => {
                // Log to terminal as fallback - don't kill the actor
                eprintln!("Failed to send log to Elasticsearch: {}", e);
            }
        }
    }
}
```

### Feature-Gated Logging Configuration

Use Cargo features to conditionally enable remote logging:

```toml
# glue/Cargo.toml
[dependencies]
tracing = "0.1.4"
tracing-subscriber = "0.3.18"
futures-util = "0.3.30"
serde_json = { version = "1.0.120", optional = true }
tokio = { version = "1.38.1", optional = true }
reqwest = { version = "0.12.5", optional = true }
chrono = { version = "0.4.38", optional = true }

[features]
actix = ["dep:actix-web"]
elastic-logger = ["dep:serde_json", "dep:tokio", "dep:reqwest", "dep:chrono"]
```

```rust
// glue/src/logger/logger.rs
#[cfg(feature = "elastic-logger")]
use super::elastic_actor::send_log;

pub async fn log_info(message: &str) {
    tracing::info!("{}", message);
    #[cfg(feature = "elastic-logger")]
    send_log("INFO", message).await;
}

pub async fn log_warn(message: &str) {
    tracing::warn!("{}", message);
    #[cfg(feature = "elastic-logger")]
    send_log("WARN", message).await;
}

pub async fn log_error(message: &str) {
    tracing::error!("{}", message);
    #[cfg(feature = "elastic-logger")]
    send_log("ERROR", message).await;
}
```

Enable in dependent crates:
```toml
# ingress/Cargo.toml
glue = { path = "../glue", features = ["actix", "elastic-logger"] }
```

## Related Skills

- **[architecture.md](architecture.md)** — Concepts, decision tables, and rules that these examples implement
- **[domain-patterns.md](domain-patterns.md)** — DDD patterns: entities, aggregates, event sourcing, CQRS
- **[SKILL.md](SKILL.md)** — Core Rust: ownership, traits, error handling, module organization
