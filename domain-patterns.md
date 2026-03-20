# Domain-Driven Design Patterns in Rust

Bounded contexts, domain entities, value objects, domain events, event sourcing, CQRS, and anti-corruption layers — all implemented with Rust's type system.

## Rules for Domain Patterns (LLM)

1. **ALWAYS model bounded contexts as separate Cargo crates** — workspace members with explicit dependencies enforce context boundaries at compile time; shared types go in a `shared` crate only when truly context-agnostic
2. **ALWAYS use newtype wrappers for domain identifiers** — `struct OrderId(Uuid)` prevents passing a `UserId` where `OrderId` is expected; derive `PartialEq, Eq, Hash, Clone, Copy` for ergonomics
3. **ALWAYS make domain events immutable and serializable** — events represent facts that happened; use `#[derive(Clone, Serialize, Deserialize)]` and never include `&mut` references
4. **ALWAYS version events from the start** — add a version field or use enum variants; upcasting old events is far easier than retroactively adding versioning
5. **ALWAYS separate command handling from event application** — `handle(cmd) -> Vec<Event>` produces events, `apply(event) -> Self` updates state; never mutate state in the command handler
6. **PREFER native async traits over `#[async_trait]`** for repository traits — native `async fn` in traits (Rust 1.75+) avoids heap allocation per call; use `async-trait` only when you need `dyn Repository`
7. **ALWAYS use the outbox pattern for reliable event publishing** — write events to a DB table in the same transaction as state changes, then publish asynchronously; this prevents lost events on publish failure
8. **NEVER put infrastructure types in domain crates** — domain crates must have zero dependency on sqlx, redis, reqwest, etc.; use trait boundaries and implement in infrastructure crates

### Common Mistakes (BAD/GOOD)

**Leaking infrastructure into domain:**
```rust
// BAD: domain entity depends on sqlx
use sqlx::FromRow;
#[derive(FromRow)]
struct Order { id: i64, total: i64 }

// GOOD: domain is pure — infrastructure maps separately
// domain/src/order.rs
struct Order { id: OrderId, total: Money }

// infrastructure/src/order_repo.rs
#[derive(sqlx::FromRow)]
struct OrderRow { id: i64, total: i64 }
impl From<OrderRow> for Order { /* ... */ }
```

**Mutating state in command handler:**
```rust
// BAD: command handler mutates state directly
fn handle(&mut self, cmd: PlaceOrder) {
    self.status = OrderStatus::Placed;  // mutation!
    self.items = cmd.items;             // no events!
}

// GOOD: handler returns events, apply mutates state
fn handle(&self, cmd: PlaceOrder) -> Result<Vec<OrderEvent>, OrderError> {
    if self.status != OrderStatus::Draft { return Err(OrderError::AlreadyPlaced); }
    Ok(vec![OrderEvent::Placed { items: cmd.items }])
}
fn apply(mut self, event: OrderEvent) -> Self {
    match event {
        OrderEvent::Placed { items } => { self.status = OrderStatus::Placed; self.items = items; }
        // ...
    }
    self
}
```

**Raw IDs instead of newtypes:**
```rust
// BAD: easy to mix up user_id and order_id — both are Uuid
fn cancel_order(user_id: Uuid, order_id: Uuid) { /* ... */ }
cancel_order(order_id, user_id); // compiles but wrong!

// GOOD: newtype wrappers prevent mixups at compile time
struct UserId(Uuid);
struct OrderId(Uuid);
fn cancel_order(user_id: UserId, order_id: OrderId) { /* ... */ }
// cancel_order(order_id, user_id); // won't compile!
```

### Section Index

| Section | Content |
|---------|---------|
| [Bounded Contexts as Cargo Workspaces](#bounded-contexts-as-cargo-workspaces) | Workspace organization, context-specific types, shared kernel |
| [Anti-Corruption Layer (ACL)](#anti-corruption-layer-acl) | Trait-based translation between contexts |
| [Inter-Context Communication](#inter-context-communication) | Event channels, shared nothing, domain event bus |
| [Domain Events](#domain-events) | Event definitions, publishing, subscribing, event store |
| [Event Sourcing](#event-sourcing) | Aggregate replay, snapshots, versioning, upcasters |
| [CQRS](#cqrs-command-query-responsibility-segregation) | Command handlers, query models, projections, read/write separation |
| [Testing](#testing) | Aggregate tests, mock repositories, event assertion patterns |
| [Best Practices](#best-practices) | Immutable events, naming, draining, outbox, snapshots |

## Bounded Contexts as Cargo Workspaces

Each bounded context gets its own crate. Types are context-specific even when they share a name.

### Workspace Organization

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "2"
members = [
    "contexts/ecommerce",
    "contexts/inventory",
    "contexts/shipping",
    "contexts/shared",      # Context-agnostic types only
    "infrastructure",       # Database, messaging implementations
]

[workspace.dependencies]
uuid = "1.0"
chrono = "0.4"
serde = { version = "1.0", features = ["derive"] }
async-trait = "0.1"
thiserror = "2.0"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["postgres", "runtime-tokio"] }
```

### Directory Structure

```
project/
├── Cargo.toml                    # Workspace manifest
├── contexts/
│   ├── ecommerce/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── domain/
│   │       │   ├── mod.rs
│   │       │   ├── entity.rs     # Order, Customer entities
│   │       │   ├── value_object.rs
│   │       │   └── repository.rs # Trait definitions
│   │       ├── use_case/
│   │       │   ├── mod.rs
│   │       │   └── place_order.rs
│   │       └── adapters/
│   │           └── payment_gateway.rs
│   │
│   ├── inventory/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── domain/
│   │       │   ├── mod.rs
│   │       │   ├── entity.rs     # Product, Stock entities
│   │       │   └── repository.rs
│   │       └── use_case/
│   │           └── reserve_stock.rs
│   │
│   └── shared/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs            # ProductId, Money (if truly shared)
│
└── infrastructure/
    ├── Cargo.toml
    └── src/
        ├── lib.rs
        ├── postgres/
        └── messaging/
```

### Type Isolation Across Contexts

```rust
// contexts/ecommerce/src/domain/entity.rs
use uuid::Uuid;

/// ProductId in e-commerce context — represents a purchasable item
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ProductId(pub Uuid);

/// Order entity — e-commerce's view of an order
#[derive(Debug, Clone)]
pub struct Order {
    pub id: OrderId,
    pub customer_id: CustomerId,
    pub items: Vec<OrderItem>,
    pub status: OrderStatus,
}

#[derive(Debug, Clone)]
pub struct OrderItem {
    pub product_id: ProductId,  // E-commerce's ProductId
    pub quantity: u32,
    pub unit_price: Money,
}
```

```rust
// contexts/inventory/src/domain/entity.rs
use uuid::Uuid;

/// ProductId in inventory context — represents stockable goods
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ProductId(pub Uuid);  // DISTINCT from e-commerce's ProductId

/// Product entity — inventory's view of a product
#[derive(Debug, Clone)]
pub struct Product {
    pub id: ProductId,  // Inventory's ProductId
    pub sku: String,
    pub stock_level: u32,
    pub location: WarehouseLocation,
}

#[derive(Debug, Clone)]
pub struct StockReservation {
    pub product_id: ProductId,
    pub quantity: u32,
    pub reservation_id: Uuid,
    pub expires_at: chrono::DateTime<chrono::Utc>,
}
```

### Context-Specific Repositories

Each context defines its own repository traits — no shared database:

```rust
// contexts/ecommerce/src/domain/repository.rs
use crate::domain::entity::{Order, OrderId, CustomerId};

#[async_trait]
pub trait OrderRepository: Send + Sync {
    async fn find_by_id(&self, id: &OrderId) -> Result<Option<Order>, RepositoryError>;
    async fn save(&self, order: &Order) -> Result<(), RepositoryError>;
    async fn find_by_customer(&self, customer_id: &CustomerId) -> Result<Vec<Order>, RepositoryError>;
}

// contexts/inventory/src/domain/repository.rs
use crate::domain::entity::{Product, ProductId, StockReservation};

#[async_trait]
pub trait ProductRepository: Send + Sync {
    async fn find_by_id(&self, id: &ProductId) -> Result<Option<Product>, RepositoryError>;
    async fn update_stock(&self, id: &ProductId, new_level: u32) -> Result<(), RepositoryError>;
}

#[async_trait]
pub trait StockReservationRepository: Send + Sync {
    async fn reserve(&self, reservation: &StockReservation) -> Result<(), RepositoryError>;
    async fn release(&self, reservation_id: Uuid) -> Result<(), RepositoryError>;
}
```

**Native async traits (Rust 1.75+):** When you don't need `dyn Repository` (trait objects), drop `#[async_trait]` for zero-overhead native async:

```rust
// No #[async_trait] needed — static dispatch via generics
pub trait OrderRepository: Send + Sync {
    fn find_by_id(&self, id: &OrderId) -> impl Future<Output = Result<Option<Order>, RepositoryError>> + Send;
    fn save(&self, order: &Order) -> impl Future<Output = Result<(), RepositoryError>> + Send;
}

// Use #[async_trait] only when you need: Box<dyn OrderRepository>
```

## Anti-Corruption Layer (ACL)

### Protecting Context Boundaries

The consuming context defines the port (trait). The infrastructure layer provides the adapter.

```rust
// contexts/ecommerce/src/adapters/inventory_service.rs

/// Port (trait) defined BY e-commerce context
/// Inventory context must adapt TO this interface
#[async_trait]
pub trait InventoryService: Send + Sync {
    async fn check_availability(
        &self,
        product_id: crate::domain::entity::ProductId,
        quantity: u32,
    ) -> Result<bool, InventoryError>;

    async fn reserve_stock(
        &self,
        product_id: crate::domain::entity::ProductId,
        quantity: u32,
    ) -> Result<ReservationId, InventoryError>;

    async fn release_reservation(
        &self,
        reservation_id: ReservationId,
    ) -> Result<(), InventoryError>;
}

#[derive(Debug, Clone, Copy)]
pub struct ReservationId(pub Uuid);

#[derive(Debug, thiserror::Error)]
pub enum InventoryError {
    #[error("Product not found")]
    ProductNotFound,
    #[error("Insufficient stock")]
    InsufficientStock,
    #[error("Service unavailable")]
    ServiceUnavailable,
}
```

### Adapter Implementation

```rust
// infrastructure/src/adapters/inventory_adapter.rs

use ecommerce::adapters::{InventoryService, InventoryError, ReservationId};
use inventory::domain::repository::ProductRepository;

/// Adapter that translates between e-commerce and inventory contexts
pub struct InventoryServiceAdapter {
    product_repo: Arc<dyn ProductRepository>,
    reservation_repo: Arc<dyn StockReservationRepository>,
}

#[async_trait]
impl InventoryService for InventoryServiceAdapter {
    async fn check_availability(
        &self,
        product_id: ecommerce::domain::entity::ProductId,
        quantity: u32,
    ) -> Result<bool, InventoryError> {
        // TRANSLATION: Convert e-commerce ProductId to inventory ProductId
        let inventory_product_id = inventory::domain::entity::ProductId(product_id.0);

        let product = self.product_repo
            .find_by_id(&inventory_product_id)
            .await
            .map_err(|_| InventoryError::ServiceUnavailable)?
            .ok_or(InventoryError::ProductNotFound)?;

        Ok(product.stock_level >= quantity)
    }

    async fn reserve_stock(
        &self,
        product_id: ecommerce::domain::entity::ProductId,
        quantity: u32,
    ) -> Result<ReservationId, InventoryError> {
        let inventory_product_id = inventory::domain::entity::ProductId(product_id.0);

        // Check availability first
        let product = self.product_repo
            .find_by_id(&inventory_product_id)
            .await
            .map_err(|_| InventoryError::ServiceUnavailable)?
            .ok_or(InventoryError::ProductNotFound)?;

        if product.stock_level < quantity {
            return Err(InventoryError::InsufficientStock);
        }

        // Create reservation in inventory context
        let reservation = inventory::domain::entity::StockReservation {
            product_id: inventory_product_id,
            quantity,
            reservation_id: Uuid::new_v4(),
            expires_at: chrono::Utc::now() + chrono::Duration::minutes(15),
        };

        self.reservation_repo.reserve(&reservation).await
            .map_err(|_| InventoryError::ServiceUnavailable)?;

        // Return e-commerce's view of the reservation
        Ok(ReservationId(reservation.reservation_id))
    }

    async fn release_reservation(
        &self,
        reservation_id: ReservationId,
    ) -> Result<(), InventoryError> {
        self.reservation_repo.release(reservation_id.0).await
            .map_err(|_| InventoryError::ServiceUnavailable)
    }
}
```

## Inter-Context Communication

### Strategy Comparison

| Strategy | Coupling | Latency | Consistency | Use When |
|----------|----------|---------|-------------|----------|
| gRPC | Medium | Low | Strong | Real-time queries, internal services |
| REST | Medium | Low-Med | Strong | External APIs, simple operations |
| Message Queue | Low | High | Eventual | Events, async workflows |
| Shared Database | HIGH | Low | Strong | **Avoid — anti-pattern** |

### gRPC Communication

```protobuf
// protos/inventory_service.proto
syntax = "proto3";
package inventory;

service InventoryService {
    rpc CheckAvailability(CheckAvailabilityRequest) returns (CheckAvailabilityResponse);
    rpc ReserveStock(ReserveStockRequest) returns (ReserveStockResponse);
}

message CheckAvailabilityRequest {
    string product_id = 1;
    uint32 quantity = 2;
}

message CheckAvailabilityResponse {
    bool available = 1;
    uint32 current_stock = 2;
}

message ReserveStockRequest {
    string product_id = 1;
    uint32 quantity = 2;
}

message ReserveStockResponse {
    string reservation_id = 1;
    string expires_at = 2;
}
```

```rust
// gRPC client adapter
use tonic::transport::Channel;
use inventory_proto::inventory_service_client::InventoryServiceClient;

pub struct GrpcInventoryAdapter {
    client: InventoryServiceClient<Channel>,
}

impl GrpcInventoryAdapter {
    pub async fn new(endpoint: &str) -> Result<Self, tonic::transport::Error> {
        let client = InventoryServiceClient::connect(endpoint.to_string()).await?;
        Ok(Self { client })
    }
}

#[async_trait]
impl InventoryService for GrpcInventoryAdapter {
    async fn check_availability(
        &self,
        product_id: ecommerce::domain::entity::ProductId,
        quantity: u32,
    ) -> Result<bool, InventoryError> {
        let request = tonic::Request::new(CheckAvailabilityRequest {
            product_id: product_id.0.to_string(),
            quantity,
        });

        let response = self.client.clone()
            .check_availability(request)
            .await
            .map_err(|_| InventoryError::ServiceUnavailable)?;

        Ok(response.into_inner().available)
    }

    // ... other methods follow same pattern
}
```

### Message Bus with Kafka

```rust
// infrastructure/src/messaging/mod.rs

use rdkafka::producer::FutureProducer;
use rdkafka::consumer::StreamConsumer;
use rdkafka::config::ClientConfig;

pub struct MessageBus {
    producer: FutureProducer,
}

impl MessageBus {
    /// Publish event to a topic named after the bounded context
    pub async fn publish<E: Serialize>(
        &self,
        context: &str,
        event_type: &str,
        event: &E,
    ) -> Result<(), MessageError> {
        let topic = format!("{}.{}", context, event_type);
        let payload = serde_json::to_vec(event)?;

        let record = FutureRecord::to(&topic)
            .payload(&payload)
            .key(event_type);

        self.producer.send(record, Duration::from_secs(5)).await?;
        Ok(())
    }
}

/// Start consumer for a specific context
pub async fn start_context_consumer(
    context: &str,
    handler: impl Fn(String, Vec<u8>) + Send + 'static,
) {
    let consumer: StreamConsumer = ClientConfig::new()
        .set("bootstrap.servers", "localhost:9092")
        .set("group.id", format!("{}-consumer", context))
        .create()
        .unwrap();

    let topic_pattern = format!("{}.*", context);
    consumer.subscribe(&[&topic_pattern]).unwrap();

    loop {
        if let Ok(message) = consumer.recv().await {
            if let Some(payload) = message.payload() {
                let topic = message.topic().to_string();
                handler(topic, payload.to_vec());
            }
        }
    }
}
```

### Shared Database Anti-Pattern

```rust
// BAD: Two contexts sharing the same table/schema

// E-commerce context directly queries inventory table
async fn check_stock_bad(pool: &PgPool, product_id: Uuid) -> Result<u32, sqlx::Error> {
    // PROBLEM: E-commerce now depends on inventory's schema
    // Changes to inventory table break e-commerce
    sqlx::query_scalar!(
        "SELECT stock_level FROM products WHERE id = $1",  // Inventory's table!
        product_id
    )
    .fetch_one(pool)
    .await
}

// PROBLEMS:
// 1. Schema coupling — changes break other contexts
// 2. Unclear ownership — who owns the 'products' table?
// 3. Testing difficulty — can't test contexts in isolation
// 4. Deployment coupling — must coordinate releases
```

```rust
// ACCEPTABLE: Read-only replicated view

// Inventory publishes to a read-replica specifically for e-commerce
// E-commerce owns this view and can query it freely
async fn check_stock_from_replica(pool: &PgPool, product_id: Uuid) -> Result<u32, sqlx::Error> {
    sqlx::query_scalar!(
        "SELECT stock_level FROM ecommerce_stock_view WHERE product_id = $1",
        product_id
    )
    .fetch_one(pool)
    .await
}

// The view is populated by:
// 1. CDC (Change Data Capture) from inventory's source table
// 2. Event subscription from inventory context
// 3. Periodic sync job
//
// Key difference: E-commerce OWNS this table, inventory POPULATES it
```

### Use Case Crossing Context Boundaries

```rust
// contexts/ecommerce/src/use_case/place_order.rs

pub struct PlaceOrderUseCase {
    order_repo: Arc<dyn OrderRepository>,
    inventory_service: Arc<dyn InventoryService>,
    payment_gateway: Arc<dyn PaymentGateway>,
    event_publisher: Arc<dyn EventPublisher>,
}

impl PlaceOrderUseCase {
    pub async fn execute(&self, input: PlaceOrderInput) -> Result<OrderId, PlaceOrderError> {
        // 1. Validate items exist and are available (crosses to inventory)
        for item in &input.items {
            let available = self.inventory_service
                .check_availability(item.product_id, item.quantity)
                .await
                .map_err(|e| PlaceOrderError::InventoryUnavailable(e.to_string()))?;

            if !available {
                return Err(PlaceOrderError::InsufficientStock(item.product_id));
            }
        }

        // 2. Reserve stock (crosses to inventory)
        let mut reservations = Vec::new();
        for item in &input.items {
            let reservation_id = self.inventory_service
                .reserve_stock(item.product_id, item.quantity)
                .await
                .map_err(|e| PlaceOrderError::ReservationFailed(e.to_string()))?;
            reservations.push(reservation_id);
        }

        // 3. Process payment (crosses to payment context)
        let payment_result = self.payment_gateway
            .charge(input.payment_info, input.total_amount)
            .await;

        if payment_result.is_err() {
            // Rollback: release all reservations
            for reservation_id in reservations {
                let _ = self.inventory_service.release_reservation(reservation_id).await;
            }
            return Err(PlaceOrderError::PaymentFailed);
        }

        // 4. Create order (this context's responsibility)
        let order = Order::new(
            OrderId::new(),
            input.customer_id,
            input.items,
        );
        self.order_repo.save(&order).await?;

        // 5. Publish event for other contexts to react
        self.event_publisher.publish(OrderPlaced::from(&order)).await?;

        Ok(order.id)
    }
}
```

## Domain Events

### Event Definition

Events are immutable facts in past tense:

```rust
use chrono::{DateTime, Utc};
use uuid::Uuid;

/// Trait for all domain events — immutable facts that have occurred.
/// Send + Sync + 'static enables safe sharing across threads.
pub trait DomainEvent: Send + Sync + 'static {
    /// Unique identifier for this specific event instance
    fn event_id(&self) -> Uuid;
    /// When the event occurred
    fn timestamp(&self) -> DateTime<Utc>;
    /// Event type name for logging/routing
    fn event_type(&self) -> &'static str;
}
```

### Implementing Domain Events

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct UserEmailVerified {
    event_id: Uuid,
    timestamp: DateTime<Utc>,
    user_id: Uuid,
    verified_email: String,
}

impl UserEmailVerified {
    pub fn new(user_id: Uuid, verified_email: String) -> Self {
        Self {
            event_id: Uuid::new_v4(),
            timestamp: Utc::now(),
            user_id,
            verified_email,
        }
    }

    pub fn user_id(&self) -> Uuid { self.user_id }
    pub fn verified_email(&self) -> &str { &self.verified_email }
}

impl DomainEvent for UserEmailVerified {
    fn event_id(&self) -> Uuid { self.event_id }
    fn timestamp(&self) -> DateTime<Utc> { self.timestamp }
    fn event_type(&self) -> &'static str { "UserEmailVerified" }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced {
    event_id: Uuid,
    timestamp: DateTime<Utc>,
    pub order_id: Uuid,
    pub customer_id: Uuid,
    pub total_amount: u64,
}

impl OrderPlaced {
    pub fn new(order_id: Uuid, customer_id: Uuid, total_amount: u64) -> Self {
        Self {
            event_id: Uuid::new_v4(),
            timestamp: Utc::now(),
            order_id,
            customer_id,
            total_amount,
        }
    }
}

impl DomainEvent for OrderPlaced {
    fn event_id(&self) -> Uuid { self.event_id }
    fn timestamp(&self) -> DateTime<Utc> { self.timestamp }
    fn event_type(&self) -> &'static str { "OrderPlaced" }
}
```

### Uncommitted Events Pattern

Aggregates collect events internally, drained after persistence:

```rust
use std::collections::VecDeque;

/// Aggregate that records domain events internally
#[derive(Debug)]
pub struct User {
    id: Uuid,
    email: String,
    is_email_verified: bool,
    uncommitted_events: VecDeque<Box<dyn DomainEvent>>,
}

impl User {
    pub fn new(id: Uuid, email: String) -> Self {
        Self {
            id,
            email,
            is_email_verified: false,
            uncommitted_events: VecDeque::new(),
        }
    }

    /// Verifies email and records the event
    pub fn verify_email(&mut self, email_to_verify: &str) -> Result<(), &'static str> {
        if self.is_email_verified {
            return Err("Email already verified");
        }
        if self.email != email_to_verify {
            return Err("Email mismatch");
        }

        self.is_email_verified = true;
        let event = UserEmailVerified::new(self.id, self.email.clone());
        self.uncommitted_events.push_back(Box::new(event));
        Ok(())
    }

    /// Drains and returns all uncommitted events.
    /// Called by repository after successful save.
    pub fn take_uncommitted_events(&mut self) -> VecDeque<Box<dyn DomainEvent>> {
        std::mem::take(&mut self.uncommitted_events)
    }

    pub fn has_uncommitted_events(&self) -> bool {
        !self.uncommitted_events.is_empty()
    }

    pub fn id(&self) -> Uuid { self.id }
    pub fn email(&self) -> &str { &self.email }
    pub fn is_email_verified(&self) -> bool { self.is_email_verified }
}
```

### Repository Publishing Events After Save

```rust
pub struct PostgresUserRepository {
    pool: PgPool,
    event_publisher: Arc<dyn EventPublisher>,
}

#[async_trait]
impl UserRepository for PostgresUserRepository {
    async fn save(&self, user: &mut User) -> Result<(), RepositoryError> {
        // 1. Persist to database
        sqlx::query!(
            "INSERT INTO users (id, email, is_email_verified) VALUES ($1, $2, $3)
             ON CONFLICT (id) DO UPDATE SET email = $2, is_email_verified = $3",
            user.id(),
            user.email(),
            user.is_email_verified()
        )
        .execute(&self.pool)
        .await
        .map_err(|e| RepositoryError::Database(Box::new(e)))?;

        // 2. Take and publish events after successful save
        let events = user.take_uncommitted_events();
        for event in events {
            if let Err(e) = self.event_publisher.publish(event).await {
                // Log but don't fail — event can be replayed from outbox
                tracing::error!("Failed to publish event: {}", e);
            }
        }

        Ok(())
    }
}
```

### Event Publisher Trait

```rust
#[async_trait]
pub trait EventPublisher: Send + Sync {
    async fn publish(&self, event: Box<dyn DomainEvent>) -> Result<(), PublishError>;
}

#[derive(Debug, thiserror::Error)]
pub enum PublishError {
    #[error("No subscribers")]
    NoSubscribers,
    #[error("Serialization failed: {0}")]
    Serialization(String),
    #[error("Transport error: {0}")]
    Transport(String),
}

/// In-memory publisher using broadcast channel
pub struct InMemoryEventPublisher {
    sender: tokio::sync::broadcast::Sender<DomainEventEnvelope>,
}

#[async_trait]
impl EventPublisher for InMemoryEventPublisher {
    async fn publish(&self, event: Box<dyn DomainEvent>) -> Result<(), PublishError> {
        // Convert to envelope based on event type
        let envelope = match event.event_type() {
            "UserEmailVerified" => {
                // Downcast or use type registry
                todo!("Convert to DomainEventEnvelope::UserVerified")
            }
            _ => return Err(PublishError::Serialization("Unknown event type".into())),
        };

        self.sender.send(envelope)
            .map_err(|_| PublishError::NoSubscribers)?;
        Ok(())
    }
}

/// Kafka publisher implementation
pub struct KafkaEventPublisher {
    producer: FutureProducer,
    topic: String,
}

#[async_trait]
impl EventPublisher for KafkaEventPublisher {
    async fn publish(&self, event: Box<dyn DomainEvent>) -> Result<(), PublishError> {
        let payload = serde_json::to_vec(&EventPayload {
            event_id: event.event_id(),
            event_type: event.event_type().to_string(),
            timestamp: event.timestamp(),
        }).map_err(|e| PublishError::Serialization(e.to_string()))?;

        let record = FutureRecord::to(&self.topic)
            .key(&event.event_id().to_string())
            .payload(&payload);

        self.producer.send(record, Duration::from_secs(5))
            .await
            .map_err(|(e, _)| PublishError::Transport(e.to_string()))?;

        Ok(())
    }
}
```

### Event Bus with Tokio Broadcast

```rust
/// Event wrapper for type-erased domain events
#[derive(Clone, Debug)]
pub enum DomainEventEnvelope {
    UserEmailVerified(UserEmailVerified),
    OrderPlaced(OrderPlaced),
}

#[derive(Clone)]
pub struct EventBus {
    sender: tokio::sync::broadcast::Sender<DomainEventEnvelope>,
}

impl EventBus {
    pub fn new(capacity: usize) -> Self {
        let (sender, _) = tokio::sync::broadcast::channel(capacity);
        Self { sender }
    }

    pub async fn publish(&self, event: DomainEventEnvelope) -> Result<(), &'static str> {
        self.sender.send(event).map(|_| ()).map_err(|_| "No active receivers")
    }

    pub fn subscribe(&self) -> tokio::sync::broadcast::Receiver<DomainEventEnvelope> {
        self.sender.subscribe()
    }

    pub fn receiver_count(&self) -> usize {
        self.sender.receiver_count()
    }
}
```

### Async Event Handlers

```rust
use tokio::sync::broadcast::error::RecvError;

pub async fn start_event_handler(event_bus: Arc<EventBus>) {
    let mut receiver = event_bus.subscribe();

    tokio::spawn(async move {
        loop {
            match receiver.recv().await {
                Ok(event) => {
                    handle_event(event).await;
                }
                Err(RecvError::Lagged(count)) => {
                    tracing::warn!("Event handler lagged, missed {} events", count);
                    // Could trigger recovery/replay logic here
                }
                Err(RecvError::Closed) => {
                    tracing::info!("Event bus closed, shutting down handler");
                    break;
                }
            }
        }
    });
}

async fn handle_event(event: DomainEventEnvelope) {
    match event {
        DomainEventEnvelope::UserEmailVerified(e) => {
            send_welcome_email(&e).await;
        }
        DomainEventEnvelope::OrderPlaced(e) => {
            notify_warehouse(&e).await;
            update_inventory(&e).await;
        }
    }
}
```

### Multiple Specialized Handlers

```rust
pub async fn start_all_handlers(event_bus: Arc<EventBus>) {
    // Email notification handler
    let bus = Arc::clone(&event_bus);
    tokio::spawn(async move {
        let mut rx = bus.subscribe();
        while let Ok(event) = rx.recv().await {
            if let DomainEventEnvelope::UserEmailVerified(e) = event {
                send_welcome_email(&e).await;
            }
        }
    });

    // Analytics handler
    let bus = Arc::clone(&event_bus);
    tokio::spawn(async move {
        let mut rx = bus.subscribe();
        while let Ok(event) = rx.recv().await {
            record_analytics(&event).await;
        }
    });

    // Audit log handler
    let bus = Arc::clone(&event_bus);
    tokio::spawn(async move {
        let mut rx = bus.subscribe();
        while let Ok(event) = rx.recv().await {
            write_audit_log(&event).await;
        }
    });
}
```

### Transactional Outbox

For reliable event publishing with database transactions:

```rust
#[derive(Debug, Clone)]
pub struct OutboxEntry {
    pub id: Uuid,
    pub event_type: String,
    pub payload: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub published_at: Option<DateTime<Utc>>,
}

pub struct OutboxRepository {
    pool: PgPool,
}

impl OutboxRepository {
    /// Save event to outbox in same transaction as aggregate
    pub async fn save_event(
        &self,
        tx: &mut sqlx::Transaction<'_, sqlx::Postgres>,
        event: &dyn DomainEvent,
        payload: serde_json::Value,
    ) -> Result<(), sqlx::Error> {
        sqlx::query!(
            r#"
            INSERT INTO outbox (id, event_type, payload, created_at)
            VALUES ($1, $2, $3, $4)
            "#,
            event.event_id(),
            event.event_type(),
            payload,
            event.timestamp()
        )
        .execute(&mut **tx)
        .await?;
        Ok(())
    }

    /// Get unpublished events for background processor
    pub async fn get_unpublished(&self, limit: i64) -> Result<Vec<OutboxEntry>, sqlx::Error> {
        sqlx::query_as!(
            OutboxEntry,
            r#"
            SELECT id, event_type, payload, created_at, published_at
            FROM outbox
            WHERE published_at IS NULL
            ORDER BY created_at ASC
            LIMIT $1
            "#,
            limit
        )
        .fetch_all(&self.pool)
        .await
    }

    /// Mark event as published
    pub async fn mark_published(&self, id: Uuid) -> Result<(), sqlx::Error> {
        sqlx::query!(
            "UPDATE outbox SET published_at = $1 WHERE id = $2",
            Utc::now(),
            id
        )
        .execute(&self.pool)
        .await?;
        Ok(())
    }
}

/// Background worker to publish outbox events
pub async fn outbox_processor(
    outbox_repo: Arc<OutboxRepository>,
    publisher: Arc<dyn EventPublisher>,
) {
    loop {
        match outbox_repo.get_unpublished(100).await {
            Ok(entries) => {
                for entry in entries {
                    if let Err(e) = publisher.publish_raw(&entry).await {
                        tracing::error!("Failed to publish outbox entry {}: {}", entry.id, e);
                    } else {
                        let _ = outbox_repo.mark_published(entry.id).await;
                    }
                }
            }
            Err(e) => tracing::error!("Outbox query failed: {}", e),
        }

        tokio::time::sleep(Duration::from_secs(1)).await;
    }
}
```

## Event Sourcing

Store all state changes as events. Reconstruct state by replaying.

**Crate ecosystem:** For production event sourcing, consider [`cqrs-es`](https://crates.io/crates/cqrs-es) which provides `Aggregate`, `DomainEvent`, `Query`, and `CqrsFramework` traits with persistence backends (postgres-es, dynamo-es, mysql-es). The patterns below work standalone or alongside cqrs-es.

### Event Enum Pattern

```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use serde::{Serialize, Deserialize};
use uuid::Uuid;

/// All possible events for a BankAccount aggregate
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum BankAccountEvent {
    AccountCreated {
        account_id: Uuid,
        holder_name: String,
        initial_balance: Decimal,
        currency: String,
        created_at: DateTime<Utc>,
    },
    DepositMade {
        account_id: Uuid,
        amount: Decimal,
        description: String,
        occurred_at: DateTime<Utc>,
    },
    WithdrawalMade {
        account_id: Uuid,
        amount: Decimal,
        description: String,
        occurred_at: DateTime<Utc>,
    },
    AccountClosed {
        account_id: Uuid,
        reason: String,
        closed_at: DateTime<Utc>,
    },
}

impl BankAccountEvent {
    pub fn account_id(&self) -> Uuid {
        match self {
            Self::AccountCreated { account_id, .. } => *account_id,
            Self::DepositMade { account_id, .. } => *account_id,
            Self::WithdrawalMade { account_id, .. } => *account_id,
            Self::AccountClosed { account_id, .. } => *account_id,
        }
    }

    pub fn occurred_at(&self) -> DateTime<Utc> {
        match self {
            Self::AccountCreated { created_at, .. } => *created_at,
            Self::DepositMade { occurred_at, .. } => *occurred_at,
            Self::WithdrawalMade { occurred_at, .. } => *occurred_at,
            Self::AccountClosed { closed_at, .. } => *closed_at,
        }
    }
}
```

### Aggregate with State Replay

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum AccountStatus { Active, Closed }

#[derive(Debug, Clone)]
pub struct BankAccount {
    id: Uuid,
    holder_name: String,
    balance: Decimal,
    currency: String,
    status: AccountStatus,
    version: u64,
}

impl Default for BankAccount {
    fn default() -> Self {
        Self {
            id: Uuid::nil(),
            holder_name: String::new(),
            balance: Decimal::ZERO,
            currency: String::new(),
            status: AccountStatus::Active,
            version: 0,
        }
    }
}

impl BankAccount {
    /// Apply a single event to update state.
    /// This is the core of event sourcing — state derived from events.
    pub fn apply_event(&mut self, event: &BankAccountEvent) {
        match event {
            BankAccountEvent::AccountCreated {
                account_id, holder_name, initial_balance, currency, ..
            } => {
                self.id = *account_id;
                self.holder_name = holder_name.clone();
                self.balance = *initial_balance;
                self.currency = currency.clone();
                self.status = AccountStatus::Active;
            }
            BankAccountEvent::DepositMade { amount, .. } => {
                self.balance += amount;
            }
            BankAccountEvent::WithdrawalMade { amount, .. } => {
                self.balance -= amount;
            }
            BankAccountEvent::AccountClosed { .. } => {
                self.status = AccountStatus::Closed;
            }
        }
        self.version += 1;
    }

    /// Reconstruct aggregate state from event history
    pub fn from_events(events: &[BankAccountEvent]) -> Self {
        let mut account = BankAccount::default();
        for event in events {
            account.apply_event(event);
        }
        account
    }

    /// Reconstruct from snapshot + subsequent events
    pub fn from_snapshot_and_events(
        snapshot: BankAccountSnapshot,
        events: &[BankAccountEvent],
    ) -> Self {
        let mut account = snapshot.into_account();
        for event in events {
            account.apply_event(event);
        }
        account
    }

    pub fn id(&self) -> Uuid { self.id }
    pub fn balance(&self) -> Decimal { self.balance }
    pub fn status(&self) -> &AccountStatus { &self.status }
    pub fn version(&self) -> u64 { self.version }
}
```

### Command Handling (Producing Events)

Commands validate business rules and return events:

```rust
#[derive(Debug, thiserror::Error)]
pub enum AccountError {
    #[error("Insufficient funds: available {available}, requested {requested}")]
    InsufficientFunds { available: Decimal, requested: Decimal },
    #[error("Account is closed")]
    AccountClosed,
    #[error("Invalid amount: {0}")]
    InvalidAmount(String),
    #[error("Account already exists")]
    AccountAlreadyExists,
}

impl BankAccount {
    /// Create a new account — returns the creation event
    pub fn create(
        account_id: Uuid,
        holder_name: String,
        initial_balance: Decimal,
        currency: String,
    ) -> Result<BankAccountEvent, AccountError> {
        if initial_balance < Decimal::ZERO {
            return Err(AccountError::InvalidAmount("Initial balance cannot be negative".into()));
        }

        Ok(BankAccountEvent::AccountCreated {
            account_id,
            holder_name,
            initial_balance,
            currency,
            created_at: Utc::now(),
        })
    }

    /// Deposit funds — validates and returns event
    pub fn deposit(
        &self,
        amount: Decimal,
        description: String,
    ) -> Result<BankAccountEvent, AccountError> {
        if self.status == AccountStatus::Closed {
            return Err(AccountError::AccountClosed);
        }
        if amount <= Decimal::ZERO {
            return Err(AccountError::InvalidAmount("Deposit must be positive".into()));
        }

        Ok(BankAccountEvent::DepositMade {
            account_id: self.id,
            amount,
            description,
            occurred_at: Utc::now(),
        })
    }

    /// Withdraw funds — validates and returns event
    pub fn withdraw(
        &self,
        amount: Decimal,
        description: String,
    ) -> Result<BankAccountEvent, AccountError> {
        if self.status == AccountStatus::Closed {
            return Err(AccountError::AccountClosed);
        }
        if amount <= Decimal::ZERO {
            return Err(AccountError::InvalidAmount("Withdrawal must be positive".into()));
        }
        if self.balance < amount {
            return Err(AccountError::InsufficientFunds {
                available: self.balance,
                requested: amount,
            });
        }

        Ok(BankAccountEvent::WithdrawalMade {
            account_id: self.id,
            amount,
            description,
            occurred_at: Utc::now(),
        })
    }

    /// Close account
    pub fn close(&self, reason: String) -> Result<BankAccountEvent, AccountError> {
        if self.status == AccountStatus::Closed {
            return Err(AccountError::AccountClosed);
        }

        Ok(BankAccountEvent::AccountClosed {
            account_id: self.id,
            reason,
            closed_at: Utc::now(),
        })
    }
}
```

### Event Store — In-Memory

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

pub struct InMemoryEventStore {
    events: Arc<RwLock<HashMap<Uuid, Vec<BankAccountEvent>>>>,
}

impl InMemoryEventStore {
    pub fn new() -> Self {
        Self { events: Arc::new(RwLock::new(HashMap::new())) }
    }

    pub fn append_events(
        &self,
        aggregate_id: Uuid,
        events: Vec<BankAccountEvent>,
        expected_version: Option<u64>,
    ) -> Result<(), EventStoreError> {
        let mut store = self.events.write().unwrap();
        let current_events = store.entry(aggregate_id).or_insert_with(Vec::new);

        // Optimistic concurrency check
        if let Some(expected) = expected_version {
            if current_events.len() as u64 != expected {
                return Err(EventStoreError::ConcurrencyConflict {
                    expected,
                    actual: current_events.len() as u64,
                });
            }
        }

        current_events.extend(events);
        Ok(())
    }

    pub fn get_events(&self, aggregate_id: Uuid) -> Vec<BankAccountEvent> {
        let store = self.events.read().unwrap();
        store.get(&aggregate_id).cloned().unwrap_or_default()
    }

    pub fn get_events_from_version(
        &self,
        aggregate_id: Uuid,
        from_version: u64,
    ) -> Vec<BankAccountEvent> {
        let store = self.events.read().unwrap();
        store
            .get(&aggregate_id)
            .map(|events| events.iter().skip(from_version as usize).cloned().collect())
            .unwrap_or_default()
    }
}

#[derive(Debug, thiserror::Error)]
pub enum EventStoreError {
    #[error("Concurrency conflict: expected version {expected}, actual {actual}")]
    ConcurrencyConflict { expected: u64, actual: u64 },
    #[error("Aggregate not found: {0}")]
    AggregateNotFound(Uuid),
    #[error("Persistence error: {0}")]
    Persistence(String),
}
```

### Event Store — PostgreSQL

```rust
pub struct PostgresEventStore {
    pool: PgPool,
}

impl PostgresEventStore {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    pub async fn append_events(
        &self,
        aggregate_id: Uuid,
        events: &[BankAccountEvent],
        expected_version: u64,
    ) -> Result<(), EventStoreError> {
        let mut tx = self.pool.begin().await
            .map_err(|e| EventStoreError::Persistence(e.to_string()))?;

        // Check current version with row lock
        let current_version: i64 = sqlx::query_scalar!(
            "SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1 FOR UPDATE",
            aggregate_id
        )
        .fetch_one(&mut *tx)
        .await
        .map_err(|e| EventStoreError::Persistence(e.to_string()))?
        .unwrap_or(0);

        if current_version as u64 != expected_version {
            return Err(EventStoreError::ConcurrencyConflict {
                expected: expected_version,
                actual: current_version as u64,
            });
        }

        // Insert new events
        for (i, event) in events.iter().enumerate() {
            let version = expected_version + i as u64 + 1;
            let event_data = serde_json::to_value(event)
                .map_err(|e| EventStoreError::Persistence(e.to_string()))?;

            sqlx::query!(
                r#"
                INSERT INTO events (aggregate_id, version, event_type, event_data, occurred_at)
                VALUES ($1, $2, $3, $4, $5)
                "#,
                aggregate_id,
                version as i64,
                event_type_name(event),
                event_data,
                event.occurred_at()
            )
            .execute(&mut *tx)
            .await
            .map_err(|e| EventStoreError::Persistence(e.to_string()))?;
        }

        tx.commit().await
            .map_err(|e| EventStoreError::Persistence(e.to_string()))?;

        Ok(())
    }

    pub async fn get_events(&self, aggregate_id: Uuid) -> Result<Vec<BankAccountEvent>, EventStoreError> {
        let rows = sqlx::query!(
            "SELECT event_data FROM events WHERE aggregate_id = $1 ORDER BY version ASC",
            aggregate_id
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| EventStoreError::Persistence(e.to_string()))?;

        rows.into_iter()
            .map(|row| {
                serde_json::from_value(row.event_data)
                    .map_err(|e| EventStoreError::Persistence(e.to_string()))
            })
            .collect()
    }

    pub async fn get_events_from_version(
        &self,
        aggregate_id: Uuid,
        from_version: u64,
    ) -> Result<Vec<BankAccountEvent>, EventStoreError> {
        let rows = sqlx::query!(
            "SELECT event_data FROM events WHERE aggregate_id = $1 AND version > $2 ORDER BY version ASC",
            aggregate_id,
            from_version as i64
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| EventStoreError::Persistence(e.to_string()))?;

        rows.into_iter()
            .map(|row| {
                serde_json::from_value(row.event_data)
                    .map_err(|e| EventStoreError::Persistence(e.to_string()))
            })
            .collect()
    }
}

fn event_type_name(event: &BankAccountEvent) -> &'static str {
    match event {
        BankAccountEvent::AccountCreated { .. } => "AccountCreated",
        BankAccountEvent::DepositMade { .. } => "DepositMade",
        BankAccountEvent::WithdrawalMade { .. } => "WithdrawalMade",
        BankAccountEvent::AccountClosed { .. } => "AccountClosed",
    }
}
```

### Snapshots

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BankAccountSnapshot {
    pub aggregate_id: Uuid,
    pub version: u64,
    pub holder_name: String,
    pub balance: Decimal,
    pub currency: String,
    pub status: AccountStatus,
    pub created_at: DateTime<Utc>,
}

impl BankAccountSnapshot {
    pub fn from_account(account: &BankAccount) -> Self {
        Self {
            aggregate_id: account.id,
            version: account.version,
            holder_name: account.holder_name.clone(),
            balance: account.balance,
            currency: account.currency.clone(),
            status: account.status.clone(),
            created_at: Utc::now(),
        }
    }

    pub fn into_account(self) -> BankAccount {
        BankAccount {
            id: self.aggregate_id,
            holder_name: self.holder_name,
            balance: self.balance,
            currency: self.currency,
            status: self.status,
            version: self.version,
        }
    }
}

pub struct SnapshotRepository {
    pool: PgPool,
}

impl SnapshotRepository {
    pub async fn save_snapshot(&self, snapshot: &BankAccountSnapshot) -> Result<(), sqlx::Error> {
        let data = serde_json::to_value(snapshot).unwrap();

        sqlx::query!(
            r#"
            INSERT INTO snapshots (aggregate_id, version, data, created_at)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (aggregate_id) DO UPDATE SET
                version = $2, data = $3, created_at = $4
            "#,
            snapshot.aggregate_id,
            snapshot.version as i64,
            data,
            snapshot.created_at
        )
        .execute(&self.pool)
        .await?;

        Ok(())
    }

    pub async fn get_snapshot(&self, aggregate_id: Uuid) -> Result<Option<BankAccountSnapshot>, sqlx::Error> {
        let row = sqlx::query!(
            "SELECT data FROM snapshots WHERE aggregate_id = $1",
            aggregate_id
        )
        .fetch_optional(&self.pool)
        .await?;

        Ok(row.map(|r| serde_json::from_value(r.data).unwrap()))
    }
}
```

### EventSourcedRepository (Snapshot + Events)

```rust
pub struct EventSourcedRepository {
    event_store: PostgresEventStore,
    snapshot_repo: SnapshotRepository,
    snapshot_threshold: u64,  // Create snapshot every N events
}

impl EventSourcedRepository {
    /// Load aggregate using snapshot + recent events
    pub async fn load(&self, aggregate_id: Uuid) -> Result<Option<BankAccount>, EventStoreError> {
        let snapshot = self.snapshot_repo.get_snapshot(aggregate_id).await
            .map_err(|e| EventStoreError::Persistence(e.to_string()))?;

        let (account, from_version) = match snapshot {
            Some(snap) => {
                let version = snap.version;
                (snap.into_account(), version)
            }
            None => (BankAccount::default(), 0),
        };

        let events = self.event_store.get_events_from_version(aggregate_id, from_version).await?;

        if from_version == 0 && events.is_empty() {
            return Ok(None);  // Aggregate doesn't exist
        }

        let mut account = account;
        for event in &events {
            account.apply_event(event);
        }

        Ok(Some(account))
    }

    /// Save events and potentially create snapshot
    pub async fn save(
        &self,
        aggregate_id: Uuid,
        events: &[BankAccountEvent],
        expected_version: u64,
    ) -> Result<(), EventStoreError> {
        self.event_store.append_events(aggregate_id, events, expected_version).await?;

        // Check if we should create a snapshot
        let new_version = expected_version + events.len() as u64;
        if new_version % self.snapshot_threshold == 0 {
            if let Some(account) = self.load(aggregate_id).await? {
                let snapshot = BankAccountSnapshot::from_account(&account);
                self.snapshot_repo.save_snapshot(&snapshot).await
                    .map_err(|e| EventStoreError::Persistence(e.to_string()))?;
            }
        }

        Ok(())
    }
}
```

### Event Versioning (Schema Evolution)

```rust
/// Version 1 of deposit event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DepositMadeV1 {
    pub account_id: Uuid,
    pub amount: Decimal,
    pub occurred_at: DateTime<Utc>,
}

/// Version 2 adds description field
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DepositMadeV2 {
    pub account_id: Uuid,
    pub amount: Decimal,
    pub description: String,
    pub occurred_at: DateTime<Utc>,
}

/// Upcaster converts old event versions to current
pub trait EventUpcaster {
    type Output;
    fn upcast(self) -> Self::Output;
}

impl EventUpcaster for DepositMadeV1 {
    type Output = DepositMadeV2;

    fn upcast(self) -> Self::Output {
        DepositMadeV2 {
            account_id: self.account_id,
            amount: self.amount,
            description: "Legacy deposit".to_string(),  // Default value
            occurred_at: self.occurred_at,
        }
    }
}

/// Deserialize with version handling
pub fn deserialize_deposit_event(
    version: u32,
    data: &serde_json::Value,
) -> Result<DepositMadeV2, serde_json::Error> {
    match version {
        1 => {
            let v1: DepositMadeV1 = serde_json::from_value(data.clone())?;
            Ok(v1.upcast())
        }
        2 => serde_json::from_value(data.clone()),
        _ => Err(serde::de::Error::custom(format!("Unknown version: {}", version))),
    }
}
```

### Command Processing Service

```rust
pub struct AccountService {
    repository: EventSourcedRepository,
}

impl AccountService {
    /// Process a deposit command
    pub async fn deposit(
        &self,
        account_id: Uuid,
        amount: Decimal,
        description: String,
    ) -> Result<Decimal, AccountServiceError> {
        // 1. Load current state from events
        let account = self.repository.load(account_id).await?
            .ok_or(AccountServiceError::AccountNotFound(account_id))?;

        // 2. Execute business logic and get new event
        let event = account.deposit(amount, description)?;

        // 3. Apply event to get new balance (for response)
        let mut updated = account.clone();
        updated.apply_event(&event);

        // 4. Persist the event
        self.repository.save(account_id, &[event], account.version()).await?;

        // 5. Return result
        Ok(updated.balance())
    }

    /// Create new account
    pub async fn create_account(
        &self,
        holder_name: String,
        initial_balance: Decimal,
        currency: String,
    ) -> Result<Uuid, AccountServiceError> {
        let account_id = Uuid::new_v4();
        let event = BankAccount::create(account_id, holder_name, initial_balance, currency)?;
        self.repository.save(account_id, &[event], 0).await?;
        Ok(account_id)
    }
}

#[derive(Debug, thiserror::Error)]
pub enum AccountServiceError {
    #[error("Account not found: {0}")]
    AccountNotFound(Uuid),
    #[error(transparent)]
    Domain(#[from] AccountError),
    #[error(transparent)]
    EventStore(#[from] EventStoreError),
}
```

## CQRS (Command Query Responsibility Segregation)

Separate read and write operations with distinct handlers, models, and data stores.

### Command and Query Traits

```rust
use async_trait::async_trait;

/// Commands represent intent to change state
pub trait Command: Send + Sync {
    type Result: Send;
}

#[async_trait]
pub trait CommandHandler<C: Command>: Send + Sync {
    async fn handle(&self, command: C) -> Result<C::Result, CommandError>;
}

#[derive(Debug, thiserror::Error)]
pub enum CommandError {
    #[error("Entity not found: {0}")]
    NotFound(String),
    #[error("Validation failed: {0}")]
    Validation(String),
    #[error("Conflict: {0}")]
    Conflict(String),
    #[error("Infrastructure error: {0}")]
    Infrastructure(#[from] Box<dyn std::error::Error + Send + Sync>),
}

/// Queries represent requests for data
pub trait Query: Send + Sync {
    type Result: Send;
}

#[async_trait]
pub trait QueryHandler<Q: Query>: Send + Sync {
    async fn handle(&self, query: Q) -> Result<Q::Result, QueryError>;
}

#[derive(Debug, thiserror::Error)]
pub enum QueryError {
    #[error("Not found: {0}")]
    NotFound(String),
    #[error("Query error: {0}")]
    QueryFailed(String),
}
```

### Command Handlers

```rust
// CreateUser command
#[derive(Debug, Clone)]
pub struct CreateUserCommand {
    pub username: String,
    pub email: String,
}

impl Command for CreateUserCommand {
    type Result = Uuid;
}

pub struct CreateUserHandler {
    user_repo: Arc<dyn UserRepository>,
    event_publisher: Arc<dyn EventPublisher>,
}

#[async_trait]
impl CommandHandler<CreateUserCommand> for CreateUserHandler {
    async fn handle(&self, cmd: CreateUserCommand) -> Result<Uuid, CommandError> {
        if cmd.username.is_empty() {
            return Err(CommandError::Validation("Username required".into()));
        }
        if !cmd.email.contains('@') {
            return Err(CommandError::Validation("Invalid email".into()));
        }
        if self.user_repo.exists_by_email(&cmd.email).await? {
            return Err(CommandError::Conflict("Email already registered".into()));
        }

        let user = User::new(cmd.username, cmd.email);
        let user_id = user.id;
        self.user_repo.save(&user).await?;

        self.event_publisher.publish(UserCreatedEvent {
            user_id,
            username: user.username.clone(),
            email: user.email.clone(),
            created_at: chrono::Utc::now(),
        }).await?;

        Ok(user_id)
    }
}

// UpdateUser command
#[derive(Debug, Clone)]
pub struct UpdateUserCommand {
    pub user_id: Uuid,
    pub new_email: Option<String>,
    pub new_username: Option<String>,
}

impl Command for UpdateUserCommand {
    type Result = ();
}

pub struct UpdateUserHandler {
    user_repo: Arc<dyn UserRepository>,
    event_publisher: Arc<dyn EventPublisher>,
}

#[async_trait]
impl CommandHandler<UpdateUserCommand> for UpdateUserHandler {
    async fn handle(&self, cmd: UpdateUserCommand) -> Result<(), CommandError> {
        let mut user = self.user_repo
            .find_by_id(cmd.user_id)
            .await?
            .ok_or_else(|| CommandError::NotFound(cmd.user_id.to_string()))?;

        if let Some(email) = cmd.new_email {
            user.change_email(email)?;
        }
        if let Some(username) = cmd.new_username {
            user.change_username(username)?;
        }

        self.user_repo.save(&user).await?;

        self.event_publisher.publish(UserUpdatedEvent {
            user_id: cmd.user_id,
            updated_at: chrono::Utc::now(),
        }).await?;

        Ok(())
    }
}
```

### Query Handlers

```rust
// GetUserById query
#[derive(Debug, Clone)]
pub struct GetUserByIdQuery {
    pub user_id: Uuid,
}

impl Query for GetUserByIdQuery {
    type Result = UserReadModel;
}

// Read model (DTO optimized for queries)
#[derive(Debug, Clone, Serialize)]
pub struct UserReadModel {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
    pub order_count: u32,      // Denormalized for performance
    pub last_order_date: Option<DateTime<Utc>>,
}

pub struct GetUserByIdHandler {
    read_store: Arc<dyn UserReadStore>,
}

#[async_trait]
impl QueryHandler<GetUserByIdQuery> for GetUserByIdHandler {
    async fn handle(&self, query: GetUserByIdQuery) -> Result<UserReadModel, QueryError> {
        self.read_store
            .find_by_id(query.user_id)
            .await
            .map_err(|e| QueryError::QueryFailed(e.to_string()))?
            .ok_or_else(|| QueryError::NotFound(query.user_id.to_string()))
    }
}

// ListUsers query with pagination
#[derive(Debug, Clone)]
pub struct ListUsersQuery {
    pub page: u32,
    pub page_size: u32,
    pub filter: Option<UserFilter>,
}

#[derive(Debug, Clone)]
pub struct UserFilter {
    pub username_contains: Option<String>,
    pub created_after: Option<DateTime<Utc>>,
}

impl Query for ListUsersQuery {
    type Result = PaginatedResult<UserSummary>;
}

#[derive(Debug, Clone, Serialize)]
pub struct UserSummary {
    pub id: Uuid,
    pub username: String,
    pub email: String,
}

#[derive(Debug, Clone, Serialize)]
pub struct PaginatedResult<T> {
    pub items: Vec<T>,
    pub total_count: u64,
    pub page: u32,
    pub page_size: u32,
    pub has_next: bool,
}

pub struct ListUsersHandler {
    read_store: Arc<dyn UserReadStore>,
}

#[async_trait]
impl QueryHandler<ListUsersQuery> for ListUsersHandler {
    async fn handle(&self, query: ListUsersQuery) -> Result<PaginatedResult<UserSummary>, QueryError> {
        let offset = (query.page - 1) * query.page_size;

        let (items, total) = self.read_store
            .list(offset, query.page_size, query.filter)
            .await
            .map_err(|e| QueryError::QueryFailed(e.to_string()))?;

        Ok(PaginatedResult {
            has_next: (offset + items.len() as u32) < total as u32,
            items,
            total_count: total,
            page: query.page,
            page_size: query.page_size,
        })
    }
}
```

### Read Models and Projections

```rust
// Write repository (for commands) — domain entities
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>, RepositoryError>;
    async fn save(&self, user: &User) -> Result<(), RepositoryError>;
    async fn exists_by_email(&self, email: &str) -> Result<bool, RepositoryError>;
}

// Read store (for queries) — optimized read models
#[async_trait]
pub trait UserReadStore: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<UserReadModel>, ReadStoreError>;
    async fn list(
        &self,
        offset: u32,
        limit: u32,
        filter: Option<UserFilter>,
    ) -> Result<(Vec<UserSummary>, u64), ReadStoreError>;
}

/// Projection updates read models based on domain events
pub struct UserProjection {
    read_store: Arc<dyn UserReadStoreWriter>,
}

impl UserProjection {
    pub async fn apply(&self, event: &DomainEvent) -> Result<(), ProjectionError> {
        match event {
            DomainEvent::UserCreated(e) => {
                let read_model = UserReadModel {
                    id: e.user_id,
                    username: e.username.clone(),
                    email: e.email.clone(),
                    created_at: e.created_at,
                    order_count: 0,
                    last_order_date: None,
                };
                self.read_store.insert(read_model).await?;
            }
            DomainEvent::UserUpdated(e) => {
                self.read_store.update_user(e.user_id, |model| {
                    if let Some(email) = &e.new_email {
                        model.email = email.clone();
                    }
                    if let Some(username) = &e.new_username {
                        model.username = username.clone();
                    }
                }).await?;
            }
            DomainEvent::OrderPlaced(e) => {
                // Update denormalized order count
                self.read_store.update_user(e.user_id, |model| {
                    model.order_count += 1;
                    model.last_order_date = Some(e.placed_at);
                }).await?;
            }
            _ => {}
        }
        Ok(())
    }
}

#[async_trait]
pub trait UserReadStoreWriter: Send + Sync {
    async fn insert(&self, model: UserReadModel) -> Result<(), ReadStoreError>;
    async fn update_user<F>(&self, id: Uuid, f: F) -> Result<(), ReadStoreError>
    where
        F: FnOnce(&mut UserReadModel) + Send;
}
```

### Eventual Consistency

```rust
/// Event consumer that updates read models with retry and dead letter
pub struct EventConsumer {
    projection: Arc<UserProjection>,
    consumer: Arc<dyn MessageConsumer>,
}

impl EventConsumer {
    pub async fn run(&self) -> Result<(), ConsumerError> {
        loop {
            let event = self.consumer.receive().await?;

            let result = retry_with_backoff(3, || async {
                self.projection.apply(&event).await
            }).await;

            match result {
                Ok(()) => {
                    self.consumer.acknowledge(&event).await?;
                }
                Err(e) => {
                    tracing::error!("Failed to process event: {}", e);
                    // Send to dead letter queue
                    self.consumer.reject(&event).await?;
                }
            }
        }
    }
}
```

### Idempotent Consumers

```rust
/// Ensures events are processed exactly once
pub struct IdempotentProjection {
    inner: Arc<UserProjection>,
    processed_events: Arc<dyn ProcessedEventStore>,
}

#[async_trait]
impl Projection for IdempotentProjection {
    async fn apply(&self, event: &DomainEvent) -> Result<(), ProjectionError> {
        let event_id = event.id();

        // Check if already processed
        if self.processed_events.contains(event_id).await? {
            return Ok(());  // Skip duplicate
        }

        // Process event
        self.inner.apply(event).await?;

        // Mark as processed
        self.processed_events.mark_processed(event_id).await?;

        Ok(())
    }
}

#[async_trait]
pub trait ProcessedEventStore: Send + Sync {
    async fn contains(&self, event_id: Uuid) -> Result<bool, StoreError>;
    async fn mark_processed(&self, event_id: Uuid) -> Result<(), StoreError>;
}
```

### Read-Your-Own-Writes Pattern

For better UX, update local state immediately while eventual consistency propagates:

```rust
pub struct CqrsMediator {
    command_handlers: HashMap<TypeId, Arc<dyn Any + Send + Sync>>,
    query_handlers: HashMap<TypeId, Arc<dyn Any + Send + Sync>>,
    local_cache: Arc<RwLock<HashMap<Uuid, UserReadModel>>>,
}

impl CqrsMediator {
    /// Execute command and update local cache for immediate reads
    pub async fn execute_command<C: Command + 'static>(
        &self,
        command: C,
    ) -> Result<C::Result, CommandError>
    where
        C::Result: Into<Option<ReadModelUpdate>>,
    {
        let handler = self.get_command_handler::<C>()?;
        let result = handler.handle(command).await?;

        // Immediately update local cache
        if let Some(update) = result.clone().into() {
            self.apply_local_update(update).await;
        }

        Ok(result)
    }

    /// Query with local cache fallback for recently written data
    pub async fn execute_query<Q: Query + 'static>(
        &self,
        query: Q,
    ) -> Result<Q::Result, QueryError> {
        // Check local cache first for recently written data
        if let Some(cached) = self.check_local_cache(&query).await {
            return Ok(cached);
        }

        // Fall back to read store
        let handler = self.get_query_handler::<Q>()?;
        handler.handle(query).await
    }
}
```

### Command/Query Dispatcher

```rust
use std::any::{Any, TypeId};

pub struct CqrsDispatcher {
    command_handlers: HashMap<TypeId, Box<dyn Any + Send + Sync>>,
    query_handlers: HashMap<TypeId, Box<dyn Any + Send + Sync>>,
}

impl CqrsDispatcher {
    pub fn new() -> Self {
        Self {
            command_handlers: HashMap::new(),
            query_handlers: HashMap::new(),
        }
    }

    pub fn register_command_handler<C, H>(&mut self, handler: H)
    where
        C: Command + 'static,
        H: CommandHandler<C> + 'static,
    {
        self.command_handlers.insert(TypeId::of::<C>(), Box::new(handler));
    }

    pub fn register_query_handler<Q, H>(&mut self, handler: H)
    where
        Q: Query + 'static,
        H: QueryHandler<Q> + 'static,
    {
        self.query_handlers.insert(TypeId::of::<Q>(), Box::new(handler));
    }

    pub async fn dispatch_command<C: Command + 'static>(
        &self,
        command: C,
    ) -> Result<C::Result, CommandError> {
        let handler = self.command_handlers
            .get(&TypeId::of::<C>())
            .ok_or_else(|| CommandError::Validation("No handler registered".into()))?;

        let handler = handler
            .downcast_ref::<Box<dyn CommandHandler<C>>>()
            .expect("Type mismatch");

        handler.handle(command).await
    }

    pub async fn dispatch_query<Q: Query + 'static>(
        &self,
        query: Q,
    ) -> Result<Q::Result, QueryError> {
        let handler = self.query_handlers
            .get(&TypeId::of::<Q>())
            .ok_or_else(|| QueryError::QueryFailed("No handler registered".into()))?;

        let handler = handler
            .downcast_ref::<Box<dyn QueryHandler<Q>>>()
            .expect("Type mismatch");

        handler.handle(query).await
    }
}
```

### Web Framework Integration

```rust
use actix_web::{web, HttpResponse, post, get};

#[post("/users")]
async fn create_user(
    dispatcher: web::Data<Arc<CqrsDispatcher>>,
    body: web::Json<CreateUserRequest>,
) -> HttpResponse {
    let command = CreateUserCommand {
        username: body.username.clone(),
        email: body.email.clone(),
    };

    match dispatcher.dispatch_command(command).await {
        Ok(user_id) => HttpResponse::Created().json(json!({ "id": user_id })),
        Err(CommandError::Validation(msg)) => {
            HttpResponse::BadRequest().json(json!({ "error": msg }))
        }
        Err(CommandError::Conflict(msg)) => {
            HttpResponse::Conflict().json(json!({ "error": msg }))
        }
        Err(e) => {
            HttpResponse::InternalServerError().json(json!({ "error": e.to_string() }))
        }
    }
}

#[get("/users/{id}")]
async fn get_user(
    dispatcher: web::Data<Arc<CqrsDispatcher>>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    let query = GetUserByIdQuery { user_id: *path };

    match dispatcher.dispatch_query(query).await {
        Ok(user) => HttpResponse::Ok().json(user),
        Err(QueryError::NotFound(_)) => HttpResponse::NotFound().finish(),
        Err(e) => {
            HttpResponse::InternalServerError().json(json!({ "error": e.to_string() }))
        }
    }
}

#[get("/users")]
async fn list_users(
    dispatcher: web::Data<Arc<CqrsDispatcher>>,
    query_params: web::Query<ListUsersParams>,
) -> HttpResponse {
    let query = ListUsersQuery {
        page: query_params.page.unwrap_or(1),
        page_size: query_params.page_size.unwrap_or(20).min(100),
        filter: query_params.username.as_ref().map(|u| UserFilter {
            username_contains: Some(u.clone()),
            created_after: None,
        }),
    };

    match dispatcher.dispatch_query(query).await {
        Ok(result) => HttpResponse::Ok().json(result),
        Err(e) => {
            HttpResponse::InternalServerError().json(json!({ "error": e.to_string() }))
        }
    }
}
```

### When to Use CQRS

**Use when:** Read/write patterns differ significantly, need independent scaling, complex queries need denormalized data, using event sourcing, multiple teams work on different parts.

**Avoid when:** Simple CRUD suffices, immediate consistency required everywhere, complexity not justified by scale.

## Testing

### Testing Domain Events

```rust
#[cfg(test)]
mod event_tests {
    use super::*;

    #[test]
    fn user_verify_email_records_event() {
        let mut user = User::new(Uuid::new_v4(), "test@example.com".to_string());

        assert!(!user.has_uncommitted_events());

        user.verify_email("test@example.com").unwrap();

        assert!(user.has_uncommitted_events());
        assert!(user.is_email_verified());

        let events = user.take_uncommitted_events();
        assert_eq!(events.len(), 1);
        assert_eq!(events[0].event_type(), "UserEmailVerified");

        // Events are cleared after take
        assert!(!user.has_uncommitted_events());
    }

    #[test]
    fn verify_already_verified_returns_error() {
        let mut user = User::new(Uuid::new_v4(), "test@example.com".to_string());
        user.verify_email("test@example.com").unwrap();

        let result = user.verify_email("test@example.com");
        assert!(result.is_err());
    }

    #[tokio::test]
    async fn event_bus_delivers_to_subscribers() {
        let bus = EventBus::new(10);
        let mut rx = bus.subscribe();

        let event = DomainEventEnvelope::UserEmailVerified(
            UserEmailVerified::new(Uuid::new_v4(), "test@example.com".to_string())
        );

        bus.publish(event.clone()).await.unwrap();

        let received = rx.recv().await.unwrap();
        assert!(matches!(received, DomainEventEnvelope::UserEmailVerified(_)));
    }
}
```

### Testing Event Sourcing

```rust
#[cfg(test)]
mod event_sourcing_tests {
    use super::*;
    use rust_decimal_macros::dec;

    #[test]
    fn account_reconstructs_from_events() {
        let account_id = Uuid::new_v4();
        let events = vec![
            BankAccountEvent::AccountCreated {
                account_id,
                holder_name: "Alice".to_string(),
                initial_balance: dec!(100),
                currency: "USD".to_string(),
                created_at: Utc::now(),
            },
            BankAccountEvent::DepositMade {
                account_id,
                amount: dec!(50),
                description: "Paycheck".to_string(),
                occurred_at: Utc::now(),
            },
            BankAccountEvent::WithdrawalMade {
                account_id,
                amount: dec!(30),
                description: "Groceries".to_string(),
                occurred_at: Utc::now(),
            },
        ];

        let account = BankAccount::from_events(&events);

        assert_eq!(account.balance(), dec!(120));  // 100 + 50 - 30
        assert_eq!(account.version(), 3);
    }

    #[test]
    fn withdraw_insufficient_funds_fails() {
        let account = BankAccount::from_events(&[
            BankAccountEvent::AccountCreated {
                account_id: Uuid::new_v4(),
                holder_name: "Bob".to_string(),
                initial_balance: dec!(50),
                currency: "USD".to_string(),
                created_at: Utc::now(),
            },
        ]);

        let result = account.withdraw(dec!(100), "Too much".to_string());
        assert!(matches!(result, Err(AccountError::InsufficientFunds { .. })));
    }

    #[test]
    fn closed_account_rejects_operations() {
        let account_id = Uuid::new_v4();
        let account = BankAccount::from_events(&[
            BankAccountEvent::AccountCreated {
                account_id,
                holder_name: "Carol".to_string(),
                initial_balance: dec!(100),
                currency: "USD".to_string(),
                created_at: Utc::now(),
            },
            BankAccountEvent::AccountClosed {
                account_id,
                reason: "Customer request".to_string(),
                closed_at: Utc::now(),
            },
        ]);

        assert!(account.deposit(dec!(10), "Test".to_string()).is_err());
        assert!(account.withdraw(dec!(10), "Test".to_string()).is_err());
    }
}
```

### Testing Across Contexts

```rust
#[cfg(test)]
mod integration_tests {
    use super::*;
    use mockall::mock;

    mock! {
        InventoryService {}

        #[async_trait]
        impl InventoryService for InventoryService {
            async fn check_availability(
                &self,
                product_id: ProductId,
                quantity: u32,
            ) -> Result<bool, InventoryError>;

            async fn reserve_stock(
                &self,
                product_id: ProductId,
                quantity: u32,
            ) -> Result<ReservationId, InventoryError>;

            async fn release_reservation(
                &self,
                reservation_id: ReservationId,
            ) -> Result<(), InventoryError>;
        }
    }

    #[tokio::test]
    async fn place_order_reserves_stock() {
        let mut mock_inventory = MockInventoryService::new();

        mock_inventory
            .expect_check_availability()
            .returning(|_, _| Ok(true));

        mock_inventory
            .expect_reserve_stock()
            .returning(|_, _| Ok(ReservationId(Uuid::new_v4())));

        let use_case = PlaceOrderUseCase {
            inventory_service: Arc::new(mock_inventory),
            // ... other dependencies
        };

        let result = use_case.execute(test_input()).await;
        assert!(result.is_ok());
    }

    #[tokio::test]
    async fn place_order_releases_reservation_on_payment_failure() {
        let mut mock_inventory = MockInventoryService::new();

        mock_inventory
            .expect_check_availability()
            .returning(|_, _| Ok(true));

        mock_inventory
            .expect_reserve_stock()
            .returning(|_, _| Ok(ReservationId(Uuid::new_v4())));

        // Expect release to be called when payment fails
        mock_inventory
            .expect_release_reservation()
            .times(1)  // Must be called exactly once
            .returning(|_| Ok(()));

        let mut mock_payment = MockPaymentGateway::new();
        mock_payment
            .expect_charge()
            .returning(|_, _| Err(PaymentError::Declined));

        // ... test verifies reservations are released on payment failure
    }
}
```

### Testing CQRS Handlers

```rust
#[cfg(test)]
mod cqrs_tests {
    use super::*;
    use mockall::mock;

    mock! {
        UserRepo {}
        #[async_trait]
        impl UserRepository for UserRepo {
            async fn find_by_id(&self, id: Uuid) -> Result<Option<User>, RepositoryError>;
            async fn save(&self, user: &User) -> Result<(), RepositoryError>;
            async fn exists_by_email(&self, email: &str) -> Result<bool, RepositoryError>;
        }
    }

    #[tokio::test]
    async fn create_user_success() {
        let mut repo = MockUserRepo::new();
        repo.expect_exists_by_email()
            .returning(|_| Ok(false));
        repo.expect_save()
            .returning(|_| Ok(()));

        let mut publisher = MockEventPub::new();
        publisher.expect_publish()
            .returning(|_: UserCreatedEvent| Ok(()));

        let handler = CreateUserHandler::new(
            Arc::new(repo),
            Arc::new(publisher),
        );

        let cmd = CreateUserCommand {
            username: "testuser".to_string(),
            email: "test@example.com".to_string(),
        };

        let result = handler.handle(cmd).await;
        assert!(result.is_ok());
    }

    #[tokio::test]
    async fn create_user_duplicate_email_returns_conflict() {
        let mut repo = MockUserRepo::new();
        repo.expect_exists_by_email()
            .returning(|_| Ok(true));  // Email exists

        let publisher = MockEventPub::new();

        let handler = CreateUserHandler::new(
            Arc::new(repo),
            Arc::new(publisher),
        );

        let cmd = CreateUserCommand {
            username: "testuser".to_string(),
            email: "existing@example.com".to_string(),
        };

        let result = handler.handle(cmd).await;
        assert!(matches!(result, Err(CommandError::Conflict(_))));
    }
}
```

## Best Practices

1. **Events are immutable facts** — once created, never modify event data
2. **Past tense naming** — `OrderPlaced`, not `PlaceOrder` (that's a command)
3. **Self-contained events** — include all data handlers need
4. **Drain events after save** — use `take_uncommitted_events()` pattern
5. **Handle subscriber lag** — check `RecvError::Lagged` for missed events
6. **Use outbox for reliability** — atomic save-and-publish with database transactions
7. **Snapshot periodically** — every 100-500 events to avoid long replays
8. **Version events** — upcasters convert old formats to current schema
9. **Domain errors are pure** — no infrastructure types in domain layer
10. **Translate at boundaries** — convert errors when crossing layer boundaries

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: traits, enums, error handling, ownership patterns
- **[architecture.md](architecture.md)** — Workspace design, application layering, DI patterns
- **[error-handling.md](error-handling.md)** — Domain error types, multi-layer error translation
- **[services.md](services.md)** — Inter-service communication, event bus, distributed patterns
- **[testing.md](testing.md)** — Aggregate testing, mock repositories, event assertion patterns
- **[serde-serialization.md](serde-serialization.md)** — Event serialization, schema versioning
