# Application Architecture in Rust

Workspace design, SOLID principles, hexagonal/onion/clean architecture, layered design, state management, and production patterns — all mapped to Rust's type system.

For extended worked examples, see [architecture-examples.md](architecture-examples.md).
For database integration and caching, see [database.md](database.md).
For event sourcing, CQRS, and DDD, see [domain-patterns.md](domain-patterns.md).

## Section Index

| Section | Key Content |
|---------|------------|
| [Architectural Principles](#architectural-principles) | 10 foundational rules governing all structural decisions |
| [Architecture Decision Rules (LLM)](#architecture-decision-rules-llm) | 17 ALWAYS/NEVER rules for LLM architectural guidance |
| [SOLID Principles](#solid-principles-in-rust) | SRP, OCP, LSP, ISP, DIP mapped to Rust traits and modules |
| [Hexagonal / Onion / Clean Architecture](#hexagonal-architecture-ports--adapters) | Ports = traits, adapters = implementations, comparison table |
| [DDD in Rust](#domain-driven-design-ddd-in-rust) | Bounded contexts as workspaces, entities, value objects, aggregates |
| [Event Sourcing & Architecture](#event-sourcing--architecture) | Aggregates, event store trait, state rebuild |
| [Cargo Workspace Organization](#cargo-workspace-organization) | Workspace layout, member crates, features, visibility, feature-gated server roles |
| [Trait-Based Dependency Inversion](#trait-based-dependency-inversion) | Repository pattern, generics vs trait objects, injection patterns |
| [Application Layering](#application-layering) | Domain → Application → Infrastructure with concrete code per layer |
| [State Management & Configuration](#state-management--configuration) | Config loading, LazyLock, connection pools |
| [Production Patterns](#production-patterns) | Graceful shutdown, health checks, tracing, metrics, idempotency |
| [Testing Architecture](#testing-architecture) | Domain tests (no mocks), application tests (mocked ports) |
| [Growing Architecture](#growing-architecture--small-to-large) | Small → medium → large progression with concrete examples |
| [Inter-Component Communication](#inter-component-communication) | Channels, shared state, message passing — when to use which |
| [Refactoring Signals](#refactoring-signals) | Signal → Refactoring → How table for common architectural smells |
| [Anti-Patterns Catalog](#anti-patterns-catalog) | Layering, state, async, DI, testing anti-patterns with fixes |
| [DI Containers](#di-containers) | Manual, TypeId, Shaku — from simplest to most structured |
| [Domain Modeling Patterns](#domain-modeling-patterns) | State machines, DTOs, presenters, mock repos |
| [Resilience Patterns](#resilience-patterns) | Retry, circuit breaker, graceful degradation |
| [Authorization Patterns](#authorization-patterns) | RBAC, domain guards, middleware, multi-tenant isolation |
| [High-Throughput Ingestion](#high-throughput-ingestion-sensor--iot-apis) | Buffered channels, batch writes, backpressure for sensor/IoT APIs |
| [Nanoservices Architecture](#nanoservices-architecture) | Monolith-to-microservices workspace pattern |
| [Async Logging Architecture](#async-logging-architecture) | tracing, log levels, middleware, actor-based remote logging |
| [Facade Crate Pattern](#facade-crate-pattern) | Re-export multiple subcrates through single entry point (ripgrep) |
| [Enum-Based Polymorphism](#enum-based-polymorphism-vs-dyn-trait) | Enum dispatch vs `dyn Trait` — when each is better (ripgrep) |
| [Tower Layer/Service Composition](#tower-layerservice-composition-axum-architecture) | Layer→Service model, `from_fn`, `map_request`, state erasure (axum) |
| [Workspace Lint Inheritance](#workspace-lint-inheritance) | Centralized lint config at workspace level (axum) |
| [Two-Stage Argument Parsing](#two-stage-argument-parsing-ripgrep-pattern) | LowArgs → HiArgs two-phase CLI parsing (ripgrep) |

## Architectural Principles

These principles govern all structural decisions in Rust applications. When patterns conflict or requirements are ambiguous, refer back here.

1. **Dependencies point inward.** Infrastructure depends on Application. Application depends on Domain. Domain depends on nothing external. A domain module must NEVER import `sqlx`, `axum`, `reqwest`, `redis`, or any infrastructure crate. The domain crate's `Cargo.toml` is the proof — if it lists framework crates, the architecture is broken.

2. **Traits are ports. Implementations are adapters.** Every external dependency (database, API, email, file system, message queue) is behind a `trait` defined by the domain. Infrastructure implements the trait. Config or the composition root selects which implementation runs. This IS hexagonal architecture — Rust's trait system has it built in.

3. **The ownership system IS the architecture boundary.** `pub(crate)` enforces aggregate roots — inner entities are invisible outside the crate. Moving an entity into an aggregate transfers ownership — no accidental sharing. Rust's type system encodes architectural decisions that other languages leave to convention.

4. **Cargo.toml encodes layer direction.** `domain/Cargo.toml` has zero infrastructure deps. `infra/Cargo.toml` depends on `domain`. If you need to add `sqlx` to your domain crate, your architecture has a boundary problem. Dependency direction is auditable from `Cargo.toml` alone.

5. **Feature flags are compile-time architecture decisions.** Cargo features let you swap adapters, enable optional subsystems, and gate infrastructure at compile time. Feature-gated code is dead-code-eliminated — zero runtime cost for disabled features.

6. **Start without frameworks, add them at the edges.** Domain logic is plain Rust — no `#[derive(FromRow)]`, no `#[actix_web::get]`, no framework annotations. Framework-specific code lives in the outermost layer only. The litmus test: can you delete the `infrastructure/` and `server/` crates and still compile `domain/` and `application/`?

7. **Design for replaceability.** Can you swap a component's implementation without changing business logic? If not, introduce a trait at the boundary. Can you test a business rule without a database, HTTP server, or external service? If not, your architecture has a boundary problem.

8. **Errors translate at layer boundaries.** Each layer has its own error type. Domain errors are business-meaningful (`OrderNotModifiable`, `InsufficientFunds`). Infrastructure errors are technical (`ConnectionTimeout`, `RowNotFound`). `From` conversions translate between them at layer boundaries. Never surface infrastructure errors to callers.

9. **The composition root wires everything.** `main()` (or a builder in `main()`) creates concrete implementations, injects them into use cases, and starts the server. This is the only place that knows about all concrete types. No service discovers its own dependencies at runtime.

10. **Keep traits small and focused.** No client should depend on methods it doesn't use. If a function only needs `find()`, don't force it to depend on a trait that also defines `save()`, `delete()`, and `export_csv()`. Split into focused traits (`Find<T>`, `Save<T>`). Compose with trait bounds: `impl Find<Order> + Save<Order>`.

## Architecture Decision Rules (LLM)

These rules capture the most critical architectural decisions. When generating or reviewing Rust code, follow these without exception.

1. **NEVER** put business logic in HTTP handlers. They extract input, delegate to a use case, and format the response — nothing else. No validation, no calculations, no conditionals on domain state.
2. **NEVER** let domain crates depend on infrastructure crates. Check `Cargo.toml` — if `domain/` lists `sqlx`, `axum`, `redis`, `reqwest`, or any I/O crate, the architecture is wrong.
3. **NEVER** expose domain entities directly as API responses. Use separate DTOs (`CreateOrderRequest`, `OrderResponse`). Domain entities carry invariants and internal state that callers should not see or depend on.
4. **NEVER** pass the entire `Config` struct to services. Extract only the fields each service needs (`smtp_url`, `from_addr`) — this documents dependencies and enables independent testing.
5. **ALWAYS** define repository and gateway traits in the domain layer. The domain owns the contract; infrastructure implements it. Never define the trait next to its implementation.
6. **ALWAYS** use constructor injection — pass dependencies into `new()` or `build()`. Never use global mutable state, `lazy_static!` service locators, or hidden singletons for dependencies.
7. **ALWAYS** translate errors at layer boundaries. Domain functions return domain errors. Infrastructure adapters convert `sqlx::Error` → `RepoError`, `reqwest::Error` → `GatewayError`. HTTP handlers convert `AppError` → status codes.
8. **ALWAYS** wire dependencies in the composition root (`main()` or a builder). This is the only module that knows all concrete types. No service resolves its own dependencies.
9. **PREFER** generics (`impl OrderRepository`) over trait objects (`Box<dyn OrderRepository>`) when there's only one implementation per compilation target. Generics enable monomorphization and inlining.
10. **PREFER** manual DI (constructor injection in `main()`) over DI containers for applications with fewer than 20 services. Containers add indirection without proportional benefit.
11. **PREFER** `Arc<T>` for sharing services across async tasks over `&'static T` globals. `Arc` makes ownership explicit and enables testing with different instances.
12. **NEVER** scatter `#[cfg(feature = "...")]` throughout domain logic. Feature gates belong in infrastructure and composition layers — domain code should be unconditionally compiled.
13. **ALWAYS** make operations idempotent when they may be retried (webhook handlers, queue consumers, distributed calls). Use idempotency keys or unique constraints.
14. **ALWAYS** set timeouts at every external boundary (HTTP clients, database queries, gRPC calls). Cascade correctly: outer > middle > inner.
15. **NEVER** use `unwrap()` or `expect()` in production paths for I/O results. These are acceptable only in tests, initialization code, and provably-safe contexts.
16. **ALWAYS** use `[workspace.lints]` for multi-crate workspaces. Centralize clippy and rustc lint configuration at the workspace level with `[lints] workspace = true` in each member. Override per-crate only when necessary.
17. **PREFER** enum dispatch over `dyn Trait` when the set of implementations is known at compile time. Enums are faster (no vtable), support `#[cfg]` on variants, and enable exhaustive matching.

## Architectural Patterns & Principles

### SOLID Principles in Rust

SOLID maps naturally to Rust — the type system and trait system enforce several principles at compile time.

**S — Single Responsibility Principle (SRP)**
Each module, struct, and trait has one reason to change. In Rust, this means:
- One struct per concern: `OrderValidator`, `OrderPricer`, `OrderRepository` — not a monolithic `OrderService`
- Traits define single capabilities: `trait Validate`, `trait Price`, `trait Save` — not `trait DoEverything`
- Crates as module boundaries: `domain/`, `application/`, `infra/` — each changes for different reasons

```rust
// BAD — one struct doing validation, pricing, and persistence
struct OrderService { pool: PgPool }
impl OrderService {
    fn validate(&self, order: &Order) -> Result<(), Error> { /* ... */ }
    fn calculate_total(&self, order: &Order) -> Money { /* ... */ }
    async fn save(&self, order: &Order) -> Result<(), Error> { /* ... */ }
}

// GOOD — separate concerns, compose in use case
struct OrderValidator;
impl OrderValidator {
    fn validate(&self, order: &Order) -> Result<(), ValidationError> { /* ... */ }
}

struct OrderPricer { tax_rate: Decimal }
impl OrderPricer {
    fn total(&self, order: &Order) -> Money { /* ... */ }
}

// Use case composes them
struct PlaceOrderUseCase<R: OrderRepository> {
    validator: OrderValidator,
    pricer: OrderPricer,
    repo: R,
}
```

**O — Open/Closed Principle (OCP)**
Open for extension, closed for modification. Rust traits are the primary extension mechanism:

```rust
// Closed for modification — trait interface is stable
trait PaymentProcessor {
    async fn charge(&self, amount: Money, method: &PaymentMethod) -> Result<PaymentId, PaymentError>;
}

// Open for extension — new implementations without changing existing code
struct StripeProcessor { client: stripe::Client }
impl PaymentProcessor for StripeProcessor { /* ... */ }

struct PayPalProcessor { api: paypal::Api }
impl PaymentProcessor for PayPalProcessor { /* ... */ }

// Adding Square support changes ZERO existing code
struct SquareProcessor { /* ... */ }
impl PaymentProcessor for SquareProcessor { /* ... */ }
```

Enums with `#[non_exhaustive]` also support OCP — downstream can't exhaustively match, so adding variants doesn't break callers.

**L — Liskov Substitution Principle (LSP)**
Any implementor of a trait must be a valid substitute. Rust's trait system enforces this syntactically, but semantic LSP is your responsibility:

```rust
// BAD — violates LSP: ReadOnlyRepo panics on save, surprising callers
impl OrderRepository for ReadOnlyRepo {
    async fn save(&self, _: &Order) -> Result<(), RepoError> {
        panic!("read-only!") // Callers expect save to work
    }
}

// GOOD — use separate traits for read vs write
trait OrderReader {
    async fn find_by_id(&self, id: OrderId) -> Result<Option<Order>, RepoError>;
}
trait OrderWriter {
    async fn save(&self, order: &Order) -> Result<(), RepoError>;
}
// ReadOnlyRepo only implements OrderReader — no surprise panics
```

**I — Interface Segregation Principle (ISP)**
Clients shouldn't depend on methods they don't use. In Rust, this means small, focused traits:

```rust
// BAD — fat trait forces implementors to provide everything
trait Repository {
    async fn find(&self, id: Id) -> Result<Entity, Error>;
    async fn save(&self, entity: &Entity) -> Result<(), Error>;
    async fn delete(&self, id: Id) -> Result<(), Error>;
    async fn search(&self, query: &str) -> Result<Vec<Entity>, Error>;
    async fn export_csv(&self) -> Result<String, Error>;  // Not every repo needs this
}

// GOOD — segregated interfaces, compose with trait bounds
trait Find<T> { async fn find(&self, id: Id) -> Result<Option<T>, RepoError>; }
trait Save<T> { async fn save(&self, entity: &T) -> Result<(), RepoError>; }
trait Delete { async fn delete(&self, id: Id) -> Result<(), RepoError>; }
trait Search<T> { async fn search(&self, query: &str) -> Result<Vec<T>, RepoError>; }

// Use case only requires what it needs
async fn get_order(repo: &(impl Find<Order> + Send + Sync), id: OrderId) -> Result<Order, Error> {
    repo.find(id.into()).await?.ok_or(Error::NotFound)
}
```

**D — Dependency Inversion Principle (DIP)**
High-level modules define abstractions (traits); low-level modules implement them. This is the foundation of hexagonal/onion architecture. See [Trait-Based Dependency Inversion](#trait-based-dependency-inversion) below for full coverage.

### Hexagonal Architecture (Ports & Adapters)

The hexagonal pattern separates business logic from external systems through **ports** (trait definitions) and **adapters** (implementations).

```
              Driving Adapters                  Driven Adapters
          (trigger the application)         (called by the application)
                    │                                │
       ┌────────────┤                                ├────────────┐
       │  HTTP API   │    ┌──────────────────┐       │  Postgres  │
       │  (axum)     ├───►│   APPLICATION    │◄──────┤  (sqlx)    │
       │             │    │                  │       │            │
       │  CLI        │    │  ┌────────────┐  │       │  Redis     │
       │  (clap)     ├───►│  │   DOMAIN   │  │◄──────┤  (redis)   │
       │             │    │  │            │  │       │            │
       │  gRPC       │    │  │  Entities  │  │       │  Stripe    │
       │  (tonic)    ├───►│  │  Rules     │  │◄──────┤  (reqwest) │
       │             │    │  │  Traits    │  │       │            │
       │  Tests      │    │  └────────────┘  │       │  Email     │
       │  (mock)     ├───►│                  │◄──────┤  (lettre)  │
       └─────────────┘    └──────────────────┘       └────────────┘
                                  │
                          Ports = Traits
                     (defined in domain layer)
```

In Rust:
- **Ports** = trait definitions in the domain crate (e.g., `trait OrderRepository`, `trait PaymentGateway`)
- **Driving adapters** = HTTP handlers, CLI commands, gRPC services that call use cases
- **Driven adapters** = struct implementations of domain traits (e.g., `SqlxOrderRepo implements OrderRepository`)
- **The hexagon** = domain + application crates that have zero dependency on any adapter

**Key rule:** The domain crate's `Cargo.toml` never lists `sqlx`, `axum`, `redis`, `reqwest`, or any infrastructure crate.

### Onion Architecture

Onion architecture is concentric layers where dependencies only point inward:

```
┌─────────────────────────────────────────────────┐
│  Infrastructure (outermost)                      │
│  - Framework code (axum, actix, tonic)          │
│  - Database implementations (sqlx, diesel)       │
│  - External service clients (reqwest, lettre)    │
│  ┌─────────────────────────────────────────┐     │
│  │  Application Services                    │     │
│  │  - Use cases / commands / queries        │     │
│  │  - Orchestration logic                   │     │
│  │  - Error translation                     │     │
│  │  ┌─────────────────────────────────┐     │     │
│  │  │  Domain Services                 │     │     │
│  │  │  - Cross-entity business rules   │     │     │
│  │  │  - Domain events                 │     │     │
│  │  │  ┌─────────────────────────┐     │     │     │
│  │  │  │  Domain Model (core)    │     │     │     │
│  │  │  │  - Entities             │     │     │     │
│  │  │  │  - Value objects        │     │     │     │
│  │  │  │  - Repository traits    │     │     │     │
│  │  │  │  - Domain errors        │     │     │     │
│  │  │  └─────────────────────────┘     │     │     │
│  │  └─────────────────────────────────┘     │     │
│  └─────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘
```

**Onion vs Hexagonal:** Nearly identical in practice. Onion emphasizes concentric layers and the domain model at the center. Hexagonal emphasizes the symmetry between driving and driven adapters. In Rust, you implement both the same way — traits in domain, implementations in infrastructure.

### Clean Architecture

Clean Architecture (Robert C. Martin) adds an explicit **Entities → Use Cases → Interface Adapters → Frameworks** layering. The key additions over hexagonal/onion:

1. **Use cases are explicit objects** — each with an `execute()` method, not just service methods
2. **Interface adapters** (presenters, controllers, gateways) translate between use case DTOs and external formats
3. **The Dependency Rule** — source code dependencies only point inward

In Rust, map Clean Architecture to:

| Clean Architecture | Rust Implementation |
|-------------------|---------------------|
| **Entities** | Domain structs, enums, value objects, domain traits |
| **Use Cases** | `struct PlaceOrderUseCase<R: OrderRepo>` with `async fn execute(&self, input: Input) -> Result<Output>` |
| **Interface Adapters** | HTTP handlers (axum extractors → use case input), presenters (domain → response DTO) |
| **Frameworks & Drivers** | axum/actix/tonic setup, sqlx/diesel pools, config loading |

```rust
// Entity (innermost) — pure domain logic
pub struct Order { /* ... */ }
impl Order {
    pub fn place(items: Vec<LineItem>) -> Result<Self, DomainError> { /* ... */ }
    pub fn cancel(&mut self) -> Result<(), DomainError> { /* ... */ }
}

// Use Case — orchestrates domain + ports
pub struct PlaceOrderUseCase<R: OrderRepository, P: PaymentGateway> {
    orders: R,
    payments: P,
}
impl<R: OrderRepository, P: PaymentGateway> PlaceOrderUseCase<R, P> {
    pub async fn execute(&self, input: PlaceOrderInput) -> Result<PlaceOrderOutput, AppError> {
        let order = Order::place(input.items)?;          // Domain logic
        self.payments.charge(order.total()).await?;       // Driven port
        self.orders.save(&order).await?;                  // Driven port
        Ok(PlaceOrderOutput::from(order))                 // DTO for callers
    }
}

// Interface Adapter — translates HTTP ↔ use case
async fn place_order_handler(
    State(uc): State<Arc<PlaceOrderUseCase<SqlxOrderRepo, StripePayments>>>,
    Json(body): Json<PlaceOrderRequest>,
) -> Result<Json<PlaceOrderResponse>, ApiError> {
    let input = PlaceOrderInput::try_from(body)?;        // HTTP → use case
    let output = uc.execute(input).await?;                // Use case
    Ok(Json(PlaceOrderResponse::from(output)))            // Use case → HTTP
}
```

### How These Patterns Relate

All three patterns (hexagonal, onion, clean) solve the same problem — **isolating domain logic from infrastructure** — with slightly different vocabulary:

| Concept | Hexagonal | Onion | Clean | Rust |
|---------|-----------|-------|-------|------|
| Core business logic | Hexagon interior | Domain Model | Entities | `domain/` crate |
| Abstractions for external systems | Ports | Domain interfaces | Use Case boundaries | `trait` definitions |
| Concrete implementations | Adapters | Infrastructure | Frameworks & Drivers | `impl Trait for ConcreteType` |
| Orchestration | Application services | Application services | Use Cases | `struct XxxUseCase<R: Repo>` |
| Direction of dependencies | Inward only | Inward only | Inward only (Dependency Rule) | Cargo.toml deps point inward |

**In practice for Rust projects**, use this simplified structure:

```
my-app/
├── Cargo.toml           (workspace)
├── domain/              (entities, value objects, traits — zero infra deps)
│   ├── Cargo.toml       (only: uuid, chrono, thiserror, serde)
│   └── src/
├── application/         (use cases — depends only on domain)
│   ├── Cargo.toml       (only: domain, async-trait, tracing)
│   └── src/
├── infrastructure/      (adapters — implements domain traits)
│   ├── Cargo.toml       (domain, application, sqlx, redis, reqwest, lettre)
│   └── src/
└── server/              (composition root — wires everything together)
    ├── Cargo.toml       (all crates, axum/actix, config)
    └── src/main.rs
```

**The litmus test:** Can you delete the `infrastructure/` and `server/` crates and still compile `domain/` and `application/`? If yes, your architecture is clean.

### Domain-Driven Design (DDD) in Rust

DDD maps well to Rust's type system — many DDD concepts that require discipline in other languages are enforced at compile time.

**Strategic DDD — Bounded Contexts as Cargo Workspaces:**
Each bounded context is a separate crate (or workspace member). Contexts communicate through well-defined interfaces, never share internal types directly.

```
workspace/
├── ordering/        ← Ordering bounded context
│   └── src/         (Order, LineItem, OrderRepository)
├── inventory/       ← Inventory bounded context
│   └── src/         (StockItem, Warehouse, StockRepository)
├── shipping/        ← Shipping bounded context
│   └── src/         (Shipment, Carrier, TrackingNumber)
└── shared-kernel/   ← Shared types across contexts
    └── src/         (Money, Address, CustomerId — value objects only)
```

**Tactical DDD patterns in Rust:**

| DDD Concept | Rust Implementation |
|-------------|---------------------|
| **Entity** | Struct with identity field (`id: EntityId`), compared by ID not value |
| **Value Object** | `#[derive(Clone, PartialEq, Eq, Hash)]` struct, compared by all fields, immutable |
| **Aggregate** | Entity that owns related entities, enforces invariants, is the transaction boundary |
| **Aggregate Root** | The only entity in an aggregate exposed to the outside — other entities are `pub(crate)` |
| **Repository** | `trait XxxRepository` in domain, implemented in infrastructure |
| **Domain Event** | Enum with one variant per event, `#[derive(Clone, Serialize, Deserialize)]` |
| **Domain Service** | Free function or struct with no state, operates across multiple aggregates |
| **Factory** | `impl Aggregate { fn create(...) -> Result<Self, DomainError> }` — validated construction |
| **Specification** | `trait Specification<T> { fn is_satisfied_by(&self, t: &T) -> bool; }` |
| **Anti-Corruption Layer** | Adapter that translates between your domain and an external system's types |

**Key Rust advantages for DDD:**
- **Newtypes enforce identity:** `struct OrderId(Uuid)` — can't accidentally pass a `CustomerId` where `OrderId` is expected
- **Enums model state machines:** Invalid state transitions are compile errors, not runtime exceptions
- **Ownership enforces aggregate boundaries:** Moving an entity into an aggregate transfers ownership — no accidental sharing
- **`pub(crate)` enforces aggregate roots:** Inner entities are invisible outside the crate

```rust
// Value object — compared by value, immutable
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct EmailAddress(String);
impl EmailAddress {
    pub fn new(email: &str) -> Result<Self, ValidationError> {
        if !email.contains('@') { return Err(ValidationError::InvalidEmail); }
        Ok(Self(email.to_lowercase()))
    }
}

// Entity — compared by identity
pub struct Customer {
    id: CustomerId,           // Identity
    name: String,
    email: EmailAddress,      // Value object
}
impl PartialEq for Customer {
    fn eq(&self, other: &Self) -> bool { self.id == other.id } // Identity equality
}

// Aggregate root — owns its children, enforces invariants
pub struct Order {
    id: OrderId,
    customer_id: CustomerId,
    items: Vec<LineItem>,     // Owned child entities
    status: OrderStatus,
}
impl Order {
    // Factory method — validates invariants at creation
    pub fn create(customer_id: CustomerId, items: Vec<LineItem>) -> Result<Self, DomainError> {
        if items.is_empty() { return Err(DomainError::EmptyOrder); }
        Ok(Self { id: OrderId::new(), customer_id, items, status: OrderStatus::Draft })
    }

    // All mutations go through the aggregate root
    pub fn add_item(&mut self, item: LineItem) -> Result<(), DomainError> {
        if self.status != OrderStatus::Draft {
            return Err(DomainError::OrderNotModifiable);
        }
        self.items.push(item);
        Ok(())
    }
}
```

See [domain-patterns.md](domain-patterns.md) for complete DDD implementation including bounded context communication, anti-corruption layers, domain events, event sourcing, CQRS, and testing strategies.

### Event Sourcing & Architecture

Event sourcing fits naturally into hexagonal/clean architecture:
- **Domain layer** defines events, aggregates, and the `apply(event)` state-rebuild logic
- **Application layer** defines commands that produce events
- **Infrastructure layer** implements the event store (Postgres, EventStoreDB)

The domain aggregate never knows how events are persisted — it only knows how to apply them:

```rust
// Domain: aggregate + events (no infrastructure dependency)
enum OrderEvent {
    Placed { id: OrderId, items: Vec<LineItem> },
    Shipped { tracking: String },
    Cancelled { reason: String },
}

struct Order { /* state rebuilt from events */ }
impl Order {
    fn apply(&mut self, event: &OrderEvent) { /* pure state transition */ }
    fn place(items: Vec<LineItem>) -> (Self, OrderEvent) { /* returns new event */ }
}

// Port: event store trait (defined in domain)
trait EventStore {
    async fn append(&self, stream: &str, events: Vec<DomainEvent>) -> Result<(), StoreError>;
    async fn load(&self, stream: &str) -> Result<Vec<DomainEvent>, StoreError>;
}
```

See [domain-patterns.md](domain-patterns.md#event-sourcing) for complete event sourcing implementation including snapshots, projections, event versioning, and testing.

## Cargo Workspace Organization

### Workspace Layout

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "2"
members = [
    "crates/domain",
    "crates/app",
    "crates/infra",
    "crates/api",
    "crates/cli",
]

# Shared dependencies — members inherit these
[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2"
anyhow = "1"
tracing = "0.1"
axum = "0.8"
```

### Member Crate Cargo.toml

```toml
# crates/domain/Cargo.toml
[package]
name = "domain"
version = "0.1.0"
edition = "2024"

[dependencies]
# Inherit from workspace — no version duplication
serde = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
# Domain crate has ZERO framework dependencies
```

```toml
# crates/infra/Cargo.toml
[package]
name = "infra"
version = "0.1.0"
edition = "2024"

[dependencies]
domain = { path = "../domain" }
app = { path = "../app" }
sqlx = { workspace = true }
tokio = { workspace = true }
serde = { workspace = true }
tracing = { workspace = true }
```

### When to Split Crates

| Signal | Action |
|--------|--------|
| Module has zero deps on parent crate | Extract to own crate |
| Two binaries share a library | Shared `lib` crate |
| Compile time > 30s for a crate | Split hot/cold paths |
| Team owns a subsystem | Crate per team boundary |
| Different MSRV or edition needs | Separate crate |

**Don't split prematurely** — a single `lib` crate with well-organized modules is fine for most projects. Workspace overhead (build scripts, CI matrix) compounds.

### Feature Flag Architecture

```toml
# Cargo.toml
[features]
default = ["postgres"]
postgres = ["sqlx/postgres"]
sqlite = ["sqlx/sqlite"]
redis-cache = ["redis"]
metrics = ["prometheus"]

# Optional dependencies gated by features
[dependencies]
redis = { version = "0.27", optional = true }
prometheus = { version = "0.13", optional = true }
```

```rust
// Conditional compilation with features
pub struct AppState {
    pub db: DatabasePool,
    #[cfg(feature = "redis-cache")]
    pub cache: redis::Client,
}

#[cfg(feature = "redis-cache")]
pub async fn get_cached(cache: &redis::Client, key: &str) -> Option<String> {
    let mut conn = cache.get_multiplexed_async_connection().await.ok()?;
    redis::cmd("GET").arg(key).query_async(&mut conn).await.ok()
}

#[cfg(not(feature = "redis-cache"))]
pub async fn get_cached(_key: &str) -> Option<String> {
    None // No-op when Redis feature disabled
}
```

### Feature-Gated Server Roles

For systems that deploy as a single binary but run different roles (gateway, media worker, console, connector), use features to compile only the code needed for each deployment mode. Pattern from atm0s-media-server:

```toml
# Cargo.toml
[features]
default = ["media"]
gateway = ["dep:poem", "dep:media-server-gateway"]
media = ["dep:str0m", "dep:transport-webrtc"]
console = ["dep:media-server-console-front", "dep:poem"]
connector = ["dep:media-server-connector"]
full = ["gateway", "media", "console", "connector"]
```

```rust
// bin/src/http.rs — each server role is a separate function
#[cfg(feature = "console")]
pub async fn run_console_server(ctx: Arc<ConsoleCtx>, port: u16) -> Result<()> {
    let app = Route::new()
        .nest("/api/", console_api(ctx.clone()))
        .nest("/ws", console_websocket(ctx));
    poem::Server::new(TcpListener::bind(("0.0.0.0", port)))
        .run(app).await?;
    Ok(())
}

#[cfg(feature = "gateway")]
pub async fn run_gateway_server(ctx: Arc<GatewayCtx>, port: u16) -> Result<()> {
    let app = Route::new()
        .nest("/webrtc/", webrtc_api(ctx.clone()))
        .nest("/whip/", whip_api(ctx.clone()))
        .nest("/whep/", whep_api(ctx));
    poem::Server::new(TcpListener::bind(("0.0.0.0", port)))
        .run(app).await?;
    Ok(())
}

// main.rs — select role at startup
fn main() {
    let role = std::env::var("SERVER_ROLE").unwrap_or("media".into());
    match role.as_str() {
        #[cfg(feature = "gateway")]
        "gateway" => run_gateway_server(ctx, port),
        #[cfg(feature = "media")]
        "media" => run_media_server(ctx, port),
        #[cfg(feature = "full")]
        "full" => { /* start all roles */ }
        _ => panic!("unknown role: {role}"),
    }
}
```

**Benefits:** Single binary artifact for CI/CD, dead-code elimination per role, smaller Docker images when deploying specialized nodes, compile-time guarantee that gateway code never runs on media workers.

### Visibility Boundaries

```rust
// Use pub(crate) to keep internals private to the crate
pub struct UserService {
    repo: Box<dyn UserRepository>,
}

impl UserService {
    pub fn new(repo: Box<dyn UserRepository>) -> Self {
        Self { repo }
    }

    // Public API
    pub async fn get_user(&self, id: UserId) -> Result<User, ServiceError> {
        self.validate_id(&id)?;
        self.repo.find_by_id(id).await?.ok_or(ServiceError::NotFound)
    }

    // Internal helper — not visible outside this crate
    pub(crate) fn validate_id(&self, id: &UserId) -> Result<(), ServiceError> {
        if id.0.is_nil() {
            return Err(ServiceError::InvalidId);
        }
        Ok(())
    }
}
```

## Trait-Based Dependency Inversion

### Core Principle

Dependencies point inward. Domain defines traits (ports), infrastructure implements them (adapters).

```
┌────────────────────────────────────────────────┐
│              Infrastructure Layer               │
│   (Postgres, Redis, HTTP clients, queues)       │
│  ┌────────────────────────────────────────┐     │
│  │          Application Layer              │     │
│  │       (Use cases, orchestration)        │     │
│  │  ┌────────────────────────────────┐     │     │
│  │  │        Domain Layer            │     │     │
│  │  │  (Entities, value objects,     │     │     │
│  │  │   traits, business rules)      │     │     │
│  │  └────────────────────────────────┘     │     │
│  └────────────────────────────────────────┘     │
└────────────────────────────────────────────────┘
         Dependencies flow INWARD →
```

### Repository Pattern

```rust
// domain/repository.rs — trait definition (port)
// IMPORTANT: Only trait definitions here — NO implementations

pub trait OrderRepository: Send + Sync {
    async fn find_by_id(&self, id: OrderId) -> Result<Option<Order>, RepoError>;
    async fn save(&self, order: &Order) -> Result<(), RepoError>;
    async fn find_by_customer(&self, id: CustomerId) -> Result<Vec<Order>, RepoError>;
    async fn delete(&self, id: OrderId) -> Result<(), RepoError>;
}

#[derive(Debug, thiserror::Error)]
pub enum RepoError {
    #[error("entity not found")]
    NotFound,
    #[error("database error: {0}")]
    Database(String),
}
```

```rust
// infra/postgres_repo.rs — concrete implementation (adapter)
use sqlx::PgPool;

pub struct PgOrderRepository {
    pool: PgPool,
}

impl PgOrderRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

impl OrderRepository for PgOrderRepository {
    async fn find_by_id(&self, id: OrderId) -> Result<Option<Order>, RepoError> {
        sqlx::query_as!(
            OrderRecord,
            "SELECT id, customer_id, status, created_at FROM orders WHERE id = $1",
            id.0
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| RepoError::Database(e.to_string()))?
        .map(|r| r.into_domain())
        .transpose()
    }

    async fn save(&self, order: &Order) -> Result<(), RepoError> {
        let mut tx = self.pool.begin().await
            .map_err(|e| RepoError::Database(e.to_string()))?;

        sqlx::query!(
            r#"INSERT INTO orders (id, customer_id, status, created_at)
               VALUES ($1, $2, $3, $4)
               ON CONFLICT (id) DO UPDATE SET status = EXCLUDED.status"#,
            order.id().0,
            order.customer_id().0,
            serde_json::to_string(order.status()).unwrap(),
            order.created_at,
        )
        .execute(&mut *tx)
        .await
        .map_err(|e| RepoError::Database(e.to_string()))?;

        tx.commit().await
            .map_err(|e| RepoError::Database(e.to_string()))?;

        Ok(())
    }

    async fn find_by_customer(&self, id: CustomerId) -> Result<Vec<Order>, RepoError> {
        let records = sqlx::query_as!(
            OrderRecord,
            "SELECT id, customer_id, status, created_at FROM orders WHERE customer_id = $1",
            id.0
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| RepoError::Database(e.to_string()))?;

        Ok(records.into_iter().map(|r| r.into_domain()).collect())
    }

    async fn delete(&self, id: OrderId) -> Result<(), RepoError> {
        sqlx::query!("DELETE FROM orders WHERE id = $1", id.0)
            .execute(&self.pool)
            .await
            .map_err(|e| RepoError::Database(e.to_string()))?;
        Ok(())
    }
}
```

### Generics vs Trait Objects

| Aspect | Generics `<T: Trait>` | Trait Objects `Box<dyn Trait>` |
|--------|----------------------|-------------------------------|
| Dispatch | Static (compile-time) | Dynamic (runtime) |
| Performance | Zero-cost, inlined | vtable lookup |
| Binary size | Larger (monomorphization) | Smaller |
| Flexibility | Type fixed at compile-time | Type chosen at runtime |
| Object safety | All traits | Only object-safe traits |

```rust
// Prefer generics for performance-critical, known-at-compile-time deps
pub struct UserService<R: UserRepository> {
    repo: R,
}

impl<R: UserRepository> UserService<R> {
    pub fn new(repo: R) -> Self {
        Self { repo }
    }
}

// Use trait objects when type varies at runtime
fn create_notifier(config: &Config) -> Box<dyn Notifier> {
    match config.notification_type {
        NotificationType::Email => Box::new(EmailNotifier::new(&config.smtp_url)),
        NotificationType::Slack => Box::new(SlackNotifier::new(&config.slack_webhook)),
    }
}
```

### Constructor Injection

```rust
// Generic constructor injection (static dispatch)
pub struct OrderService<R: OrderRepository, P: PaymentGateway, E: EventPublisher> {
    order_repo: R,
    payment: P,
    events: E,
}

impl<R: OrderRepository, P: PaymentGateway, E: EventPublisher> OrderService<R, P, E> {
    pub fn new(order_repo: R, payment: P, events: E) -> Self {
        Self { order_repo, payment, events }
    }

    pub async fn place_order(&self, input: PlaceOrderInput) -> Result<OrderId, ServiceError> {
        let order = Order::new(input.customer_id, input.items)?;
        self.payment.charge(order.customer_id(), order.total()).await?;
        self.order_repo.save(&order).await?;
        self.events.publish(OrderPlaced { order_id: order.id() }).await?;
        Ok(order.id())
    }
}
```

```rust
// Dynamic dispatch injection (runtime flexibility)
pub struct OrderServiceDyn {
    order_repo: Arc<dyn OrderRepository>,
    payment: Arc<dyn PaymentGateway>,
    events: Arc<dyn EventPublisher>,
}

impl OrderServiceDyn {
    pub fn new(
        order_repo: Arc<dyn OrderRepository>,
        payment: Arc<dyn PaymentGateway>,
        events: Arc<dyn EventPublisher>,
    ) -> Self {
        Self { order_repo, payment, events }
    }
}
```

### Method Injection

For dependencies needed only in specific operations:

```rust
pub struct OrderProcessor {
    repo: Box<dyn OrderRepository>,
}

impl OrderProcessor {
    pub fn new(repo: Box<dyn OrderRepository>) -> Self {
        Self { repo }
    }

    // Notifier injected per-call — not owned, just borrowed
    pub async fn process_with_notification(
        &self,
        order_id: OrderId,
        notifier: &dyn Notifier,
    ) -> Result<(), ProcessError> {
        let order = self.repo.find_by_id(order_id).await?
            .ok_or(ProcessError::NotFound)?;
        // Process...
        notifier.send(&format!("Order {} processed", order_id.0)).await?;
        Ok(())
    }
}
```

### Extension Traits

Add behavior to existing types without modifying them:

```rust
// Define an extension trait for Vec<Order>
pub trait OrderCollectionExt {
    fn total_revenue(&self) -> Money;
    fn by_status(&self, status: &OrderStatus) -> Vec<&Order>;
}

impl OrderCollectionExt for Vec<Order> {
    fn total_revenue(&self) -> Money {
        self.iter()
            .map(|o| o.total())
            .fold(Money::zero(), |acc, m| acc.add(&m).unwrap_or(acc))
    }

    fn by_status(&self, status: &OrderStatus) -> Vec<&Order> {
        self.iter()
            .filter(|o| std::mem::discriminant(o.status()) == std::mem::discriminant(status))
            .collect()
    }
}
```

### Sealed Traits

Prevent external crates from implementing your trait:

```rust
mod private {
    pub trait Sealed {}
}

/// Storage backend — only implementations in this crate are allowed.
pub trait StorageBackend: private::Sealed {
    async fn store(&self, key: &str, data: &[u8]) -> Result<(), StorageError>;
    async fn load(&self, key: &str) -> Result<Vec<u8>, StorageError>;
}

// Only these implementations exist
pub struct LocalFs;
impl private::Sealed for LocalFs {}
impl StorageBackend for LocalFs { /* ... */ }

pub struct S3;
impl private::Sealed for S3 {}
impl StorageBackend for S3 { /* ... */ }
```

## Application Layering

### Domain Layer (Innermost)

Pure business logic. **Zero framework dependencies.** Only std + domain-relevant crates (uuid, chrono, rust_decimal).

```rust
// domain/value_object.rs — immutable, compared by value
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Money {
    amount: i64,        // Store as cents to avoid floating point
    currency: Currency,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Currency { USD, EUR, GBP }

impl Money {
    pub fn from_cents(cents: i64, currency: Currency) -> Self {
        Self { amount: cents, currency }
    }

    pub fn add(&self, other: &Money) -> Result<Money, DomainError> {
        if self.currency != other.currency {
            return Err(DomainError::CurrencyMismatch);
        }
        Ok(Money { amount: self.amount + other.amount, currency: self.currency })
    }

    pub fn cents(&self) -> i64 { self.amount }
    pub fn currency(&self) -> Currency { self.currency }
}
```

```rust
// domain/entity.rs — has identity, mutable, compared by ID
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(pub Uuid);

#[derive(Debug, Clone)]
pub struct Order {
    id: OrderId,
    customer_id: CustomerId,
    items: Vec<OrderItem>,
    status: OrderStatus,
    created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum OrderStatus {
    Pending,
    Confirmed { confirmed_at: DateTime<Utc> },
    Shipped { tracking: String, shipped_at: DateTime<Utc> },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String, cancelled_at: DateTime<Utc> },
}

impl Order {
    // Factory method with validation
    pub fn new(customer_id: CustomerId, items: Vec<OrderItem>) -> Result<Self, DomainError> {
        if items.is_empty() {
            return Err(DomainError::EmptyOrder);
        }
        Ok(Self {
            id: OrderId(Uuid::new_v4()),
            customer_id,
            items,
            status: OrderStatus::Pending,
            created_at: Utc::now(),
        })
    }

    // State transition with validation
    pub fn confirm(&mut self) -> Result<(), DomainError> {
        match &self.status {
            OrderStatus::Pending => {
                self.status = OrderStatus::Confirmed { confirmed_at: Utc::now() };
                Ok(())
            }
            _ => Err(DomainError::InvalidTransition {
                from: format!("{:?}", self.status),
                to: "Confirmed".into(),
            }),
        }
    }

    pub fn ship(&mut self, tracking: String) -> Result<(), DomainError> {
        match &self.status {
            OrderStatus::Confirmed { .. } => {
                self.status = OrderStatus::Shipped { tracking, shipped_at: Utc::now() };
                Ok(())
            }
            _ => Err(DomainError::InvalidTransition {
                from: format!("{:?}", self.status),
                to: "Shipped".into(),
            }),
        }
    }

    pub fn cancel(&mut self, reason: String) -> Result<(), DomainError> {
        match &self.status {
            OrderStatus::Pending | OrderStatus::Confirmed { .. } => {
                self.status = OrderStatus::Cancelled { reason, cancelled_at: Utc::now() };
                Ok(())
            }
            _ => Err(DomainError::CannotCancelAfterShipment),
        }
    }

    pub fn total(&self) -> Money {
        self.items.iter().fold(Money::from_cents(0, Currency::USD), |acc, item| {
            let line = Money::from_cents(
                item.unit_price.cents() * item.quantity as i64,
                item.unit_price.currency(),
            );
            acc.add(&line).unwrap_or(acc)
        })
    }

    // Getters preserve encapsulation
    pub fn id(&self) -> OrderId { self.id }
    pub fn customer_id(&self) -> CustomerId { self.customer_id }
    pub fn status(&self) -> &OrderStatus { &self.status }
    pub fn items(&self) -> &[OrderItem] { &self.items }
}
```

```rust
// domain/error.rs
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("order cannot be empty")]
    EmptyOrder,
    #[error("currency mismatch")]
    CurrencyMismatch,
    #[error("invalid transition from {from} to {to}")]
    InvalidTransition { from: String, to: String },
    #[error("cannot cancel after shipment")]
    CannotCancelAfterShipment,
}
```

### Application Layer (Use Cases)

Orchestrates domain objects. Depends on domain traits, not infrastructure.

```rust
// app/dto.rs — Data Transfer Objects for input/output
#[derive(Debug, Clone, Deserialize)]
pub struct PlaceOrderInput {
    pub customer_id: Uuid,
    pub items: Vec<OrderItemInput>,
}

#[derive(Debug, Clone, Deserialize)]
pub struct OrderItemInput {
    pub product_id: Uuid,
    pub quantity: u32,
}

#[derive(Debug, Clone, Serialize)]
pub struct OrderOutput {
    pub order_id: Uuid,
    pub status: String,
    pub total_cents: i64,
    pub created_at: DateTime<Utc>,
}

impl From<&Order> for OrderOutput {
    fn from(order: &Order) -> Self {
        Self {
            order_id: order.id().0,
            status: format!("{:?}", order.status()),
            total_cents: order.total().cents(),
            created_at: order.created_at,
        }
    }
}
```

```rust
// app/use_case/place_order.rs
pub struct PlaceOrderUseCase<O, C, P>
where
    O: OrderRepository,
    C: CustomerRepository,
    P: PaymentGateway,
{
    orders: O,
    customers: C,
    payments: P,
}

impl<O, C, P> PlaceOrderUseCase<O, C, P>
where
    O: OrderRepository,
    C: CustomerRepository,
    P: PaymentGateway,
{
    pub fn new(orders: O, customers: C, payments: P) -> Self {
        Self { orders, customers, payments }
    }

    pub async fn execute(&self, input: PlaceOrderInput) -> Result<OrderOutput, UseCaseError> {
        // 1. Validate customer exists
        let customer_id = CustomerId(input.customer_id);
        let _customer = self.customers.find_by_id(customer_id).await?
            .ok_or(UseCaseError::CustomerNotFound)?;

        // 2. Create domain entity (business validation happens here)
        let items = input.items.into_iter().map(|i| OrderItem {
            product_id: ProductId(i.product_id),
            quantity: i.quantity,
            unit_price: Money::from_cents(0, Currency::USD), // fetch real price
        }).collect();

        let order = Order::new(customer_id, items)
            .map_err(UseCaseError::Domain)?;

        // 3. Charge payment
        self.payments.charge(customer_id, order.total()).await
            .map_err(UseCaseError::Payment)?;

        // 4. Persist
        self.orders.save(&order).await?;

        Ok(OrderOutput::from(&order))
    }
}

#[derive(Debug, thiserror::Error)]
pub enum UseCaseError {
    #[error("customer not found")]
    CustomerNotFound,
    #[error("domain: {0}")]
    Domain(#[from] DomainError),
    #[error("repo: {0}")]
    Repo(#[from] RepoError),
    #[error("payment: {0}")]
    Payment(PaymentError),
}
```

### Infrastructure Layer (Outermost)

Concrete implementations of domain traits. Depends on domain + application layers.

```rust
// infra/external/stripe_gateway.rs
pub struct StripeGateway {
    client: reqwest::Client,
    api_key: String,
}

impl StripeGateway {
    pub fn new(api_key: String) -> Self {
        Self {
            client: reqwest::Client::new(),
            api_key,
        }
    }
}

impl PaymentGateway for StripeGateway {
    async fn charge(&self, customer_id: CustomerId, amount: Money) -> Result<PaymentId, PaymentError> {
        let resp = self.client
            .post("https://api.stripe.com/v1/charges")
            .header("Authorization", format!("Bearer {}", self.api_key))
            .form(&[
                ("amount", amount.cents().to_string()),
                ("currency", format!("{:?}", amount.currency()).to_lowercase()),
            ])
            .send()
            .await
            .map_err(|e| PaymentError::Network(e.to_string()))?;

        if !resp.status().is_success() {
            return Err(PaymentError::Declined);
        }

        let body: serde_json::Value = resp.json().await
            .map_err(|e| PaymentError::Parse(e.to_string()))?;

        Ok(PaymentId(body["id"].as_str().unwrap_or_default().to_string()))
    }
}
```

### HTTP Handler (Driving Adapter)

```rust
// infra/web/handlers.rs
use axum::{extract::State, Json};

type AppOrderService = OrderServiceDyn; // or generic version

async fn place_order(
    State(svc): State<Arc<AppOrderService>>,
    Json(input): Json<PlaceOrderInput>,
) -> Result<(StatusCode, Json<OrderOutput>), ApiError> {
    let output = svc.place_order(input).await?;
    Ok((StatusCode::CREATED, Json(output)))
}

impl From<UseCaseError> for ApiError {
    fn from(e: UseCaseError) -> Self {
        match e {
            UseCaseError::CustomerNotFound => ApiError::NotFound("customer"),
            UseCaseError::Domain(d) => ApiError::BadRequest(d.to_string()),
            UseCaseError::Payment(_) => ApiError::PaymentRequired,
            UseCaseError::Repo(_) => ApiError::Internal,
        }
    }
}
```

### Project Structure

```
my-service/
├── Cargo.toml              # Workspace root
├── crates/
│   ├── domain/
│   │   ├── Cargo.toml       # Zero framework deps
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── entity.rs    # Order, Customer
│   │       ├── value.rs     # Money, Address
│   │       ├── repo.rs      # Repository trait definitions
│   │       ├── service.rs   # Domain services
│   │       └── error.rs
│   ├── app/
│   │   ├── Cargo.toml       # Depends on domain
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── dto.rs
│   │       └── use_case/
│   │           ├── mod.rs
│   │           ├── place_order.rs
│   │           └── cancel_order.rs
│   ├── infra/
│   │   ├── Cargo.toml       # Depends on domain + app
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── postgres/
│   │       │   ├── mod.rs
│   │       │   └── order_repo.rs
│   │       └── external/
│   │           ├── mod.rs
│   │           └── stripe.rs
│   └── api/
│       ├── Cargo.toml       # Depends on all crates + axum
│       └── src/
│           ├── main.rs      # Composition root
│           ├── handlers.rs
│           └── routes.rs
```

### Composition Root

Wire everything at startup:

```rust
// api/src/main.rs
use std::sync::Arc;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize tracing
    tracing_subscriber::fmt()
        .with_env_filter("info,sqlx=warn")
        .init();

    // Load configuration
    let config = Config::from_env()?;

    // Infrastructure
    let db = sqlx::PgPool::connect(&config.database_url).await?;
    sqlx::migrate!().run(&db).await?;

    // Repositories (driven adapters)
    let order_repo: Arc<dyn OrderRepository> =
        Arc::new(PgOrderRepository::new(db.clone()));
    let customer_repo: Arc<dyn CustomerRepository> =
        Arc::new(PgCustomerRepository::new(db.clone()));

    // External services
    let payment: Arc<dyn PaymentGateway> =
        Arc::new(StripeGateway::new(config.stripe_key.clone()));

    // Application services
    let order_svc = Arc::new(OrderServiceDyn::new(
        order_repo,
        customer_repo,
        payment,
    ));

    // Build router
    let app = axum::Router::new()
        .route("/orders", axum::routing::post(handlers::place_order))
        .route("/orders/{id}", axum::routing::get(handlers::get_order))
        .route("/health", axum::routing::get(|| async { "ok" }))
        .with_state(order_svc);

    // Start server
    let listener = tokio::net::TcpListener::bind(&config.listen_addr).await?;
    tracing::info!("listening on {}", config.listen_addr);
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    Ok(())
}
```

## State Management & Configuration

### Configuration Loading

```rust
use serde::Deserialize;

#[derive(Debug, Clone, Deserialize)]
pub struct Config {
    #[serde(default = "default_listen")]
    pub listen_addr: String,
    pub database_url: String,
    pub stripe_key: String,
    #[serde(default = "default_pool_size")]
    pub db_pool_size: u32,
    #[serde(default)]
    pub log_level: LogLevel,
}

fn default_listen() -> String { "0.0.0.0:8080".into() }
fn default_pool_size() -> u32 { 10 }

#[derive(Debug, Clone, Default, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum LogLevel {
    #[default]
    Info,
    Debug,
    Warn,
}

impl Config {
    /// Load from environment variables (12-factor app style)
    pub fn from_env() -> Result<Self, envy::Error> {
        envy::from_env()
    }
}
```

Using `figment` for layered configuration:

```rust
use figment::{Figment, providers::{Env, Format, Toml, Serialized}};

#[derive(Debug, Deserialize, Serialize)]
pub struct Config {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub redis: Option<RedisConfig>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct DatabaseConfig {
    pub url: String,
    pub max_connections: u32,
}

impl Config {
    pub fn load() -> Result<Self, figment::Error> {
        Figment::new()
            // Lowest priority: defaults
            .merge(Serialized::defaults(Config::default()))
            // Then config file
            .merge(Toml::file("config.toml"))
            // Highest priority: environment variables (APP_ prefix)
            .merge(Env::prefixed("APP_").split("_"))
            .extract()
    }
}

impl Default for Config {
    fn default() -> Self {
        Self {
            server: ServerConfig { host: "0.0.0.0".into(), port: 8080 },
            database: DatabaseConfig { url: String::new(), max_connections: 10 },
            redis: None,
        }
    }
}
```

### Global State with LazyLock

```rust
use std::sync::LazyLock;

// Compile-time known values
static APP_VERSION: &str = env!("CARGO_PKG_VERSION");

// Runtime-initialized globals (use sparingly — prefer DI)
static CONFIG: LazyLock<Config> = LazyLock::new(|| {
    Config::from_env().expect("failed to load configuration")
});

// Thread-safe metrics counter
static REQUEST_COUNT: LazyLock<AtomicU64> = LazyLock::new(|| AtomicU64::new(0));

// When to use LazyLock vs dependency injection:
// - LazyLock: truly global, read-only after init (config, metrics, version)
// - DI: testable, swappable, per-request (repos, services, caches)
```

### Connection Pool Patterns

```rust
use sqlx::postgres::PgPoolOptions;

pub async fn create_pool(config: &DatabaseConfig) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(config.max_connections)
        .min_connections(2)
        .acquire_timeout(std::time::Duration::from_secs(5))
        .idle_timeout(std::time::Duration::from_secs(600))
        .test_before_acquire(true)
        .connect(&config.url)
        .await
}

// Share pool via Arc (or axum State)
pub struct AppState {
    pub db: PgPool,
    pub http: reqwest::Client,
}

impl AppState {
    pub async fn new(config: &Config) -> Result<Self, anyhow::Error> {
        let db = create_pool(&config.database).await?;
        let http = reqwest::Client::builder()
            .timeout(std::time::Duration::from_secs(10))
            .pool_max_idle_per_host(20)
            .build()?;
        Ok(Self { db, http })
    }
}
```

## Production Patterns

### Graceful Shutdown

```rust
use tokio::signal;
use tokio_util::sync::CancellationToken;

pub async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => tracing::info!("received Ctrl+C"),
        _ = terminate => tracing::info!("received SIGTERM"),
    }
}

// CancellationToken for coordinated shutdown of background tasks
pub async fn run_with_shutdown(token: CancellationToken) {
    // Spawn background workers
    let worker_token = token.clone();
    let worker = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = worker_token.cancelled() => {
                    tracing::info!("worker shutting down");
                    break;
                }
                _ = do_work() => {}
            }
        }
    });

    // Wait for shutdown signal
    shutdown_signal().await;
    token.cancel();

    // Wait for workers with timeout
    let _ = tokio::time::timeout(
        std::time::Duration::from_secs(30),
        worker,
    ).await;

    tracing::info!("shutdown complete");
}
```

### Health Checks & Readiness

```rust
use axum::{extract::State, http::StatusCode, Json};
use serde::Serialize;

#[derive(Serialize)]
struct HealthResponse {
    status: &'static str,
    version: &'static str,
    checks: Vec<HealthCheck>,
}

#[derive(Serialize)]
struct HealthCheck {
    name: &'static str,
    status: &'static str,
    #[serde(skip_serializing_if = "Option::is_none")]
    message: Option<String>,
}

// Liveness probe — is the process alive?
async fn health_live() -> &'static str {
    "ok"
}

// Readiness probe — can the service handle requests?
async fn health_ready(
    State(state): State<Arc<AppState>>,
) -> Result<Json<HealthResponse>, StatusCode> {
    let mut checks = Vec::new();

    // Check database
    let db_check = match sqlx::query("SELECT 1")
        .execute(&state.db)
        .await
    {
        Ok(_) => HealthCheck { name: "database", status: "up", message: None },
        Err(e) => HealthCheck {
            name: "database",
            status: "down",
            message: Some(e.to_string()),
        },
    };
    checks.push(db_check);

    let all_up = checks.iter().all(|c| c.status == "up");

    let response = HealthResponse {
        status: if all_up { "healthy" } else { "degraded" },
        version: env!("CARGO_PKG_VERSION"),
        checks,
    };

    if all_up {
        Ok(Json(response))
    } else {
        Err(StatusCode::SERVICE_UNAVAILABLE)
    }
}

// Wire into router
fn health_routes() -> axum::Router<Arc<AppState>> {
    axum::Router::new()
        .route("/health/live", axum::routing::get(health_live))
        .route("/health/ready", axum::routing::get(health_ready))
}
```

### Structured Logging with tracing

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

pub fn init_tracing() {
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| "info,sqlx=warn,tower_http=debug".into()))
        .with(tracing_subscriber::fmt::layer()
            .json()                    // JSON format for production
            .with_target(true)
            .with_thread_ids(true)
            .with_span_events(tracing_subscriber::fmt::format::FmtSpan::CLOSE))
        .init();
}

// Request tracing middleware
use tower_http::trace::TraceLayer;

fn app_router() -> axum::Router<Arc<AppState>> {
    axum::Router::new()
        .route("/orders", axum::routing::post(handlers::place_order))
        .layer(TraceLayer::new_for_http())
}

// Structured spans on handlers
#[tracing::instrument(skip(svc), fields(customer_id))]
async fn place_order(
    State(svc): State<Arc<OrderServiceDyn>>,
    Json(input): Json<PlaceOrderInput>,
) -> Result<Json<OrderOutput>, ApiError> {
    tracing::Span::current().record("customer_id", &input.customer_id.to_string());
    let output = svc.place_order(input).await?;
    tracing::info!(order_id = %output.order_id, "order placed");
    Ok(Json(output))
}
```

### Idempotency

Prevent duplicate operations on network retries:

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use uuid::Uuid;

#[derive(Clone)]
pub struct IdempotencyStore {
    cache: Arc<RwLock<HashMap<Uuid, (StatusCode, serde_json::Value)>>>,
}

impl IdempotencyStore {
    pub fn new() -> Self {
        Self { cache: Arc::new(RwLock::new(HashMap::new())) }
    }

    pub fn get(&self, key: &Uuid) -> Option<(StatusCode, serde_json::Value)> {
        self.cache.read().unwrap().get(key).cloned()
    }

    pub fn set(&self, key: Uuid, status: StatusCode, body: serde_json::Value) {
        self.cache.write().unwrap().insert(key, (status, body));
    }
}

// Axum middleware
pub async fn idempotency_middleware(
    State(store): State<IdempotencyStore>,
    headers: axum::http::HeaderMap,
    request: axum::extract::Request,
    next: axum::middleware::Next,
) -> axum::response::Response {
    let key = headers
        .get("Idempotency-Key")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| Uuid::parse_str(s).ok());

    let Some(key) = key else {
        return next.run(request).await;
    };

    // Return cached response if exists
    if let Some((status, body)) = store.get(&key) {
        return (status, Json(body)).into_response();
    }

    // Execute and cache
    let response = next.run(request).await;
    // In production: extract body, cache it
    response
}
```

### Metrics with Prometheus

```rust
use prometheus::{IntCounter, IntGauge, Histogram, register_int_counter, register_int_gauge, register_histogram};
use std::sync::LazyLock;

static HTTP_REQUESTS: LazyLock<IntCounter> = LazyLock::new(|| {
    register_int_counter!("http_requests_total", "Total HTTP requests").unwrap()
});

static ACTIVE_CONNECTIONS: LazyLock<IntGauge> = LazyLock::new(|| {
    register_int_gauge!("active_connections", "Active connections").unwrap()
});

static REQUEST_DURATION: LazyLock<Histogram> = LazyLock::new(|| {
    register_histogram!(
        "http_request_duration_seconds",
        "HTTP request duration",
        vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
    ).unwrap()
});

// Metrics endpoint
async fn metrics_handler() -> String {
    use prometheus::Encoder;
    let encoder = prometheus::TextEncoder::new();
    let metric_families = prometheus::gather();
    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer).unwrap();
    String::from_utf8(buffer).unwrap()
}
```

### Error Translation Across Layers

Each layer has its own error type. Convert at boundaries:

```rust
// Domain error
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("order empty")]
    EmptyOrder,
    #[error("invalid transition")]
    InvalidTransition { from: String, to: String },
}

// Application error — wraps domain + repo errors
#[derive(Debug, thiserror::Error)]
pub enum UseCaseError {
    #[error("{0}")]
    Domain(#[from] DomainError),
    #[error("{0}")]
    Repo(#[from] RepoError),
    #[error("not found")]
    NotFound,
}

// API error — user-facing, hides internals
#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("not found")]
    NotFound(&'static str),
    #[error("bad request: {0}")]
    BadRequest(String),
    #[error("payment required")]
    PaymentRequired,
    #[error("internal error")]
    Internal,
}

impl axum::response::IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let (status, code, msg) = match &self {
            ApiError::NotFound(r) => (StatusCode::NOT_FOUND, "NOT_FOUND", r.to_string()),
            ApiError::BadRequest(m) => (StatusCode::BAD_REQUEST, "BAD_REQUEST", m.clone()),
            ApiError::PaymentRequired => (StatusCode::PAYMENT_REQUIRED, "PAYMENT_REQUIRED", self.to_string()),
            ApiError::Internal => {
                tracing::error!("internal error"); // Log full details server-side
                (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL", "internal error".into())
            }
        };
        (status, Json(serde_json::json!({ "error": { "code": code, "message": msg } }))).into_response()
    }
}

// Convert at boundary
impl From<UseCaseError> for ApiError {
    fn from(e: UseCaseError) -> Self {
        match e {
            UseCaseError::NotFound => ApiError::NotFound("resource"),
            UseCaseError::Domain(d) => ApiError::BadRequest(d.to_string()),
            UseCaseError::Repo(r) => {
                tracing::error!(error = %r, "repository error");
                ApiError::Internal
            }
        }
    }
}
```

**Multi-crate error hierarchy (workspace with 4+ crates):**

In a workspace, each crate defines its own error type. The key question: which crate defines the "master" error? Answer: **there is no master — errors translate at each boundary.**

```
domain::DomainError          (business rules: EmptyOrder, InsufficientFunds)
    ↓ From
app::UseCaseError            (orchestration: NotFound, Domain(..), Repo(..), Forbidden)
    ↓ From
api::ApiError                (HTTP-facing: NotFound, BadRequest, Forbidden, Internal)
    ↓ IntoResponse
axum::Response               (status code + JSON body)
```

Rules for multi-crate error design:
- **Domain errors are exhaustive** — every variant is a business rule violation, callers can match on each
- **Application errors wrap domain + infrastructure** — use `#[from]` for automatic conversion
- **API errors are opaque** — never expose domain or infrastructure details to HTTP clients
- **Infrastructure errors log at conversion** — when `RepoError` becomes `ApiError::Internal`, the `From` impl logs the full details server-side
- **Never propagate `anyhow::Error` across crate boundaries** — it erases type information. Use `thiserror` for library/crate errors, reserve `anyhow` for binary crates and `main()`

```rust
// In domain crate — no #[from], these are pure business errors
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("order must have at least one item")]
    EmptyOrder,
    #[error("insufficient funds: need {required}, have {available}")]
    InsufficientFunds { required: Decimal, available: Decimal },
    #[error("invalid state transition from {from} to {to}")]
    InvalidTransition { from: &'static str, to: &'static str },
}

// In infra crate — wraps underlying driver errors
#[derive(Debug, thiserror::Error)]
pub enum RepoError {
    #[error("database error: {0}")]
    Db(#[from] sqlx::Error),
    #[error("serialization error: {0}")]
    Serde(#[from] serde_json::Error),
    #[error("unique constraint violated: {0}")]
    Conflict(String),
}

// In app crate — composes domain + infra, adds orchestration errors
#[derive(Debug, thiserror::Error)]
pub enum UseCaseError {
    #[error(transparent)]
    Domain(#[from] DomainError),        // domain → app
    #[error(transparent)]
    Repo(#[from] RepoError),            // infra → app
    #[error("resource not found")]
    NotFound,
    #[error("operation forbidden")]
    Forbidden,
}

// In api crate — HTTP-safe, hides internals
#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("not found")] NotFound,
    #[error("bad request: {0}")] BadRequest(String),
    #[error("forbidden")] Forbidden,
    #[error("conflict: {0}")] Conflict(String),
    #[error("internal error")] Internal,
}

impl From<UseCaseError> for ApiError {
    fn from(e: UseCaseError) -> Self {
        match e {
            UseCaseError::NotFound => Self::NotFound,
            UseCaseError::Forbidden => Self::Forbidden,
            UseCaseError::Domain(d) => Self::BadRequest(d.to_string()),
            UseCaseError::Repo(RepoError::Conflict(msg)) => Self::Conflict(msg),
            UseCaseError::Repo(other) => {
                tracing::error!(error = %other, "infrastructure error");
                Self::Internal  // Never leak DB errors to client
            }
        }
    }
}
```

## Testing Architecture

### Domain Layer Tests (No Mocks Needed)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn order_cannot_be_empty() {
        let result = Order::new(CustomerId(Uuid::new_v4()), vec![]);
        assert!(matches!(result, Err(DomainError::EmptyOrder)));
    }

    #[test]
    fn order_state_transitions() {
        let mut order = Order::new(
            CustomerId(Uuid::new_v4()),
            vec![sample_item()],
        ).unwrap();

        assert!(order.confirm().is_ok());
        assert!(order.ship("TRACK123".into()).is_ok());
        assert!(order.cancel("changed mind".into()).is_err()); // Can't cancel shipped
    }

    #[test]
    fn money_rejects_currency_mismatch() {
        let usd = Money::from_cents(100, Currency::USD);
        let eur = Money::from_cents(100, Currency::EUR);
        assert!(usd.add(&eur).is_err());
    }
}
```

### Application Layer Tests (Mocked Ports)

```rust
#[cfg(test)]
mod tests {
    use mockall::mock;

    mock! {
        OrderRepo {}
        impl OrderRepository for OrderRepo {
            async fn find_by_id(&self, id: OrderId) -> Result<Option<Order>, RepoError>;
            async fn save(&self, order: &Order) -> Result<(), RepoError>;
            async fn find_by_customer(&self, id: CustomerId) -> Result<Vec<Order>, RepoError>;
            async fn delete(&self, id: OrderId) -> Result<(), RepoError>;
        }
    }

    #[tokio::test]
    async fn place_order_succeeds() {
        let mut repo = MockOrderRepo::new();
        repo.expect_save().returning(|_| Ok(()));

        let mut customers = MockCustomerRepo::new();
        customers.expect_find_by_id().returning(|_| Ok(Some(sample_customer())));

        let mut payments = MockPaymentGateway::new();
        payments.expect_charge().returning(|_, _| Ok(PaymentId("pay_123".into())));

        let uc = PlaceOrderUseCase::new(repo, customers, payments);
        let result = uc.execute(sample_input()).await;
        assert!(result.is_ok());
    }
}
```

## Growing Architecture — Small to Large

Architecture should match project scale. Don't start with a full workspace when a single crate suffices. Don't stay in a single file when boundaries are needed.

**Quick start:**
- Building a **small CRUD app** (prototype, solo dev, few entities)? → Start at [Stage 1](#stage-1-small-app-1-3-concerns-prototyping-or-solo-dev). Single crate, modules, no traits for internal boundaries.
- Building a **medium app** (team, multiple external deps, test isolation)? → Start at [Stage 2](#stage-2-medium-app-3-8-concerns-small-team). One crate, module-level layering, traits at boundaries.
- Building a **production system** (multiple teams, resilience, auth, expandable)? → Start at [Stage 3](#stage-3-large-app-8-concerns-multiple-teams). Full workspace, crate-level enforcement, see also [Hexagonal Architecture](#hexagonal-architecture-ports--adapters) and [Resilience Patterns](#resilience-patterns).

### Stage 1: Small App (1-3 concerns, prototyping or solo dev)

A single crate with well-organized modules. No workspace, no traits for internal boundaries.

```
my-app/
├── Cargo.toml          (single [package], edition = "2024")
└── src/
    ├── main.rs          (composition: config → state → router → serve)
    ├── config.rs        (figment/envy config loading)
    ├── models.rs        (domain structs, newtypes, validation)
    ├── handlers.rs      (HTTP handlers — thin, delegate to models/db)
    ├── db.rs            (database queries, connection pool setup)
    └── errors.rs        (unified error type with IntoResponse)
```

**What matters at this stage:**
- Domain logic in pure functions on structs (no framework annotations on domain types)
- One error type with `From` conversions
- Handlers delegate to domain functions — no business logic in route handlers
- Config loaded once in `main()`, passed as `State`
- No traits for internal boundaries — direct function calls are fine
- No workspace overhead

**When to grow:** When `models.rs` exceeds ~500 lines, or you need to share types with a second binary (CLI, worker), or tests require mocking an external dependency.

### Stage 2: Medium App (3-8 concerns, small team)

Split into library + binary(ies). Introduce traits for external dependencies.

```
my-app/
├── Cargo.toml          (single [package] with [[bin]] + [lib])
├── src/
│   ├── main.rs          (composition root — wires everything)
│   ├── lib.rs           (re-exports domain, app, infra modules)
│   ├── domain/
│   │   ├── mod.rs
│   │   ├── order.rs     (entities, value objects, domain errors)
│   │   ├── customer.rs
│   │   └── ports.rs     (trait OrderRepository, trait PaymentGateway)
│   ├── app/
│   │   ├── mod.rs
│   │   └── use_cases.rs (PlaceOrderUseCase, orchestration logic)
│   ├── infra/
│   │   ├── mod.rs
│   │   ├── postgres.rs  (PgOrderRepository implements OrderRepository)
│   │   └── stripe.rs    (StripeGateway implements PaymentGateway)
│   └── api/
│       ├── mod.rs
│       ├── routes.rs    (axum Router setup)
│       └── handlers.rs  (extract → use case → respond)
```

**What changes from Stage 1:**
- Traits appear for external dependencies — enables mocking in tests
- Domain module has zero I/O imports
- Error types split: `DomainError`, `InfraError`, `ApiError` with `From` conversions
- Use case structs take trait-bounded dependencies via constructor injection
- `main()` is the composition root — only place that knows concrete types
- Still one crate — modules provide boundaries, `pub(crate)` hides internals

**When to grow:** When compile times exceed 30s, when multiple teams work on distinct subsystems, when you need different Cargo features for different deployment targets, or when a library crate should be published independently.

### Stage 3: Large App (8+ concerns, multiple teams)

Full Cargo workspace with crate-level boundaries.

```
my-app/
├── Cargo.toml           (workspace, [workspace.dependencies])
├── crates/
│   ├── domain/          (zero infra deps — entities, traits, errors)
│   │   ├── Cargo.toml   (only: uuid, chrono, thiserror, serde)
│   │   └── src/
│   ├── app/             (use cases — depends only on domain)
│   │   ├── Cargo.toml   (only: domain, tracing)
│   │   └── src/
│   ├── infra/           (adapters — depends on domain, app, sqlx, redis, etc.)
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── api/             (HTTP layer — axum, routing, auth middleware)
│   │   ├── Cargo.toml
│   │   └── src/
│   └── cli/             (CLI binary — clap, different entry point)
│       ├── Cargo.toml
│       └── src/
```

**What changes from Stage 2:**
- Crate boundaries enforce dependency direction at `Cargo.toml` level — impossible to accidentally import `sqlx` in domain
- `[workspace.dependencies]` ensures version consistency across crates
- Each crate compiles independently — better incremental builds
- Feature flags gate optional adapters (`redis-cache`, `metrics`)
- CI can test crates independently (`cargo test -p domain`)
- Nested supervision: domain crate might have sub-crates for distinct bounded contexts

### What DOESN'T Change Between Stages

These principles hold at every scale:
- Domain logic lives in pure functions on structs — this never changes
- External dependencies are behind traits — this never changes (Stage 1 uses direct calls, but refactors to traits when testing demands it)
- `Result<T, E>` for fallible operations, `?` for propagation — this never changes
- Composition happens in `main()` — this never changes
- Error types translate at boundaries — this never changes

**The progression is additive.** Add modules, then traits, then crates as needed. Never restructure the fundamentals.

## Inter-Component Communication

How components talk to each other is a critical architectural decision in Rust. Unlike BEAM/Elixir where processes and message passing are built-in, Rust requires explicit choices about communication mechanisms.

### Decision Guide

| Need | Mechanism | When to Use |
|------|-----------|-------------|
| Simple sync call | Direct function call | Default — one component calls another's public API |
| Shared data across async tasks | `Arc<T>` with interior mutability | Connection pools, config, shared caches |
| One-to-one async message passing | `tokio::sync::mpsc` | Producer-consumer, work queues, log shipping |
| One-to-many broadcast | `tokio::sync::broadcast` | Event notification, price feeds, state changes |
| Latest-value watch | `tokio::sync::watch` | Config reload, status updates, health state |
| One-shot response | `tokio::sync::oneshot` | Request-response within async tasks |
| CPU-bound parallel work | `rayon::par_iter()` | Data parallelism, batch processing |
| Cross-service communication | HTTP/gRPC (reqwest/tonic) | Separate deployments, different languages |
| Persistent async jobs | Database-backed queue (custom or crate) | Must survive restarts, need retries |

### Escalation Path

Start with the simplest mechanism. Escalate only when you have the specific problem the next level solves.

```
1. Direct function calls (default — no channel overhead)
   │
   ├── Need async decoupling? → tokio::sync::mpsc (bounded channel)
   │
   ├── Need multiple listeners? → tokio::sync::broadcast
   │
   ├── Need latest value only? → tokio::sync::watch
   │
   ├── Need backpressure? → Bounded mpsc (blocks sender when full)
   │
   ├── Events must survive restarts? → Database-backed queue
   │
   └── Cross-service? → HTTP/gRPC with retry + circuit breaker
```

### Channel Patterns

```rust
// Producer-consumer: bounded mpsc for backpressure
let (tx, mut rx) = tokio::sync::mpsc::channel::<LogEntry>(1000);

// Producer — sends without blocking (or awaits when buffer full)
tx.send(LogEntry { level: Level::Info, message: "started".into() }).await?;

// Consumer — processes at its own pace
tokio::spawn(async move {
    while let Some(entry) = rx.recv().await {
        write_to_file(&entry).await;
    }
});

// Broadcast: one-to-many event notification
let (tx, _) = tokio::sync::broadcast::channel::<PriceUpdate>(100);
let mut rx1 = tx.subscribe(); // Dashboard consumer
let mut rx2 = tx.subscribe(); // Alert consumer
tx.send(PriceUpdate { symbol: "AAPL".into(), price: 185.50 })?;

// Watch: latest-value — readers always see current state
let (tx, rx) = tokio::sync::watch::channel(AppConfig::default());
// Any task can read the latest config without subscribing
let current = rx.borrow().clone();
// Config reloader updates the value
tx.send(new_config)?;
```

### Shared State vs Channels

| Use shared state (`Arc<Mutex<T>>`) when... | Use channels when... |
|---------------------------------------------|---------------------|
| Multiple readers, infrequent writes | Clear producer-consumer relationship |
| Need atomic read-modify-write | Tasks should be decoupled (different lifetimes) |
| State is a single value (counter, cache) | Processing involves I/O or blocking work |
| All accessors are in the same task group | Need backpressure or buffering |

**Shared state anti-patterns:**
- Holding `MutexGuard` across `.await` — use `tokio::sync::Mutex` or restructure
- Using `RwLock` for write-heavy workloads — contention defeats the purpose
- Global `lazy_static!` / `LazyLock` for service instances — use `Arc` + injection instead

### When NOT to Add Channels

Most Rust applications don't need channels for internal component communication. Direct function calls through trait-bounded dependencies are the right default:

```rust
// GOOD — direct call through trait boundary (no channel needed)
pub struct PlaceOrderUseCase<R: OrderRepository, P: PaymentGateway> {
    orders: R,
    payments: P,
}
impl<R: OrderRepository, P: PaymentGateway> PlaceOrderUseCase<R, P> {
    pub async fn execute(&self, input: Input) -> Result<Output, AppError> {
        let order = Order::place(input.items)?;
        self.payments.charge(order.total()).await?;
        self.orders.save(&order).await?;
        Ok(Output::from(order))
    }
}

// OVERKILL — channel between use case and repository (unnecessary indirection)
// Don't do this unless you have a specific reason (batching, rate limiting, etc.)
```

**Signals that you need a channel:**
- Work can be deferred (logging, metrics, notifications)
- Producer is faster than consumer and you need backpressure
- Multiple consumers process different aspects of the same events
- You need to buffer work across async task boundaries

## Refactoring Signals

When existing Rust code needs structural changes. **Priorities: preserve behavior, improve boundaries, increase testability.**

| Signal | Refactoring | How |
|--------|------------|-----|
| Domain struct imports `sqlx`/`axum`/`reqwest` | Extract domain layer | Move to a separate module/crate with zero infra deps |
| Business logic in HTTP handler | Extract use case | Create a struct with `execute()` method, handler delegates |
| `main()` > 100 lines of setup | Extract composition root builder | `AppBuilder::new().with_db(pool).with_routes().build()` |
| Same error mapping repeated 3+ times | Layer-specific error types | Create `DomainError`, `InfraError`, `ApiError` with `From` conversions |
| Test requires running database/HTTP server | Missing trait boundary | Define trait in domain, implement in infra, mock in tests |
| Module > 500 lines | Split by responsibility | Extract sub-modules: `order/mod.rs`, `order/validation.rs`, `order/pricing.rs` |
| Two crates have circular `use` dependencies | Extract shared types | Create `shared` or `common` crate with types both need |
| Whole `Config` struct passed to every service | Extract needed fields | Service takes `smtp_url: String`, not `config: Config` |
| `Arc<Mutex<T>>` used where only reads happen | Use `Arc<RwLock<T>>` or `Arc<T>` | If data is immutable after init, `Arc<T>` with no lock |
| Compile time > 30s for a single crate | Split hot/cold paths | Heavy deps (sqlx, serde derives) in separate crate |
| `unwrap()` / `expect()` in production code paths | Proper error handling | Return `Result`, use `?`, handle at boundaries |
| Single giant `match` on enum growing with variants | Trait-based dispatch | Define trait, implement per variant, dispatch via `dyn Trait` or generics |
| `clone()` called excessively to satisfy borrow checker | Restructure ownership | Use references, `Cow<T>`, or restructure data flow |
| Feature flags scattered through domain logic | Move gates to infra layer | Domain code unconditional, features gate adapter selection |
| Global mutable state (`static mut`, `lazy_static!` with `Mutex`) | Dependency injection | Pass state through constructors, use `Arc<T>` |
| Multiple binaries duplicate setup code | Extract shared library crate | `lib.rs` with builder pattern, binaries call `lib::build_app()` |

## Anti-Patterns Catalog

### Layering Violations

**1. Domain depends on infrastructure**
```rust
// BAD — domain knows about sqlx
impl Order {
    pub async fn save(&self, pool: &PgPool) { /* ... */ }
}
// GOOD — domain defines trait, infra implements
pub trait OrderRepository: Send + Sync {
    async fn save(&self, order: &Order) -> Result<(), RepoError>;
}
```

**2. Infrastructure types leak into domain**
```rust
// BAD — sqlx type in domain struct
pub struct Order {
    pub id: sqlx::types::Uuid,
}
// GOOD — domain value object
pub struct Order {
    pub id: OrderId, // newtype wrapping Uuid
}
```

**3. Business logic in HTTP handlers**
```rust
// BAD — validation and calculation in handler
async fn create_order(Json(body): Json<OrderRequest>) -> impl IntoResponse {
    if body.items.is_empty() { return StatusCode::BAD_REQUEST.into_response(); }
    let total = body.items.iter().map(|i| i.price * i.qty).sum::<f64>();
    // ... save to DB directly ...
}
// GOOD — handler delegates to use case
async fn create_order(
    State(svc): State<Arc<OrderServiceDyn>>,
    Json(input): Json<PlaceOrderInput>,
) -> Result<Json<OrderOutput>, ApiError> {
    Ok(Json(svc.place_order(input).await?))
}
```

**4. Framework annotations on domain types**
```rust
// BAD — domain struct coupled to sqlx and actix
#[derive(FromRow)]
#[get("/orders/{id}")]
pub struct Order { /* ... */ }
// GOOD — domain struct is plain Rust; persistence mapping in infra
#[derive(Debug, Clone, PartialEq)]
pub struct Order { /* ... */ }
// infra/postgres.rs: OrderRecord with #[derive(FromRow)], converts to/from Order
```

### State Management Anti-Patterns

**5. Passing entire Config to services**
```rust
// BAD — every service takes the whole config
pub struct EmailService { config: Config }
// GOOD — take only what you need
pub struct EmailService { smtp_url: String, from_addr: String }
```

**6. Global mutable state as service locator**
```rust
// BAD — hidden global singleton
static SERVICE: LazyLock<Mutex<OrderService>> = LazyLock::new(|| ...);
pub fn get_service() -> &'static Mutex<OrderService> { &SERVICE }
// GOOD — explicit injection
pub struct App {
    order_service: Arc<OrderService>,
}
impl App {
    pub fn new(order_service: Arc<OrderService>) -> Self { Self { order_service } }
}
```

**7. Holding MutexGuard across .await**
```rust
// BAD — blocks other tasks waiting for the lock across an await point
let mut guard = self.state.lock().await;
let result = self.db.query(&guard.query).await; // guard held during I/O!
guard.last_result = result;

// GOOD — minimize lock scope
let query = {
    let guard = self.state.lock().await;
    guard.query.clone()
};
let result = self.db.query(&query).await;
{
    let mut guard = self.state.lock().await;
    guard.last_result = result;
}
```

### Async Anti-Patterns

**8. Blocking the async runtime**
```rust
// BAD — synchronous file I/O on async runtime
async fn process(path: &str) -> Result<String> {
    Ok(std::fs::read_to_string(path)?) // Blocks the executor thread!
}
// GOOD — use async I/O or spawn_blocking
async fn process(path: &str) -> Result<String> {
    Ok(tokio::fs::read_to_string(path).await?)
}
// Or for CPU-heavy work:
let result = tokio::task::spawn_blocking(move || expensive_computation()).await?;
```

**9. Unbounded channels causing memory exhaustion**
```rust
// BAD — unbounded channel with fast producer, slow consumer
let (tx, rx) = tokio::sync::mpsc::unbounded_channel();
// If consumer can't keep up, memory grows without limit

// GOOD — bounded channel provides backpressure
let (tx, rx) = tokio::sync::mpsc::channel(1000); // Buffer limit
// Sender awaits when buffer is full — natural backpressure
```

**10. Spawning tasks without join handles**
```rust
// BAD — fire-and-forget, no way to know if task panicked
tokio::spawn(async { do_critical_work().await; });
// GOOD — track the handle, propagate errors
let handle = tokio::spawn(async { do_critical_work().await });
handle.await??; // Propagate both JoinError and inner error
```

### Dependency Injection Anti-Patterns

**11. Fat traits forcing unnecessary implementations**
```rust
// BAD — every implementor must provide all methods
trait Repository {
    async fn find(&self, id: Id) -> Result<Entity, Error>;
    async fn save(&self, entity: &Entity) -> Result<(), Error>;
    async fn delete(&self, id: Id) -> Result<(), Error>;
    async fn export_csv(&self) -> Result<String, Error>;
}
// GOOD — segregated traits, compose with bounds
trait Find<T> { async fn find(&self, id: Id) -> Result<Option<T>, Error>; }
trait Save<T> { async fn save(&self, entity: &T) -> Result<(), Error>; }
// Use case: fn process(repo: &(impl Find<Order> + Save<Order>)) { ... }
```

**12. Trait defined next to its implementation**
```rust
// BAD — trait and impl in the same infra module
// infra/postgres.rs
trait OrderRepository { /* ... */ }
struct PgOrderRepository { /* ... */ }
impl OrderRepository for PgOrderRepository { /* ... */ }
// Now domain depends on infra to use the trait!

// GOOD — trait in domain, impl in infra
// domain/ports.rs
trait OrderRepository { /* ... */ }
// infra/postgres.rs
impl OrderRepository for PgOrderRepository { /* ... */ }
```

### Testing Anti-Patterns

**13. Tests that require infrastructure**
```rust
// BAD — unit test hits a real database
#[tokio::test]
async fn test_order_validation() {
    let pool = PgPool::connect("postgres://localhost/test").await.unwrap();
    let repo = PgOrderRepo::new(pool);
    // Testing business logic but requiring Postgres to be running!
}
// GOOD — test domain logic without infrastructure
#[test]
fn test_order_validation() {
    let result = Order::new(customer_id, vec![]); // Pure domain, no I/O
    assert!(matches!(result, Err(DomainError::EmptyOrder)));
}
```

**14. Mocking internals instead of boundaries**
```rust
// BAD — mocking private helper functions
// This couples tests to implementation details

// GOOD — mock at trait boundaries (ports)
let mut mock_repo = MockOrderRepository::new();
mock_repo.expect_save().returning(|_| Ok(()));
let uc = PlaceOrderUseCase::new(mock_repo);
// Test the use case behavior, not internal mechanics
```

**15. No in-memory implementation for integration tests**
```rust
// BAD — only Postgres implementation exists, tests always need DB

// GOOD — provide an in-memory implementation for fast tests
pub struct InMemoryOrderRepo {
    orders: Arc<Mutex<HashMap<OrderId, Order>>>,
}
impl OrderRepository for InMemoryOrderRepo {
    async fn save(&self, order: &Order) -> Result<(), RepoError> {
        self.orders.lock().await.insert(order.id.clone(), order.clone());
        Ok(())
    }
    // ... other methods ...
}
// Integration tests use InMemoryOrderRepo — fast, no external deps
// E2E tests use PgOrderRepo — full stack verification
```

## DI Containers

Rust doesn't have runtime reflection, so DI containers work differently than in Java/C#. Three approaches, from simplest to most structured:

| Approach | When to Use | Complexity |
|----------|-------------|------------|
| **Manual constructor injection** | Small–medium apps, <20 services | Low |
| **Simple `TypeId` container** | Medium apps, need runtime resolution | Medium |
| **Shaku framework** | Large apps, deep dependency graphs, Actix integration | Higher |

**Manual injection** (covered in Composition Root above) — wire dependencies in `main()`. This is the most common and recommended approach in Rust.

**Simple container** — use `TypeId` + `Any` to store and resolve services at runtime. Build a `HashMap<TypeId, Box<dyn Any>>`, register with `.register::<T>(instance)`, resolve with `.resolve::<T>()`.

**Shaku** — provides `#[derive(Component)]` and `#[derive(Interface)]` for compile-time DI with `ContainerBuilder`. Supports singleton scope (default), providers for transient instances, and integrates with Actix Web via `Inject<T>`.

**Lifetime and scope management:**

| Scope | Mechanism | Use For |
|-------|-----------|---------|
| Singleton | `Arc<T>` | Connection pools, config, shared caches |
| Scoped | References with lifetimes | Per-request state, transaction context |
| Transient | `Box::new(T)` per call | Stateless services, formatters |

**Interior mutability for `&self` trait methods:** When your trait has `&self` but implementation needs mutation, use `Mutex<T>`, `RwLock<T>`, or `RefCell<T>` (single-threaded) inside the struct.

See [architecture-examples.md](architecture-examples.md#di-containers) for full implementations of all three approaches.

## Domain Modeling Patterns

How to model domain entities, value objects, and use cases in Rust's type system.

**Rich Domain Entity with State Machine:** Encode entity lifecycle states as enum variants. Each state carries only its valid data. Transitions are methods that consume the current state and return the next — invalid transitions are compile-time errors.

```rust
// State machine enforced by the type system
enum Order {
    Draft(DraftOrder),
    Placed(PlacedOrder),
    Shipped(ShippedOrder),
    Delivered(DeliveredOrder),
    Cancelled(CancelledOrder),
}
// DraftOrder can .place() → PlacedOrder, but not .deliver()
```

**Immutable Entity Pattern:** Use `Clone` + modification methods that return new instances. Good for audit trails, event sourcing, and concurrent access. Expensive for large structs — use `Cow` fields for selective cloning.

**Data Transfer Objects (DTOs):** Separate input DTOs (what the API accepts) from domain entities (internal) from output DTOs (what the API returns). Never expose domain entities directly.

| Layer | Type | Purpose |
|-------|------|---------|
| HTTP input | `CreateOrderRequest` | Validation + deserialization |
| Domain | `Order` | Business logic + invariants |
| HTTP output | `OrderResponse` | Formatted for API consumers |
| Persistence | `OrderRecord` | Maps to DB schema |

**Sensitive data protection:** Implement custom `Debug` that redacts secrets, `Serialize` that masks values, and `Zeroize` on drop for passwords/tokens.

**Presenter / View Model pattern:** Transform domain entities into UI-specific view models. Different presenters for web, mobile, CLI. Domain entities never know about display concerns.

**Mock Repository for testing:** Implement repository traits with `Arc<Mutex<Vec<T>>>` for in-memory test doubles. No external mocking library needed for simple cases.

See [architecture-examples.md](architecture-examples.md#domain-modeling-patterns) for complete implementations.

## Resilience Patterns

Patterns for building fault-tolerant systems that degrade gracefully under failure.

**Retry with Exponential Backoff and Jitter:**
- Base delay doubles each attempt: 100ms → 200ms → 400ms → ...
- Add random jitter (±50%) to prevent thundering herd
- Set max retries and max delay cap
- Only retry on transient errors (timeouts, 503s), never on 4xx client errors

```rust
// Core formula
let delay = base_delay * 2u64.pow(attempt);
let jitter = rng.gen_range(0..delay / 2);
let actual = Duration::from_millis(delay + jitter).min(max_delay);
```

**Retryable trait abstraction:** Define `trait Retryable { fn is_retryable(&self) -> bool; }` on your error types. Implement a generic `retry_with_backoff<F, T, E: Retryable>()` that checks `is_retryable()` before each retry.

**Circuit Breaker:**

| State | Behavior | Transitions |
|-------|----------|-------------|
| **Closed** | Normal operation, count failures | → Open when failures ≥ threshold |
| **Open** | Reject all calls immediately | → Half-Open after timeout |
| **Half-Open** | Allow one probe request | → Closed on success, → Open on failure |

Key config: `failure_threshold` (e.g., 5), `success_threshold` (e.g., 1), `timeout` (e.g., 30s).

**Graceful Degradation:** When a dependency fails, return cached/stale data or a reduced-functionality response instead of an error. Use circuit breaker state to trigger fallback behavior automatically.

See [architecture-examples.md](architecture-examples.md#resilience-patterns) for full implementations.

## Authorization Patterns

Authentication verifies identity (see [web-apis.md](web-apis.md) for JWT/Argon2). Authorization decides what an authenticated user can *do*. This section covers authorization architecture — where permission checks live and how to enforce them.

### Role-Based Access Control (RBAC)

Define roles as enums, permissions as trait methods:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Deserialize)]
pub enum Role {
    Viewer,
    Editor,
    Admin,
    SuperAdmin,
}

impl Role {
    pub fn can_read(&self) -> bool { true }                  // All roles
    pub fn can_write(&self) -> bool { !matches!(self, Self::Viewer) }
    pub fn can_delete(&self) -> bool { matches!(self, Self::Admin | Self::SuperAdmin) }
    pub fn can_manage_users(&self) -> bool { matches!(self, Self::SuperAdmin) }
}
```

### Authorization Middleware (axum)

Extract the user's role from JWT claims and enforce at the HTTP layer:

```rust
use axum::{extract::Request, middleware::Next, response::Response};

/// Middleware that requires a minimum role for the route.
pub async fn require_role(
    role: Role,
) -> impl Fn(Request, Next) -> impl Future<Output = Result<Response, ApiError>> {
    move |req: Request, next: Next| async move {
        let claims = req.extensions().get::<Claims>()
            .ok_or(ApiError::Unauthorized)?;
        if !claims.role.has_at_least(role) {
            return Err(ApiError::Forbidden("insufficient permissions"));
        }
        Ok(next.run(req).await)
    }
}

// Usage in router:
let admin_routes = Router::new()
    .route("/users", post(create_user))
    .route("/users/{id}", delete(delete_user))
    .layer(axum::middleware::from_fn(|| require_role(Role::Admin)));
```

### Domain-Layer Authorization Guards

For fine-grained authorization (row-level, resource-ownership), enforce in the domain layer — not middleware:

```rust
// Domain: the entity knows its own access rules
impl Project {
    pub fn can_modify(&self, user_id: UserId, role: Role) -> bool {
        self.owner_id == user_id || role.can_write()
    }
}

// Application layer: use case checks before mutating
pub async fn update_project(
    &self, user: &AuthUser, project_id: ProjectId, update: ProjectUpdate,
) -> Result<Project, UseCaseError> {
    let project = self.repo.find(project_id).await?
        .ok_or(UseCaseError::NotFound)?;

    if !project.can_modify(user.id, user.role) {
        return Err(UseCaseError::Forbidden);
    }

    let updated = project.apply_update(update)?;  // Domain validation
    self.repo.save(&updated).await?;
    Ok(updated)
}
```

### Multi-Tenant Data Isolation

For SaaS applications where tenants must never see each other's data:

```rust
// Newtype ensures tenant context is threaded through all queries
#[derive(Clone, Copy)]
pub struct TenantId(Uuid);

// Repository trait requires tenant context — impossible to forget
#[async_trait]
pub trait OrderRepository {
    async fn find(&self, tenant: TenantId, id: OrderId) -> Result<Option<Order>, RepoError>;
    async fn list(&self, tenant: TenantId, filter: OrderFilter) -> Result<Vec<Order>, RepoError>;
    // No method exists without TenantId — architectural guarantee
}

// Infrastructure: every query includes WHERE tenant_id = $1
impl OrderRepository for PgOrderRepo {
    async fn find(&self, tenant: TenantId, id: OrderId) -> Result<Option<Order>, RepoError> {
        sqlx::query_as!(Order,
            "SELECT * FROM orders WHERE tenant_id = $1 AND id = $2",
            tenant.0, id.0
        ).fetch_optional(&self.pool).await.map_err(Into::into)
    }
}
```

**Where to enforce authorization:**
| Check Type | Layer | Example |
|-----------|-------|---------|
| Route-level role gating | HTTP middleware | "Only admins can access `/admin/*`" |
| Resource ownership | Application (use case) | "Users can only edit their own projects" |
| Tenant isolation | Repository trait | "Queries always scoped to tenant" |
| Field-level visibility | DTO/presenter | "Viewers can't see `cost_price`" |

## High-Throughput Ingestion (Sensor / IoT APIs)

Sensor telemetry APIs have different requirements than CRUD — high volume, fire-and-forget, backpressure tolerance. Don't process each reading synchronously.

**Pattern: Buffered channel → batch writer**

```rust
// Ingestion endpoint: validate, enqueue, return immediately
async fn ingest_readings(
    State(tx): State<mpsc::Sender<SensorReading>>,
    Json(batch): Json<Vec<SensorReadingInput>>,
) -> Result<StatusCode, ApiError> {
    for input in batch {
        let reading = SensorReading::validate(input)?;  // Domain validation
        tx.try_send(reading).map_err(|_| ApiError::ServiceOverloaded)?;  // Backpressure
    }
    Ok(StatusCode::ACCEPTED)  // 202 — accepted for processing, not yet persisted
}

// Background writer: drains channel in batches
async fn batch_writer(mut rx: mpsc::Receiver<SensorReading>, pool: PgPool) {
    let mut buf = Vec::with_capacity(500);
    loop {
        // Drain up to 500 readings or wait 100ms
        let deadline = tokio::time::sleep(Duration::from_millis(100));
        tokio::pin!(deadline);
        loop {
            tokio::select! {
                Some(reading) = rx.recv() => {
                    buf.push(reading);
                    if buf.len() >= 500 { break; }
                }
                _ = &mut deadline => break,
            }
        }
        if !buf.is_empty() {
            if let Err(e) = bulk_insert(&pool, &buf).await {
                tracing::error!(count = buf.len(), error = %e, "batch insert failed");
                // Retain for retry or push to dead-letter queue
            }
            buf.clear();
        }
    }
}
```

**Key differences from CRUD:**
- Return `202 Accepted`, not `200 OK` — the data is enqueued, not persisted yet
- Use bounded `mpsc::channel(10_000)` for backpressure — `try_send` returns error when full
- Batch writes (`COPY` or multi-row `INSERT`) instead of per-row inserts
- Per-device rate limiting via `DashMap<DeviceId, RateLimiter>` if needed
- Consider TimescaleDB hypertables or ClickHouse for time-series storage at scale

## Nanoservices Architecture

A workspace-based pattern for building modular applications that can deploy as a monolith or split into microservices.

**Core idea:** Organize into three layers as separate crates in a Cargo workspace:

| Layer | Crate | Depends On | Contains |
|-------|-------|------------|----------|
| **Core** | `core` | Nothing | Domain types, business logic, trait definitions |
| **DAL** | `dal` | `core` | Storage implementations (Postgres, JSON files, Redis) |
| **Networking** | `server` | `core`, `dal` | HTTP handlers, middleware, routing |

**Feature-gated DAL:** Use Cargo features to swap storage backends at compile time:
```toml
[features]
default = ["json-file"]
json-file = ["serde_json"]
sqlx-postgres = ["sqlx"]
```

**Glue module pattern:** A shared crate for cross-cutting concerns (auth tokens, error types, the `safe_eject!` macro). All workspace members depend on `glue` for common types.

**`safe_eject!` macro:** Wraps function calls to convert diverse error types into a unified `NanoServiceError` — eliminates boilerplate `map_err` chains.

**Build commands:**
- `cargo build` — monolith with default features
- `cargo build -p server --features sqlx-postgres` — single service with Postgres
- Each crate can also be deployed independently

See [architecture-examples.md](architecture-examples.md#nanoservices-architecture) for complete workspace setup and implementation.

## Async Logging Architecture

**Why not `println!`:** Blocks the thread, no structured metadata, no log levels, no routing to external systems. Always use `tracing` (see Production Patterns above for basic setup).

**Log level guidelines:**

| Level | When to Use | Production |
|-------|-------------|------------|
| **Error** | Critical failures requiring attention | Alert + possibly retry |
| **Warn** | Non-critical failures, recoverable | Monitor + alert threshold |
| **Info** | Normal operations (start/stop, checkpoints) | Always enabled |
| **Debug** | Detailed flow for debugging | Disable in production |
| **Trace** | Very granular (per-iteration, per-byte) | Never in production |

**Warning vs Error:** If you can retroactively fix the issue (email failed but order saved), use warn. If the user must be informed and retry (DB insert failed), use error.

**Custom HTTP logging middleware:** Implement `Transform` + `Service` traits (Actix) or use `tower_http::trace::TraceLayer` (axum) to log method, path, status, and duration for every request.

**Actor-based remote logging:** For non-blocking log shipping to Elasticsearch/Loki/etc., spawn a background task that receives log entries via `tokio::sync::mpsc` and batches them for network delivery. Never block request handlers waiting for log writes.

**Feature-gated logging:** Use Cargo features to conditionally compile remote logging support — keep the binary lean when only console logging is needed.

See [architecture-examples.md](architecture-examples.md#actor-based-async-logging) for middleware and actor implementations.

## Facade Crate Pattern

A facade crate re-exports multiple subcrates through a single entry point, simplifying the dependency graph for consumers.

**ripgrep's `grep` crate** is the canonical example — it contains no code, only re-exports:

```rust
// grep/src/lib.rs — pure facade, zero logic
pub use grep_cli as cli;
pub use grep_matcher as matcher;
#[cfg(feature = "pcre2")]
pub use grep_pcre2 as pcre2;
pub use grep_printer as printer;
pub use grep_regex as regex;
pub use grep_searcher as searcher;
```

```toml
# grep/Cargo.toml — declares all subcrates as dependencies
[dependencies]
grep-cli = { version = "0.1.11", path = "../cli" }
grep-matcher = { version = "0.1.7", path = "../matcher" }
grep-printer = { version = "0.2.3", path = "../printer" }
grep-regex = { version = "0.1.13", path = "../regex" }
grep-searcher = { version = "0.1.14", path = "../searcher" }

[dependencies.grep-pcre2]
version = "0.1.8"
path = "../pcre2"
optional = true

[features]
pcre2 = ["grep-pcre2"]
```

**When to use:**
- Multiple subcrates that are usually consumed together
- Want to provide a "batteries included" experience
- Each subcrate is also independently publishable
- Version management — consumers track one version, not five

**Variation: conditional facade (serde pattern):**

```rust
// serde/src/lib.rs — facade with conditional re-exports
// The actual implementation lives in serde_core (no_std compatible)
// The facade adds std-dependent features and derives

#![cfg_attr(not(feature = "std"), no_std)]

// Re-export everything from the core crate
pub use serde_core::*;

// Conditionally add std-only derives
#[cfg(feature = "derive")]
pub use serde_derive::{Serialize, Deserialize};
```

This enables one crate name (`serde`) for users while splitting internals by capability (std vs no_std). The core crate has zero std dependencies; the facade adds them behind feature flags.

**When NOT to use:**
- Single-crate projects
- Subcrates are genuinely independent (no shared consumer)

## Enum-Based Polymorphism (vs `dyn Trait`)

When the set of implementations is known at compile time, use an enum instead of `dyn Trait`. This is faster (no vtable indirection), smaller (no heap allocation), and enables exhaustive matching.

**ripgrep uses this for its core polymorphism:**

```rust
// PatternMatcher — enum dispatch instead of Box<dyn Matcher>
enum PatternMatcher {
    RustRegex(grep_regex::RegexMatcher),
    #[cfg(feature = "pcre2")]
    PCRE2(grep_pcre2::RegexMatcher),
}

impl PatternMatcher {
    fn search(&self, haystack: &[u8]) -> Result<bool, Error> {
        match self {
            PatternMatcher::RustRegex(m) => m.search(haystack),
            #[cfg(feature = "pcre2")]
            PatternMatcher::PCRE2(m) => m.search(haystack),
        }
    }
}

// Similarly for output format selection
enum Printer<W: WriteColor> {
    Standard(grep_printer::Standard<W>),
    Summary(grep_printer::Summary<W>),
    JSON(grep_printer::JSON<W>),
}
```

**Decision guide:**

| Criteria | Enum dispatch | `dyn Trait` |
|----------|--------------|-------------|
| Set of types known at compile time | **Yes — use enum** | No — open set |
| Need to add types without recompiling | No | **Yes** |
| Performance critical | **Yes — no indirection** | Acceptable overhead |
| Feature-gated variants | **Yes** — `#[cfg]` on variants | Possible but awkward |
| Need heterogeneous collection of unknown types | No | **Yes** — `Vec<Box<dyn T>>` |

## Tower Layer/Service Composition (axum Architecture)

axum's architecture is built entirely on Tower's `Service` and `Layer` traits. Understanding this is essential for axum middleware:

```
Layer transforms Service → Service
Service handles Request → Response

                Layer
                  │
    ┌─────────────┼─────────────┐
    │  Outer Service (wrapped)  │
    │  ┌─────────────────────┐  │
    │  │  Inner Service      │  │
    │  │  (your handler)     │  │
    │  └─────────────────────┘  │
    └───────────────────────────┘
```

**axum middleware construction methods:**

```rust
use axum::{Router, middleware};

let app = Router::new()
    .route("/api/users", get(list_users))
    // from_fn — simplest, write async functions as middleware
    .layer(middleware::from_fn(auth_middleware))
    // map_request — transform the request before the handler
    .layer(middleware::map_request(add_request_id))
    // map_response — transform the response after the handler
    .layer(middleware::map_response(add_cors_headers))
    // Tower layers work directly
    .layer(tower_http::trace::TraceLayer::new_for_http())
    .layer(tower_http::timeout::TimeoutLayer::new(Duration::from_secs(30)));

// from_fn middleware example
async fn auth_middleware(
    headers: HeaderMap,
    request: Request,
    next: middleware::Next,
) -> Result<Response, StatusCode> {
    let token = headers.get("Authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;
    // Validate token...
    Ok(next.run(request).await)
}

// State-aware middleware
async fn rate_limit(
    State(limiter): State<Arc<RateLimiter>>,
    request: Request,
    next: middleware::Next,
) -> Result<Response, StatusCode> {
    limiter.check().map_err(|_| StatusCode::TOO_MANY_REQUESTS)?;
    Ok(next.run(request).await)
}
// Applied with: .layer(middleware::from_fn_with_state(limiter, rate_limit))
```

**Key rule:** Layers are applied bottom-to-top — the last `.layer()` call wraps outermost. Outermost layers see the request first and the response last.

**axum's `Router<S>` state erasure pattern:**
```rust
// Router<S> tracks what state is still needed
let api = Router::new()
    .route("/users", get(list_users))  // Router<AppState> — needs AppState
    .with_state(state);                // Router<()> — state provided, ready to serve

// Only Router<()> can be passed to axum::serve()
```

## Workspace Lint Inheritance

Centralize lint configuration at the workspace level (axum pattern):

```toml
# Workspace root Cargo.toml
[workspace.lints.rust]
unreachable_pub = "warn"
missing_debug_implementations = "warn"
# missing_docs = "warn"  # Enable when ready

[workspace.lints.clippy]
dbg_macro = "warn"
print_stdout = "warn"
needless_pass_by_value = "warn"
mutex_atomic = "warn"                    # Use atomics for simple counters
# Allow type_complexity — generic-heavy frameworks need this
type_complexity = "allow"

# Member crate Cargo.toml — inherit all workspace lints
[lints]
workspace = true
```

**Advantages over per-crate `#![warn(...)]`:**
- Single source of truth — change once, applies everywhere
- New crates get lints automatically
- Easy to audit (`grep "workspace = true" crates/*/Cargo.toml`)
- Can still override per-crate: `[lints.clippy]\nsome_lint = "allow"`

## Two-Stage Argument Parsing (ripgrep Pattern)

For complex CLI applications, parse arguments in two phases:

```
Phase 1: LowArgs (raw parse)           Phase 2: HiArgs (validated, derived)
─────────────────────────              ──────────────────────────────────
Parse CLI tokens into raw              Apply heuristics and defaults
values without validation              Resolve conflicts between flags
Fast, no side effects                  Build derived objects (matchers,
                                       printers, globs)
                                       Detect terminal capabilities
                                       Set thread counts based on input
```

**Why two stages:**
- Separates parsing (fast, testable) from configuration (complex, may fail)
- Heuristics can inspect the full set of flags before committing
- Documentation generation uses `LowArgs` metadata without building `HiArgs`
- Shell completion uses `LowArgs` without executing any search logic

**ripgrep's 58-field `HiArgs` struct** is built from `LowArgs` through extensive heuristic application: terminal detection adjusts color defaults, thread count optimizes based on path count, memory-mapping strategy adapts to file count, and conflicting flags are resolved with documented precedence rules.

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: ownership, traits, modules, cargo features, API design
- **[architecture-examples.md](architecture-examples.md)** — Complete worked examples for patterns in this file
- **[domain-patterns.md](domain-patterns.md)** — DDD patterns, bounded contexts, event sourcing, CQRS
- **[async-concurrency.md](async-concurrency.md)** — Tokio runtime, Tower services, graceful shutdown
- **[services.md](services.md)** — Microservices, service discovery, Redis, resilience patterns
- **[deployment.md](deployment.md)** — Build profiles, Docker, CI/CD, observability
- **[error-handling.md](error-handling.md)** — Multi-layer error translation, domain error design

