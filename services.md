# Services and Infrastructure

Microservices architecture, service discovery, Tower middleware, Redis caching and job queues, resilience patterns, and TCP/TLS networking.

## Rules for Services & Infrastructure (LLM)

1. **ALWAYS use Tower `Service` trait for composable middleware** — retry, timeout, rate-limit, and load balancing compose via `ServiceBuilder::new().layer()` rather than hand-rolled wrappers
2. **ALWAYS implement retry with Tower's `Policy` trait** — separate retry logic (when, how many, backoff) from the transport; `clone_request()` controls whether retry is even possible
3. **ALWAYS use exponential backoff with jitter for retries** — fixed delays cause thundering herd; use `backoff = base * 2^attempt * (1 + random(0..0.5))` capped at a maximum
4. **ALWAYS add idempotency keys for non-idempotent operations** — POST/PUT retries without idempotency keys cause duplicate side effects; pass `Idempotency-Key` header and cache responses server-side
5. **ALWAYS use circuit breakers for external service calls** — prevent cascade failures; track failures per-instance, not per-service; use Closed→Open→HalfOpen state machine
6. **NEVER create a new `reqwest::Client` per request** — `Client` holds a connection pool internally (is `Arc` inside); create once, clone cheaply, share via `State`
7. **ALWAYS separate liveness from readiness probes** — liveness = "process is running" (always 200), readiness = "can serve traffic" (checks DB, dependencies); Kubernetes uses these differently
8. **PREFER `redis::aio::MultiplexedConnection` over `get_async_connection`** — multiplexed connections share a single TCP socket across concurrent commands, avoiding connection pool exhaustion

### Common Mistakes (BAD/GOOD)

**Hand-rolled retry vs Tower composition:**
```rust
// BAD: manual retry loop duplicated across every service call
loop {
    match client.get(url).send().await {
        Ok(resp) if resp.status().is_success() => return Ok(resp),
        _ => { attempt += 1; tokio::time::sleep(delay).await; }
    }
}

// GOOD: composable Tower retry — reusable across all services
let svc = ServiceBuilder::new()
    .retry(MyRetryPolicy::new(3, Duration::from_millis(100)))
    .timeout(Duration::from_secs(5))
    .service(HttpService::new(client));
```

**Creating clients per request:**
```rust
// BAD: new client (and connection pool) per request
async fn get_user(id: i32) -> Result<User> {
    let client = reqwest::Client::new();  // expensive!
    client.get(format!("{}/users/{}", BASE_URL, id)).send().await?.json().await
}

// GOOD: shared client, created once
async fn get_user(client: &reqwest::Client, id: i32) -> Result<User> {
    client.get(format!("{}/users/{}", BASE_URL, id)).send().await?.json().await
}
```

**Retry without idempotency:**
```rust
// BAD: retrying a POST without idempotency key — may create duplicates
client.post("/orders").json(&order).send().await?;

// GOOD: idempotency key ensures at-most-once execution
client.post("/orders")
    .header("Idempotency-Key", uuid::Uuid::new_v4().to_string())
    .json(&order).send().await?;
```

### Section Index

| Section | Content |
|---------|---------|
| [Tower Service Composition](#tower-service-composition) | Service trait, Layer, ServiceBuilder, retry, timeout |
| [Microservices Trade-offs](#microservices-trade-offs) | When microservices help vs hurt, nanoservices pattern |
| [Kernel Pattern](#kernel-pattern) | Feature-gated monolith/microservice hybrid |
| [Inter-Service HTTP Communication](#inter-service-http-communication) | reqwest, retry, circuit breakers between services |
| [User Tethering and Connection Tables](#user-tethering-and-connection-tables) | WebSocket connection management, session routing |
| [Docker Compose for Multi-Service Development](#docker-compose-for-multi-service-development) | Local dev environment setup |
| [Service Discovery](#service-discovery) | Kubernetes, kube-rs, health checks, readiness probes |
| [Redis Caching](#redis-caching) | Cache-aside, write-through, TTL strategies |
| [Redis Job Queues](#redis-job-queues) | Background processing, BLPOP, reliable queues |
| [Redis User Session Caching](#redis-user-session-caching) | Session storage, token management |
| [Custom Redis Modules in Rust](#custom-redis-modules-in-rust) | Extending Redis with Rust modules |
| [Re-Queue Pattern](#re-queue-pattern-for-continuation-tasks) | Continuation tasks, work-stealing |
| [Resilience Patterns](#resilience-patterns) | Circuit breakers, retries, bulkheads, timeouts |
| [CAP Theorem and Trade-offs](#cap-theorem-and-trade-offs) | Consistency vs availability, partition tolerance |
| [Consensus Algorithms](#consensus-algorithms) | Raft, leader election |
| [Distributed Tracing](#distributed-tracing-with-opentelemetry) | OpenTelemetry, trace context propagation |
| [TCP Server/Client](#tcp-serverclient) | Raw TCP with tokio |
| [TLS/HTTPS](#tlshttps) | rustls, certificate configuration |
| [Inter-Context Communication](#inter-context-communication) | Cross-service messaging patterns |
| [Protocol Versioning](#protocol-versioning) | Schema evolution, backward compatibility |

## Tower Service Composition

Tower is the middleware framework underpinning axum, hyper, and tonic. Its core abstraction is the `Service` trait — a function from request to response that supports backpressure.

### The Service Trait

```rust
use std::future::Future;
use std::task::{Context, Poll};

// Tower's core abstraction (simplified)
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    /// Check if the service is ready to accept a request.
    /// Enables backpressure — return Poll::Pending to slow callers.
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

    /// Process a request. Only call after poll_ready returns Ready(Ok(())).
    fn call(&mut self, req: Request) -> Self::Future;
}
```

### Layer Composition with ServiceBuilder

Layers wrap services to add behavior. `ServiceBuilder` composes them in order (outermost first):

```rust
use tower::ServiceBuilder;
use std::time::Duration;

let service = ServiceBuilder::new()
    // Layers execute top-to-bottom on request, bottom-to-top on response
    .timeout(Duration::from_secs(10))        // 1. Timeout entire request
    .rate_limit(100, Duration::from_secs(1)) // 2. Rate limit to 100 req/s
    .retry(MyRetryPolicy::default())         // 3. Retry on failure
    .service(MyHttpService::new());          // Inner service
```

### Retry with Policy Trait

Tower's retry separates the retry decision from the transport. The `Policy` trait controls when and how to retry:

```rust
use tower::retry::Policy;
use std::future::Ready;

#[derive(Clone)]
struct RetryPolicy {
    max_retries: u32,
    remaining: u32,
}

impl RetryPolicy {
    fn new(max_retries: u32) -> Self {
        Self { max_retries, remaining: max_retries }
    }
}

impl<Req: Clone, Res, E> Policy<Req, Res, E> for RetryPolicy {
    type Future = Ready<()>;

    /// Decide whether to retry. Returns Some(delay_future) to retry, None to stop.
    fn retry(&mut self, _req: &mut Req, result: &mut Result<Res, E>) -> Option<Self::Future> {
        match result {
            Ok(_) => None,  // Success — don't retry
            Err(_) if self.remaining > 0 => {
                self.remaining -= 1;
                Some(std::future::ready(()))  // Retry immediately
            }
            Err(_) => None,  // Exhausted retries
        }
    }

    /// Clone the request for retry. Return None if request can't be retried.
    fn clone_request(&mut self, req: &Req) -> Option<Req> {
        Some(req.clone())
    }
}
```

### Retry with Backoff

For production use, add exponential backoff using `tokio::time::sleep`:

```rust
use std::pin::Pin;

#[derive(Clone)]
struct ExponentialBackoff {
    max_retries: u32,
    remaining: u32,
    base_delay: Duration,
}

impl<Req: Clone, Res, E> Policy<Req, Res, E> for ExponentialBackoff {
    type Future = Pin<Box<dyn Future<Output = ()> + Send>>;

    fn retry(&mut self, _req: &mut Req, result: &mut Result<Res, E>) -> Option<Self::Future> {
        match result {
            Ok(_) => None,
            Err(_) if self.remaining > 0 => {
                self.remaining -= 1;
                let attempt = self.max_retries - self.remaining;
                let delay = self.base_delay * 2u32.pow(attempt - 1);
                // Add jitter to prevent thundering herd
                let jitter = Duration::from_millis(rand::random::<u64>() % delay.as_millis() as u64 / 2);
                Some(Box::pin(tokio::time::sleep(delay + jitter)))
            }
            Err(_) => None,
        }
    }

    fn clone_request(&mut self, req: &Req) -> Option<Req> {
        Some(req.clone())
    }
}
```

### Timeout and Rate Limiting

```rust
use tower::timeout::Timeout;
use tower::limit::RateLimit;

// Timeout wraps a service with a deadline
let svc = Timeout::new(inner_service, Duration::from_secs(5));
// On timeout, returns tower::timeout::error::Elapsed

// RateLimit applies token-bucket rate limiting
// poll_ready returns Pending when rate exceeded (backpressure)
let svc = RateLimit::new(inner_service, 100, Duration::from_secs(1));
```

### Load Measurement for Balancing

Tower's `Load` trait measures service capacity for load balancing decisions:

```rust
use tower::load::Load;

// PendingRequests: tracks in-flight requests as load metric
// PeakEwma: exponentially weighted moving average of peak latency
// Constant: fixed load value for uniform distribution

// Load balancers use these metrics to route requests:
// - Lower Load::metric() → prefer this instance
// - Balance layer selects least-loaded service automatically
```

### Custom Tower Service Example

```rust
use tower::Service;
use std::task::{Context, Poll};

/// Middleware that adds request-id header
#[derive(Clone)]
struct RequestId<S> {
    inner: S,
}

impl<S, B> Service<http::Request<B>> for RequestId<S>
where
    S: Service<http::Request<B>>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, mut req: http::Request<B>) -> Self::Future {
        req.headers_mut().insert(
            "x-request-id",
            uuid::Uuid::new_v4().to_string().parse().unwrap(),
        );
        self.inner.call(req)
    }
}
```

## Microservices Trade-offs

### When Microservices Cause Problems

**Container orchestration complexity** — each service needs Docker builds, network config, environment variables. Tweaking one requires changes in others.

**Flying blind** — services can't see what others have already checked. A single order might call Auth 50-300 times because each service defensively re-validates.

**Network call latency** — function calls take nanoseconds; network calls take milliseconds (10,000-10,000,000x slower). Each call requires serialize → TCP handshake → send → wait → deserialize.

**Circular dependencies** — the compiler catches circular module dependencies. In microservices, A→B→C→A compiles fine but creates runtime deadlocks.

### When Microservices Help

- **Release isolation** — small teams own and release services independently
- **Code isolation** — bad code is contained to one service, worst case: rewrite it
- **Experimentation** — try new languages/frameworks one service at a time

### Nanoservices: Best of Both Worlds

Feature-gated libraries that compile as monolith or microservices:

```rust
// Direct database access (nanoseconds)
#[cfg(feature = "core-postgres")]
pub async fn get_user(pool: &PgPool, id: i32) -> Result<User, Error> {
    sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_one(pool).await.map_err(Into::into)
}

// HTTP access (milliseconds)
#[cfg(feature = "http")]
pub async fn get_user(base_url: &str, id: i32) -> Result<User, Error> {
    reqwest::get(format!("{}/api/v1/users/{}", base_url, id))
        .await?.json().await.map_err(Into::into)
}
```

## Kernel Pattern

The kernel exposes core functionality via shared libraries, avoiding HTTP overhead between co-located services.

### Structure

```
nanoservices/
├── auth-server/
│   ├── core/           # Shared business logic (the kernel)
│   │   ├── Cargo.toml  # feature-gated: core-postgres OR http
│   │   └── src/
│   └── server/         # HTTP wrapper
└── app-server/
    ├── core/
    └── server/
```

### Feature-Gated Cargo.toml

```toml
[package]
name = "auth-core"

[features]
default = []
core-postgres = ["sqlx"]
http = ["reqwest"]

[dependencies]
serde = { version = "1.0", features = ["derive"] }
thiserror = "1.0"
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio"], optional = true }
reqwest = { version = "0.12", features = ["json"], optional = true }
```

### Feature-Gated Implementation

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User { pub id: i32, pub email: String, pub unique_id: String }

#[cfg(feature = "core-postgres")]
pub async fn get_user_by_unique_id(
    pool: &sqlx::PgPool, unique_id: &str,
) -> Result<User, AuthError> {
    sqlx::query_as!(User, "SELECT id, email, unique_id FROM users WHERE unique_id = $1", unique_id)
        .fetch_optional(pool).await?.ok_or(AuthError::NotFound)
}

#[cfg(feature = "http")]
pub async fn get_user_by_unique_id(
    base_url: &str, unique_id: &str,
) -> Result<User, AuthError> {
    let url = format!("{}/api/v1/users/unique/{}", base_url, unique_id);
    let response = reqwest::Client::new().get(&url).send().await
        .map_err(|e| AuthError::Network(e.to_string()))?;
    if response.status().is_success() {
        response.json().await.map_err(|e| AuthError::Parse(e.to_string()))
    } else {
        Err(AuthError::NotFound)
    }
}

#[derive(Debug, thiserror::Error)]
pub enum AuthError {
    #[error("User not found")] NotFound,
    #[error("Network error: {0}")] Network(String),
    #[error("Parse error: {0}")] Parse(String),
    #[cfg(feature = "core-postgres")]
    #[error("Database error: {0}")] Database(#[from] sqlx::Error),
}
```

### Consuming the Kernel

```toml
# In app-server/core/Cargo.toml
[dependencies]
# HTTP variant for remote calls:
auth-core = { path = "../../auth-server/core", features = ["http"] }
# Or postgres variant for same-process:
# auth-core = { path = "../../auth-server/core", features = ["core-postgres"] }
```

## Inter-Service HTTP Communication

### Service Client

```rust
use reqwest::{Client, StatusCode};
use serde::{Serialize, de::DeserializeOwned};

pub struct ServiceClient {
    client: Client,
    base_url: String,
}

impl ServiceClient {
    pub fn new(base_url: String) -> Self {
        Self { client: Client::new(), base_url }
    }

    pub async fn get<T: DeserializeOwned>(
        &self, path: &str, token: Option<&str>,
    ) -> Result<T, ServiceError> {
        let mut req = self.client.get(format!("{}{}", self.base_url, path));
        if let Some(t) = token { req = req.header("token", t); }
        let resp = req.send().await.map_err(|e| ServiceError::Network(e.to_string()))?;
        self.handle_response(resp).await
    }

    pub async fn post<T: DeserializeOwned, B: Serialize>(
        &self, path: &str, body: &B, token: Option<&str>,
    ) -> Result<T, ServiceError> {
        let mut req = self.client.post(format!("{}{}", self.base_url, path)).json(body);
        if let Some(t) = token { req = req.header("token", t); }
        let resp = req.send().await.map_err(|e| ServiceError::Network(e.to_string()))?;
        self.handle_response(resp).await
    }

    async fn handle_response<T: DeserializeOwned>(
        &self, response: reqwest::Response,
    ) -> Result<T, ServiceError> {
        match response.status() {
            StatusCode::OK | StatusCode::CREATED =>
                response.json().await.map_err(|e| ServiceError::Parse(e.to_string())),
            StatusCode::NOT_FOUND => Err(ServiceError::NotFound),
            StatusCode::UNAUTHORIZED => Err(ServiceError::Unauthorized),
            status => {
                let body = response.text().await.unwrap_or_default();
                Err(ServiceError::Other(format!("{}: {}", status, body)))
            }
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("Network error: {0}")] Network(String),
    #[error("Parse error: {0}")] Parse(String),
    #[error("Not found")] NotFound,
    #[error("Unauthorized")] Unauthorized,
    #[error("Service error: {0}")] Other(String),
}
```

### Descriptor Pattern for Service Abstraction

```rust
pub trait AuthService: Send + Sync {
    fn get_user(&self, unique_id: &str)
        -> impl Future<Output = Result<User, ServiceError>> + Send;
}

pub struct HttpAuthService { client: ServiceClient }

impl AuthService for HttpAuthService {
    async fn get_user(&self, unique_id: &str) -> Result<User, ServiceError> {
        self.client.get(&format!("/api/v1/users/unique/{}", unique_id), None).await
    }
}

#[cfg(feature = "core-postgres")]
pub struct DirectAuthService { pool: PgPool }

#[cfg(feature = "core-postgres")]
impl AuthService for DirectAuthService {
    async fn get_user(&self, unique_id: &str) -> Result<User, ServiceError> {
        auth_core::get_user_by_unique_id(&self.pool, unique_id).await
            .map_err(|e| ServiceError::Other(e.to_string()))
    }
}
```

## User Tethering and Connection Tables

In microservices, each service often needs to know which user is making the request. Rather than passing raw credentials, services share connection tables that map session tokens to user IDs.

### Connection Table Pattern

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use uuid::Uuid;

/// Shared connection state across services
#[derive(Debug, Clone)]
pub struct UserConnection {
    pub user_id: i32,
    pub unique_id: Uuid,
    pub permissions: Vec<String>,
    pub connected_at: chrono::DateTime<chrono::Utc>,
}

pub struct ConnectionTable {
    connections: Arc<RwLock<HashMap<Uuid, UserConnection>>>,
}

impl ConnectionTable {
    pub fn new() -> Self {
        Self { connections: Arc::new(RwLock::new(HashMap::new())) }
    }

    pub fn tether(&self, token: Uuid, connection: UserConnection) {
        self.connections.write().unwrap().insert(token, connection);
    }

    pub fn get(&self, token: &Uuid) -> Option<UserConnection> {
        self.connections.read().unwrap().get(token).cloned()
    }

    pub fn release(&self, token: &Uuid) {
        self.connections.write().unwrap().remove(token);
    }
}
```

### Token Extraction Middleware

```rust
use axum::{
    extract::Request,
    http::HeaderMap,
    middleware::Next,
    response::Response,
};

pub struct HeaderToken {
    pub unique_id: String,
}

impl HeaderToken {
    pub fn from_headers(headers: &HeaderMap) -> Result<Self, ServiceError> {
        let token = headers
            .get("token")
            .and_then(|v| v.to_str().ok())
            .ok_or(ServiceError::Unauthorized)?;
        Ok(Self { unique_id: token.to_string() })
    }
}

/// Middleware that validates the token and attaches user info
pub async fn auth_middleware(
    headers: HeaderMap,
    mut request: Request,
    next: Next,
) -> Result<Response, ServiceError> {
    let token = HeaderToken::from_headers(&headers)?;

    // Validate token via auth service or cache
    let user = validate_token(&token.unique_id).await?;

    // Attach user to request extensions
    request.extensions_mut().insert(user);

    Ok(next.run(request).await)
}
```

## Docker Compose for Multi-Service Development

### Development Environment

```yaml
services:
  auth-server:
    build: ./nanoservices/auth-server/server
    ports:
      - "8081:8081"
    environment:
      - DATABASE_URL=postgres://user:pass@postgres:5432/auth_db
      - CACHE_API_URL=redis://redis:6379
      - RUST_LOG=info
    depends_on:
      - postgres
      - redis

  app-server:
    build: ./nanoservices/app-server/server
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@postgres:5432/app_db
      - AUTH_SERVICE_URL=http://auth-server:8081
      - CACHE_API_URL=redis://redis:6379
      - RUST_LOG=info
    depends_on:
      - auth-server
      - postgres
      - redis

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

### Integration Testing with Docker Compose

```rust
/// Integration test that requires all services running
#[cfg(test)]
mod integration {
    use reqwest::Client;
    use std::time::Duration;

    async fn wait_for_service(url: &str, timeout: Duration) -> bool {
        let client = Client::new();
        let start = std::time::Instant::now();

        while start.elapsed() < timeout {
            if client.get(url).send().await.is_ok() {
                return true;
            }
            tokio::time::sleep(Duration::from_millis(500)).await;
        }
        false
    }

    #[tokio::test]
    async fn test_full_flow() {
        // Assumes docker-compose up has been run
        let base_url = "http://localhost:8080";
        assert!(
            wait_for_service(&format!("{}/health", base_url), Duration::from_secs(30)).await,
            "Service not ready"
        );

        let client = Client::new();

        // Create user via auth service
        let resp = client
            .post("http://localhost:8081/api/v1/users")
            .json(&serde_json::json!({
                "email": "test@example.com",
                "password": "secure123"
            }))
            .send()
            .await
            .unwrap();
        assert!(resp.status().is_success());

        // Login to get token
        let login_resp: serde_json::Value = client
            .post("http://localhost:8081/api/v1/auth/login")
            .json(&serde_json::json!({
                "email": "test@example.com",
                "password": "secure123"
            }))
            .send()
            .await
            .unwrap()
            .json()
            .await
            .unwrap();

        let token = login_resp["token"].as_str().unwrap();

        // Access protected endpoint on app-server
        let resp = client
            .get(&format!("{}/api/v1/items", base_url))
            .header("token", token)
            .send()
            .await
            .unwrap();
        assert!(resp.status().is_success());
    }
}
```

## Service Discovery

### Client-Side Discovery

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

#[derive(Debug, Clone)]
pub struct ServiceInstance {
    pub id: String,
    pub host: String,
    pub port: u16,
    pub healthy: bool,
}

impl ServiceInstance {
    pub fn url(&self, scheme: &str) -> String {
        format!("{}://{}:{}", scheme, self.host, self.port)
    }
}

pub trait ServiceRegistry: Send + Sync {
    fn get_instances(&self, service_name: &str)
        -> impl Future<Output = Result<Vec<ServiceInstance>, RegistryError>> + Send;
    fn register(&self, service_name: &str, instance: ServiceInstance)
        -> impl Future<Output = Result<(), RegistryError>> + Send;
}

pub struct ServiceDiscoveryClient {
    registry: Arc<dyn ServiceRegistry>,
    counter: AtomicUsize,  // round-robin
}

impl ServiceDiscoveryClient {
    pub async fn discover(&self, service_name: &str) -> Result<ServiceInstance, DiscoveryError> {
        let instances: Vec<_> = self.registry.get_instances(service_name).await?
            .into_iter().filter(|i| i.healthy).collect();
        if instances.is_empty() {
            return Err(DiscoveryError::NoHealthyInstances(service_name.into()));
        }
        let idx = self.counter.fetch_add(1, Ordering::Relaxed) % instances.len();
        Ok(instances[idx].clone())
    }
}
```

### Kubernetes Integration (kube-rs)

```rust
use kube::{Api, Client};
use k8s_openapi::api::core::v1::Endpoints;

pub struct KubernetesServiceDiscovery {
    client: Client,
    namespace: String,
}

impl KubernetesServiceDiscovery {
    pub async fn new(namespace: &str) -> Result<Self, kube::Error> {
        Ok(Self { client: Client::try_default().await?, namespace: namespace.into() })
    }

    pub async fn get_endpoints(&self, service_name: &str) -> Result<Vec<ServiceInstance>, kube::Error> {
        let endpoints: Api<Endpoints> = Api::namespaced(self.client.clone(), &self.namespace);
        let ep = endpoints.get(service_name).await?;

        let mut instances = Vec::new();
        if let Some(subsets) = ep.subsets {
            for subset in subsets {
                let port = subset.ports.as_ref()
                    .and_then(|p| p.first()).map(|p| p.port as u16).unwrap_or(80);
                if let Some(addresses) = subset.addresses {
                    for addr in addresses {
                        instances.push(ServiceInstance {
                            id: addr.target_ref.as_ref()
                                .and_then(|r| r.uid.clone()).unwrap_or_default(),
                            host: addr.ip,
                            port,
                            healthy: true,
                        });
                    }
                }
            }
        }
        Ok(instances)
    }
}
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels: { app: order-service }
  template:
    metadata:
      labels: { app: order-service }
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:latest
        ports: [{ containerPort: 8080 }]
        env:
        - name: DATABASE_URL
          valueFrom: { secretKeyRef: { name: db-credentials, key: url } }
        resources:
          requests: { memory: "64Mi", cpu: "100m" }
          limits: { memory: "256Mi", cpu: "500m" }
        livenessProbe:
          httpGet: { path: /health/live, port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet: { path: /health/ready, port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service Mesh (Istio)

With a service mesh, your Rust code doesn't need discovery logic — the sidecar proxy handles it:

```rust
pub struct ServiceMeshClient { client: reqwest::Client }

impl ServiceMeshClient {
    pub async fn call_service(
        &self, service_name: &str, path: &str,
    ) -> Result<reqwest::Response, reqwest::Error> {
        // Mesh intercepts and routes to correct pod
        self.client.get(format!("http://{}{}", service_name, path)).send().await
    }
}
```

### Load Balancing Strategies

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::collections::HashMap;
use std::sync::Mutex;
use rand::seq::SliceRandom;

pub enum LoadBalancer {
    RoundRobin(AtomicUsize),
    Random,
    LeastConnections(Mutex<HashMap<String, usize>>),
}

impl LoadBalancer {
    pub fn select(&self, instances: &[ServiceInstance]) -> ServiceInstance {
        match self {
            LoadBalancer::RoundRobin(counter) => {
                let idx = counter.fetch_add(1, Ordering::Relaxed) % instances.len();
                instances[idx].clone()
            }
            LoadBalancer::Random => {
                instances.choose(&mut rand::thread_rng())
                    .cloned().unwrap()
            }
            LoadBalancer::LeastConnections(connections) => {
                let conns = connections.lock().unwrap();
                instances.iter()
                    .min_by_key(|i| conns.get(&i.id).unwrap_or(&0))
                    .cloned().unwrap()
            }
        }
    }
}
```

### Discovery-Aware HTTP Client

```rust
pub struct DiscoveryAwareClient {
    discovery: Arc<ServiceDiscoveryClient>,
    http_client: reqwest::Client,
}

impl DiscoveryAwareClient {
    pub async fn get(&self, service_name: &str, path: &str) -> Result<reqwest::Response, ClientError> {
        let instance = self.discovery.discover(service_name).await?;
        let url = format!("{}{}", instance.url("http"), path);
        self.http_client.get(&url).send().await.map_err(ClientError::Http)
    }

    pub async fn post<T: serde::Serialize>(
        &self, service_name: &str, path: &str, body: &T,
    ) -> Result<reqwest::Response, ClientError> {
        let instance = self.discovery.discover(service_name).await?;
        let url = format!("{}{}", instance.url("http"), path);
        self.http_client.post(&url).json(body).send().await.map_err(ClientError::Http)
    }
}
```

### Server-Side Discovery with Heartbeat

```rust
pub struct ServiceRegistrar {
    registry: Arc<dyn ServiceRegistry>,
    service_name: String,
    instance: ServiceInstance,
}

impl ServiceRegistrar {
    pub async fn register(&self) -> Result<(), RegistryError> {
        self.registry.register(&self.service_name, self.instance.clone()).await
    }

    pub async fn deregister(&self) -> Result<(), RegistryError> {
        self.registry.deregister(&self.service_name, &self.instance.id).await
    }

    /// Start heartbeat to maintain registration
    pub fn start_heartbeat(self: Arc<Self>, interval: Duration) -> tokio::task::JoinHandle<()> {
        tokio::spawn(async move {
            let mut ticker = tokio::time::interval(interval);
            loop {
                ticker.tick().await;
                if let Err(e) = self.register().await {
                    eprintln!("Heartbeat failed: {}", e);
                }
            }
        })
    }
}

// Graceful shutdown with deregistration
async fn run_server(registrar: Arc<ServiceRegistrar>) {
    registrar.register().await.expect("Failed to register");
    let heartbeat = registrar.clone().start_heartbeat(Duration::from_secs(10));

    let server = HttpServer::new(|| App::new())
        .bind("0.0.0.0:8080").unwrap().run();

    tokio::select! {
        _ = server => {},
        _ = tokio::signal::ctrl_c() => {
            println!("Shutting down...");
        }
    }

    heartbeat.abort();
    registrar.deregister().await.ok();
}
```

### Consul Integration

```rust
use reqwest::Client;

pub struct ConsulRegistry {
    client: Client,
    consul_url: String,
}

impl ConsulRegistry {
    pub fn new(consul_url: &str) -> Self {
        Self { client: Client::new(), consul_url: consul_url.to_string() }
    }
}

impl ServiceRegistry for ConsulRegistry {
    async fn get_instances(&self, service_name: &str) -> Result<Vec<ServiceInstance>, RegistryError> {
        let url = format!(
            "{}/v1/health/service/{}?passing=true",
            self.consul_url, service_name
        );

        let response: Vec<ConsulServiceEntry> = self.client
            .get(&url).send().await
            .map_err(|e| RegistryError::Network(e.to_string()))?
            .json().await
            .map_err(|e| RegistryError::Parse(e.to_string()))?;

        Ok(response.into_iter().map(|entry| ServiceInstance {
            id: entry.service.id,
            host: entry.service.address,
            port: entry.service.port,
            healthy: true,
        }).collect())
    }

    async fn register(&self, service_name: &str, instance: ServiceInstance) -> Result<(), RegistryError> {
        let url = format!("{}/v1/agent/service/register", self.consul_url);

        let registration = serde_json::json!({
            "ID": instance.id,
            "Name": service_name,
            "Address": instance.host,
            "Port": instance.port,
            "Check": {
                "HTTP": format!("http://{}:{}/health", instance.host, instance.port),
                "Interval": "10s",
                "Timeout": "5s"
            }
        });

        self.client.put(&url).json(&registration).send().await
            .map_err(|e| RegistryError::Network(e.to_string()))?;
        Ok(())
    }
}

#[derive(Debug, serde::Deserialize)]
struct ConsulServiceEntry {
    #[serde(rename = "Service")]
    service: ConsulService,
}

#[derive(Debug, serde::Deserialize)]
struct ConsulService {
    #[serde(rename = "ID")]
    id: String,
    #[serde(rename = "Address")]
    address: String,
    #[serde(rename = "Port")]
    port: u16,
}
```

### Health Check Endpoints

```rust
use actix_web::{get, HttpResponse, web};
use std::sync::atomic::{AtomicBool, Ordering};

#[derive(Debug, Clone)]
pub struct HealthState {
    ready: Arc<AtomicBool>,
    db_pool: PgPool,
}

#[get("/health/live")]
async fn liveness() -> HttpResponse {
    // Simple liveness — is the process running?
    HttpResponse::Ok().json(serde_json::json!({"status": "alive"}))
}

#[get("/health/ready")]
async fn readiness(state: web::Data<HealthState>) -> HttpResponse {
    if !state.ready.load(Ordering::Relaxed) {
        return HttpResponse::ServiceUnavailable()
            .json(serde_json::json!({"status": "not ready", "reason": "initializing"}));
    }

    // Check database connection
    match sqlx::query("SELECT 1").execute(&state.db_pool).await {
        Ok(_) => HttpResponse::Ok().json(serde_json::json!({
            "status": "ready",
            "checks": { "database": "ok" }
        })),
        Err(e) => HttpResponse::ServiceUnavailable().json(serde_json::json!({
            "status": "not ready",
            "reason": "database unavailable",
            "error": e.to_string()
        })),
    }
}

// Mark as ready after initialization
async fn startup(state: web::Data<HealthState>) {
    // Perform initialization...
    state.ready.store(true, Ordering::Relaxed);
}
```

### Watching Kubernetes Endpoints

```rust
use kube::api::ListParams;
use futures::StreamExt;

/// Cached discovery that watches for changes
pub struct CachedServiceDiscovery {
    k8s: Arc<KubernetesServiceDiscovery>,
    cache: Arc<tokio::sync::RwLock<HashMap<String, Vec<ServiceInstance>>>>,
}

impl CachedServiceDiscovery {
    pub async fn start_watcher(self: Arc<Self>, service_name: &str) {
        let endpoints: Api<Endpoints> = Api::namespaced(
            self.k8s.client.clone(), &self.k8s.namespace
        );
        let params = ListParams::default()
            .fields(&format!("metadata.name={}", service_name));

        let mut stream = endpoints.watch(&params, "0").await.unwrap();

        while let Some(event) = stream.next().await {
            match event {
                Ok(kube::api::WatchEvent::Added(ep))
                | Ok(kube::api::WatchEvent::Modified(ep)) => {
                    let instances = self.k8s.parse_endpoints(&ep);
                    self.cache.write().await
                        .insert(service_name.to_string(), instances);
                }
                Ok(kube::api::WatchEvent::Deleted(_)) => {
                    self.cache.write().await.remove(service_name);
                }
                Err(e) => eprintln!("Watch error: {}", e),
                _ => {}
            }
        }
    }

    pub async fn get_instances(&self, service_name: &str) -> Vec<ServiceInstance> {
        self.cache.read().await
            .get(service_name).cloned().unwrap_or_default()
    }
}
```

### Environment Variables from Kubernetes

```rust
/// Configuration from Kubernetes environment
#[derive(Debug, Clone)]
pub struct KubernetesConfig {
    pub database_url: String,
    pub redis_host: String,
    pub redis_port: u16,
    pub service_name: String,
}

impl KubernetesConfig {
    pub fn from_env() -> Result<Self, ConfigError> {
        Ok(Self {
            // From Secret or ConfigMap
            database_url: std::env::var("DATABASE_URL")
                .map_err(|_| ConfigError::Missing("DATABASE_URL"))?,

            // Kubernetes injects service discovery env vars
            // Format: {SERVICE_NAME}_SERVICE_HOST and {SERVICE_NAME}_SERVICE_PORT
            redis_host: std::env::var("REDIS_SERVICE_HOST")
                .unwrap_or_else(|_| "redis".to_string()),
            redis_port: std::env::var("REDIS_SERVICE_PORT")
                .ok().and_then(|p| p.parse().ok()).unwrap_or(6379),

            // Pod metadata
            service_name: std::env::var("HOSTNAME")
                .unwrap_or_else(|_| "unknown".to_string()),
        })
    }
}
```

### Istio Traffic Management

```yaml
# VirtualService for canary releases
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: order-service
        subset: canary
  - route:
    - destination:
        host: order-service
        subset: stable
---
# DestinationRule for load balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary
```

### Circuit Breaker with Discovery

```rust
pub struct ResilientServiceClient {
    discovery: Arc<ServiceDiscoveryClient>,
    circuit_breakers: Arc<tokio::sync::RwLock<HashMap<String, CircuitBreaker>>>,
    http_client: reqwest::Client,
}

impl ResilientServiceClient {
    pub async fn call(
        &self, service_name: &str, path: &str,
    ) -> Result<reqwest::Response, ClientError> {
        let instance = self.discovery.discover(service_name).await?;
        let instance_key = instance.id.clone();

        // Check circuit breaker for this specific instance
        let breakers = self.circuit_breakers.read().await;
        if let Some(breaker) = breakers.get(&instance_key) {
            if !breaker.allow_request() {
                return Err(ClientError::CircuitOpen(instance_key));
            }
        }
        drop(breakers);

        // Make request
        let url = format!("{}{}", instance.url("http"), path);
        let result = self.http_client.get(&url).send().await;

        // Update circuit breaker
        let mut breakers = self.circuit_breakers.write().await;
        let breaker = breakers.entry(instance_key)
            .or_insert_with(|| CircuitBreaker::new(5, Duration::from_secs(30)));

        match &result {
            Ok(response) if response.status().is_success() => breaker.record_success(),
            _ => breaker.record_failure(),
        }

        result.map_err(ClientError::Http)
    }
}
```

### Discovery Pattern Summary

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| **DNS-based (K8s)** | Simple K8s deployments | Low |
| **Client-side** | Fine-grained control, custom LB | Medium |
| **Server-side** | External load balancer, legacy systems | Medium |
| **Service Mesh** | Advanced traffic management, mTLS | High |
| **Consul/etcd** | Multi-cloud, hybrid environments | Medium |

## Redis Caching

### Setup

```toml
[dependencies]
redis = { version = "0.25", features = ["tokio-comp"] }
```

```rust
use redis::aio::MultiplexedConnection;
use std::sync::LazyLock;

pub static REDIS_CLIENT: LazyLock<redis::Client> = LazyLock::new(|| {
    let url = std::env::var("REDIS_URL").unwrap_or_else(|_| "redis://127.0.0.1:6379".into());
    redis::Client::open(url).expect("Failed to create Redis client")
});

pub async fn get_redis_connection() -> Result<MultiplexedConnection, CacheError> {
    REDIS_CLIENT.get_multiplexed_async_connection().await
        .map_err(|e| CacheError::Connection(e.to_string()))
}
```

### Generic Cache Operations

```rust
use redis::AsyncCommands;
use serde::{Serialize, de::DeserializeOwned};

pub async fn cache_get<T: DeserializeOwned>(
    con: &mut MultiplexedConnection, key: &str,
) -> Result<Option<T>, CacheError> {
    let result: Option<String> = con.get(key).await
        .map_err(|e| CacheError::Command(e.to_string()))?;
    match result {
        Some(json) => Ok(Some(serde_json::from_str(&json)
            .map_err(|e| CacheError::Command(e.to_string()))?)),
        None => Ok(None),
    }
}

pub async fn cache_set<T: Serialize>(
    con: &mut MultiplexedConnection, key: &str, value: &T, ttl_seconds: u64,
) -> Result<(), CacheError> {
    let json = serde_json::to_string(value)
        .map_err(|e| CacheError::Command(e.to_string()))?;
    con.set_ex(key, json, ttl_seconds).await
        .map_err(|e| CacheError::Command(e.to_string()))
}
```

### Cache-Aside Pattern

```rust
pub async fn get_user_with_cache(
    user_id: i32, cache: &mut MultiplexedConnection, db: &PgPool,
) -> Result<User, AppError> {
    let cache_key = format!("user:{}", user_id);

    // Try cache first
    if let Some(user) = cache_get::<User>(cache, &cache_key).await? {
        return Ok(user);
    }

    // Cache miss — fetch from database
    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(user_id).fetch_optional(db).await?.ok_or(AppError::NotFound)?;

    // Store in cache (TTL: 5 minutes)
    cache_set(cache, &cache_key, &user, 300).await?;
    Ok(user)
}
```

## Redis Job Queues

### Queue Architecture

```
┌────────┐    ┌────────┐    ┌──────────────────┐
│ Client │───>│ Server │───>│  Redis Queue      │
└────────┘    └────────┘    │  (LPUSH/BLPOP)    │
               (lpush)      └────────┬───────────┘
                                     │
                            ┌────────┴────────┐
                            │    Workers       │
                            │  (blpop loop)    │
                            └─────────────────┘
```

### Enqueue Tasks

```rust
use redis::AsyncCommands;
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct TaskPayload {
    pub task_type: String,
    pub data: serde_json::Value,
}

pub async fn enqueue_task(
    con: &mut MultiplexedConnection, queue: &str, payload: &TaskPayload,
) -> Result<(), CacheError> {
    let json = serde_json::to_string(payload)
        .map_err(|e| CacheError::Command(e.to_string()))?;
    con.lpush::<_, _, ()>(queue, json).await
        .map_err(|e| CacheError::Command(e.to_string()))
}
```

### Worker (Blocking Pop)

```rust
async fn worker(client: redis::Client) {
    let mut con = client.get_async_connection().await.unwrap();

    loop {
        match con.blpop::<&str, Option<(String, String)>>("task_queue", 0.0).await {
            Ok(Some((_queue, json))) => {
                if let Ok(payload) = serde_json::from_str::<TaskPayload>(&json) {
                    match payload.task_type.as_str() {
                        "email" => send_email(&payload.data).await,
                        "report" => generate_report(&payload.data).await,
                        _ => eprintln!("Unknown task: {}", payload.task_type),
                    }
                }
            }
            Ok(None) => {}
            Err(e) => {
                eprintln!("Queue error: {}", e);
                tokio::time::sleep(Duration::from_secs(1)).await;
            }
        }
    }
}

// Spawn multiple workers
#[tokio::main]
async fn main() {
    let client = redis::Client::open("redis://127.0.0.1/").unwrap();
    let mut handles = Vec::new();
    for _ in 0..4 {
        let c = client.clone();
        handles.push(tokio::spawn(async move { worker(c).await }));
    }
    handles.pop().unwrap().await.unwrap();
}
```

### Axum Endpoint for Queue Submission

```rust
use axum::{extract::State, routing::post, Json, Router};

async fn submit_task(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<TaskPayload>,
) -> Result<&'static str, (StatusCode, String)> {
    let mut con = state.redis_client.get_multiplexed_async_connection().await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
    let json = serde_json::to_string(&payload)
        .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?;
    con.lpush::<_, _, ()>("task_queue", json).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
    Ok("Task submitted")
}
```

## Redis User Session Caching

### Session Structure

```rust
use chrono::{DateTime, Utc};

pub struct UserSession {
    pub user_id: String,
    pub key: String,
    pub session_datetime: DateTime<Utc>,
}

impl UserSession {
    pub fn from_id(user_id: String) -> Self {
        Self {
            key: format!("user_session_{}", user_id),
            user_id,
            session_datetime: Utc::now(),
        }
    }
}

#[derive(Debug)]
pub enum UserSessionStatus {
    Ok(i32),      // Contains permanent user ID
    Refresh,      // Token needs refresh
}
```

### Session Operations

```rust
use redis::aio::MultiplexedConnection;

/// Create a new user session in cache
pub async fn session_login(
    con: &mut MultiplexedConnection,
    user_id: &str,
    timeout_mins: usize,
    perm_user_id: i32,
) -> Result<(), CacheError> {
    let key = format!("user_session_{}", user_id);

    redis::cmd("HSET")
        .arg(&key)
        .arg("last_interacted")
        .arg(Utc::now().format("%Y-%m-%d %H:%M:%S").to_string())
        .arg("timeout_mins").arg(timeout_mins)
        .arg("counter").arg(0)
        .arg("perm_user_id").arg(perm_user_id)
        .query_async(con).await
        .map_err(|e| CacheError::Command(e.to_string()))
}

/// Update session and check for timeout/refresh
pub async fn session_update(
    con: &mut MultiplexedConnection,
    user_id: &str,
) -> Result<UserSessionStatus, CacheError> {
    let key = format!("user_session_{}", user_id);

    let exists: bool = redis::cmd("EXISTS")
        .arg(&key).query_async(con).await
        .map_err(|e| CacheError::Command(e.to_string()))?;

    if !exists { return Err(CacheError::NotFound); }

    let (last_interacted, timeout_mins, counter, perm_user_id): (String, i64, i64, i32) =
        redis::cmd("HMGET")
            .arg(&key)
            .arg("last_interacted").arg("timeout_mins")
            .arg("counter").arg("perm_user_id")
            .query_async(con).await
            .map_err(|e| CacheError::Command(e.to_string()))?;

    let last_time = chrono::NaiveDateTime::parse_from_str(
        &last_interacted, "%Y-%m-%d %H:%M:%S"
    ).map_err(|_| CacheError::Command("Invalid datetime format".into()))?;

    let now = Utc::now().naive_utc();
    let elapsed_mins = now.signed_duration_since(last_time).num_minutes();

    if elapsed_mins > timeout_mins {
        redis::cmd("DEL").arg(&key).query_async::<_, ()>(con).await
            .map_err(|e| CacheError::Command(e.to_string()))?;
        return Err(CacheError::Timeout);
    }

    let new_counter = counter + 1;
    redis::cmd("HSET")
        .arg(&key)
        .arg("last_interacted").arg(now.format("%Y-%m-%d %H:%M:%S").to_string())
        .arg("counter").arg(new_counter)
        .query_async::<_, ()>(con).await
        .map_err(|e| CacheError::Command(e.to_string()))?;

    if new_counter > 20 {
        return Ok(UserSessionStatus::Refresh);
    }

    Ok(UserSessionStatus::Ok(perm_user_id))
}

/// Remove session (logout)
pub async fn session_logout(
    con: &mut MultiplexedConnection, user_id: &str,
) -> Result<bool, CacheError> {
    let key = format!("user_session_{}", user_id);
    let deleted: i32 = redis::cmd("DEL").arg(&key)
        .query_async(con).await
        .map_err(|e| CacheError::Command(e.to_string()))?;
    Ok(deleted > 0)
}
```

### Session Cache Kernel (Descriptor Pattern)

```rust
/// Descriptor for different cache implementations
pub struct RedisSessionDescriptor;

pub trait GetUserSession {
    fn get_user_session(unique_id: String)
        -> impl std::future::Future<Output = Result<CachedSession, CacheError>> + Send;
}

#[derive(Clone)]
pub struct CachedSession {
    pub user_id: i32,
}

impl GetUserSession for RedisSessionDescriptor {
    async fn get_user_session(unique_id: String) -> Result<CachedSession, CacheError> {
        let address = std::env::var("CACHE_API_URL")
            .map_err(|_| CacheError::Connection("CACHE_API_URL not set".into()))?;
        let client = redis::Client::open(address)
            .map_err(|e| CacheError::Connection(e.to_string()))?;
        let mut con = client.get_multiplexed_async_connection().await
            .map_err(|e| CacheError::Connection(e.to_string()))?;

        match session_update(&mut con, &unique_id).await? {
            UserSessionStatus::Ok(user_id) => Ok(CachedSession { user_id }),
            UserSessionStatus::Refresh => {
                // Re-authenticate and create new session
                let user = get_user_by_unique_id(unique_id.clone()).await?;
                session_login(&mut con, &unique_id, 20, user.id).await?;
                Ok(CachedSession { user_id: user.id })
            }
        }
    }
}

/// Using session descriptor in web handlers
pub async fn protected_endpoint<T, X>(
    token: HeaderToken,
    body: axum::Json<serde_json::Value>,
) -> Result<axum::response::Response, ServiceError>
where
    T: ItemRepository,
    X: GetUserSession,
{
    let session = X::get_user_session(token.unique_id).await
        .map_err(|_| ServiceError::Unauthorized)?;

    let items = T::get_all(session.user_id).await
        .map_err(|e| ServiceError::Other(e.to_string()))?;

    Ok(axum::Json(items).into_response())
}
```

## Custom Redis Modules in Rust

Build custom Redis commands by creating a C dynamic library:

### Module Setup

```toml
[package]
name = "cache-module"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]  # C dynamic library for Redis

[dependencies]
redis-module = "2.0"
chrono = "0.4"
serde = { version = "1.0", features = ["derive"] }
```

### Module Definition

```rust
use redis_module::redis_module;

redis_module! {
    name: "user_sessions",
    version: 1,
    allocator: (
        redis_module::alloc::RedisAlloc,
        redis_module::alloc::RedisAlloc
    ),
    data_types: [],
    commands: [
        ["login.set", login, "write fast deny-oom", 1, 1, 1],
        ["logout.set", logout, "write fast deny-oom", 1, 1, 1],
        ["update.set", update, "write fast deny-oom", 1, 1, 1],
    ]
}
```

### Command Implementation

```rust
use redis_module::{Context, NextArg, RedisError, RedisResult, RedisString, RedisValue};

pub fn login(ctx: &Context, args: Vec<RedisString>) -> RedisResult {
    if args.len() < 4 { return Err(RedisError::WrongArity); }

    let mut args = args.into_iter().skip(1);
    let user_id = args.next_arg()?.to_string();
    let timeout_mins = args.next_arg()?;
    let perm_user_id = args.next_arg()?.to_string();

    let key_string = RedisString::create(None, format!("user_session_{}", user_id));
    let key = ctx.open_key_writable(&key_string);

    let now = chrono::Utc::now().format("%Y-%m-%d %H:%M:%S").to_string();
    key.hash_set("last_interacted", ctx.create_string(now));
    key.hash_set("timeout_mins", ctx.create_string(timeout_mins));
    key.hash_set("counter", ctx.create_string("0"));
    key.hash_set("perm_user_id", ctx.create_string(perm_user_id));

    Ok(RedisValue::SimpleStringStatic("OK"))
}

pub fn update(ctx: &Context, args: Vec<RedisString>) -> RedisResult {
    if args.len() < 2 { return Err(RedisError::WrongArity); }

    let mut args = args.into_iter().skip(1);
    let user_id = args.next_arg()?.to_string();

    let key_string = RedisString::create(None, format!("user_session_{}", user_id));
    let key = ctx.open_key_writable(&key_string);

    if key.is_empty() {
        return Ok(RedisValue::SimpleStringStatic("NOT_FOUND"));
    }

    // Check timeout
    let last_str = key.hash_get("last_interacted")?
        .ok_or(RedisError::Str("Missing last_interacted"))?.to_string();
    let timeout_mins: i64 = key.hash_get("timeout_mins")?
        .ok_or(RedisError::Str("Missing timeout"))?.to_string()
        .parse().map_err(|_| RedisError::Str("Invalid timeout"))?;

    let last_time = chrono::NaiveDateTime::parse_from_str(&last_str, "%Y-%m-%d %H:%M:%S")
        .map_err(|_| RedisError::Str("Invalid datetime"))?;
    let now = chrono::Utc::now().naive_utc();
    let elapsed = now.signed_duration_since(last_time).num_minutes();

    if elapsed > timeout_mins {
        key.delete()?;
        return Ok(RedisValue::SimpleStringStatic("TIMEOUT"));
    }

    // Increment counter
    let counter: i64 = key.hash_get("counter")?
        .ok_or(RedisError::Str("Missing counter"))?.to_string()
        .parse().map_err(|_| RedisError::Str("Invalid counter"))?;
    let new_counter = counter + 1;
    key.hash_set("counter", ctx.create_string(new_counter.to_string()));

    // Update timestamp
    key.hash_set("last_interacted", ctx.create_string(
        now.format("%Y-%m-%d %H:%M:%S").to_string()
    ));

    if new_counter > 20 {
        return Ok(RedisValue::SimpleStringStatic("REFRESH"));
    }

    let perm_user_id = key.hash_get("perm_user_id")?
        .ok_or(RedisError::Str("Missing perm_user_id"))?;
    Ok(RedisValue::SimpleString(perm_user_id.to_string()))
}
```

### Docker for Redis Module

```dockerfile
FROM rust:latest as build
RUN apt-get update && apt-get install libclang-dev -y
WORKDIR /app
COPY . .
RUN cargo build --release

FROM redis:latest
COPY --from=build /app/target/release/libcache_module.so ./libcache_module.so
EXPOSE 6379
CMD ["redis-server", "--loadmodule", "./libcache_module.so"]
```

## Re-Queue Pattern for Continuation Tasks

For tasks that need multiple processing rounds (coroutine-style):

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct ContinuableTask {
    pub id: String,
    pub state: serde_json::Value,
    pub iteration: u32,
    pub max_iterations: u32,
}

async fn worker_with_requeue(client: redis::Client) {
    let mut con = client.get_async_connection().await.unwrap();

    loop {
        if let Ok(Some((_, json))) = con.blpop::<&str, Option<(String, String)>>(
            "continuable_queue", 0.0
        ).await {
            if let Ok(mut task) = serde_json::from_str::<ContinuableTask>(&json) {
                task.iteration += 1;
                process_iteration(&mut task).await;

                if task.iteration < task.max_iterations {
                    // Not done — re-queue with updated state
                    let updated = serde_json::to_string(&task).unwrap();
                    con.rpush::<_, _, ()>("continuable_queue", updated).await.unwrap();
                    println!("Task {} re-queued ({}/{})",
                        task.id, task.iteration, task.max_iterations);
                } else {
                    println!("Task {} completed", task.id);
                }
            }
        }
    }
}
```

## Resilience Patterns

### Idempotency

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use uuid::Uuid;

pub struct IdempotentProcessor<T: Clone> {
    processed: Arc<Mutex<HashMap<Uuid, T>>>,
}

impl<T: Clone> IdempotentProcessor<T> {
    pub fn execute<F>(&self, key: Uuid, operation: F) -> T
    where F: FnOnce() -> T {
        let mut cache = self.processed.lock().unwrap();
        if let Some(result) = cache.get(&key) { return result.clone(); }
        let result = operation();
        cache.insert(key, result.clone());
        result
    }
}
```

### Client-Side Retry with Exponential Backoff

```rust
pub async fn create_order_with_retry(
    client: &reqwest::Client, base_url: &str,
    request: &CreateOrderRequest, max_retries: u32,
) -> Result<OrderResponse, OrderError> {
    let mut attempt = 0;
    let mut backoff = Duration::from_millis(100);

    loop {
        let response = client.post(format!("{}/orders", base_url))
            .json(request).send().await;

        match response {
            Ok(resp) if resp.status().is_success() =>
                return resp.json().await.map_err(OrderError::from),
            Ok(resp) if resp.status() == reqwest::StatusCode::CONFLICT =>
                return resp.json().await.map_err(OrderError::from),  // Already created
            Ok(resp) if resp.status().is_server_error() => {
                tracing::warn!(status = %resp.status(), attempt, "Server error, retrying");
            }
            Ok(resp) => {
                let status = resp.status().as_u16();
                let msg = resp.text().await.unwrap_or_default();
                return Err(OrderError::Server { status, message: msg });
            }
            Err(e) if e.is_timeout() || e.is_connect() => {
                tracing::warn!(error = %e, attempt, "Network error, retrying");
            }
            Err(e) => return Err(OrderError::Network(e)),
        }

        attempt += 1;
        if attempt >= max_retries { return Err(OrderError::MaxRetriesExceeded); }

        let jitter = rand::random::<f64>() * 0.5;
        tokio::time::sleep(backoff.mul_f64(1.0 + jitter)).await;
        backoff = backoff.saturating_mul(2).min(Duration::from_secs(30));
    }
}
```

### Server-Side Idempotency Middleware

```rust
use axum::{
    extract::{Request, State},
    http::{HeaderMap, StatusCode},
    middleware::Next,
    response::{IntoResponse, Response},
    Json,
};
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use uuid::Uuid;

#[derive(Clone)]
pub struct IdempotencyStore {
    responses: Arc<RwLock<HashMap<Uuid, (StatusCode, serde_json::Value)>>>,
}

impl IdempotencyStore {
    pub fn new() -> Self {
        Self { responses: Arc::new(RwLock::new(HashMap::new())) }
    }

    pub fn get(&self, key: &Uuid) -> Option<(StatusCode, serde_json::Value)> {
        self.responses.read().unwrap().get(key).cloned()
    }

    pub fn set(&self, key: Uuid, status: StatusCode, body: serde_json::Value) {
        self.responses.write().unwrap().insert(key, (status, body));
    }
}

const IDEMPOTENCY_KEY_HEADER: &str = "Idempotency-Key";

pub async fn idempotency_middleware(
    State(store): State<IdempotencyStore>,
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Response {
    let idempotency_key = match headers
        .get(IDEMPOTENCY_KEY_HEADER)
        .and_then(|v| v.to_str().ok())
        .and_then(|s| Uuid::parse_str(s).ok())
    {
        Some(key) => key,
        None => return next.run(request).await, // No key — proceed normally
    };

    // Check for cached response
    if let Some((status, body)) = store.get(&idempotency_key) {
        tracing::debug!(idempotency_key = %idempotency_key, "Returning cached response");
        return (status, Json(body)).into_response();
    }

    let response = next.run(request).await;

    // Cache successful responses
    let status = response.status();
    if status.is_success() {
        store.set(idempotency_key, status, serde_json::json!({"cached": true}));
    }

    response
}
```

### Circuit Breaker

```rust
use std::sync::atomic::{AtomicU32, Ordering};
use std::sync::RwLock;
use std::time::{Duration, Instant};

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum CircuitState { Closed, Open, HalfOpen }

#[derive(Debug, Clone)]
pub struct CircuitBreakerConfig {
    pub failure_threshold: u32,
    pub reset_timeout: Duration,
    pub success_threshold: u32,
}

impl Default for CircuitBreakerConfig {
    fn default() -> Self {
        Self {
            failure_threshold: 5,
            reset_timeout: Duration::from_secs(30),
            success_threshold: 3,
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum CircuitError<E> {
    #[error("Circuit is open — service unavailable")]
    CircuitOpen,
    #[error("Operation failed: {0}")]
    OperationFailed(#[source] E),
}

pub struct CircuitBreaker {
    config: CircuitBreakerConfig,
    state: RwLock<CircuitState>,
    failure_count: AtomicU32,
    success_count: AtomicU32,
    last_failure_time: RwLock<Option<Instant>>,
}

impl CircuitBreaker {
    pub fn new(config: CircuitBreakerConfig) -> Self {
        Self {
            config,
            state: RwLock::new(CircuitState::Closed),
            failure_count: AtomicU32::new(0),
            success_count: AtomicU32::new(0),
            last_failure_time: RwLock::new(None),
        }
    }

    /// Execute an operation through the circuit breaker
    pub fn call<F, T, E>(&self, operation: F) -> Result<T, CircuitError<E>>
    where F: FnOnce() -> Result<T, E> {
        if !self.should_allow_request() {
            return Err(CircuitError::CircuitOpen);
        }
        match operation() {
            Ok(result) => { self.on_success(); Ok(result) }
            Err(e) => { self.on_failure(); Err(CircuitError::OperationFailed(e)) }
        }
    }

    /// Async version
    pub async fn call_async<F, Fut, T, E>(&self, operation: F) -> Result<T, CircuitError<E>>
    where
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = Result<T, E>>,
    {
        if !self.should_allow_request() {
            return Err(CircuitError::CircuitOpen);
        }
        match operation().await {
            Ok(result) => { self.on_success(); Ok(result) }
            Err(e) => { self.on_failure(); Err(CircuitError::OperationFailed(e)) }
        }
    }

    fn should_allow_request(&self) -> bool {
        match *self.state.read().unwrap() {
            CircuitState::Closed => true,
            CircuitState::Open => {
                if let Some(last) = *self.last_failure_time.read().unwrap() {
                    if last.elapsed() >= self.config.reset_timeout {
                        self.transition_to(CircuitState::HalfOpen);
                        return true;
                    }
                }
                false
            }
            CircuitState::HalfOpen => true,
        }
    }

    fn on_success(&self) {
        match *self.state.read().unwrap() {
            CircuitState::HalfOpen => {
                let successes = self.success_count.fetch_add(1, Ordering::SeqCst) + 1;
                if successes >= self.config.success_threshold {
                    self.transition_to(CircuitState::Closed);
                    tracing::info!("Circuit breaker closed — service recovered");
                }
            }
            CircuitState::Closed => {
                self.failure_count.store(0, Ordering::SeqCst);
            }
            _ => {}
        }
    }

    fn on_failure(&self) {
        *self.last_failure_time.write().unwrap() = Some(Instant::now());
        match *self.state.read().unwrap() {
            CircuitState::Closed => {
                let failures = self.failure_count.fetch_add(1, Ordering::SeqCst) + 1;
                if failures >= self.config.failure_threshold {
                    self.transition_to(CircuitState::Open);
                    tracing::warn!(failures, "Circuit breaker opened");
                }
            }
            CircuitState::HalfOpen => {
                self.transition_to(CircuitState::Open);
                tracing::warn!("Circuit breaker reopened — recovery failed");
            }
            _ => {}
        }
    }

    fn transition_to(&self, new_state: CircuitState) {
        *self.state.write().unwrap() = new_state;
        self.failure_count.store(0, Ordering::SeqCst);
        self.success_count.store(0, Ordering::SeqCst);
    }

    pub fn state(&self) -> CircuitState { *self.state.read().unwrap() }
    pub fn reset(&self) {
        self.transition_to(CircuitState::Closed);
        *self.last_failure_time.write().unwrap() = None;
    }
}
```

### Using Circuit Breaker with Services

```rust
pub struct PaymentService {
    circuit_breaker: Arc<CircuitBreaker>,
    http_client: reqwest::Client,
    base_url: String,
}

impl PaymentService {
    pub fn new(base_url: String) -> Self {
        Self {
            circuit_breaker: Arc::new(CircuitBreaker::new(CircuitBreakerConfig {
                failure_threshold: 3,
                reset_timeout: Duration::from_secs(60),
                success_threshold: 2,
            })),
            http_client: reqwest::Client::new(),
            base_url,
        }
    }

    pub async fn process_payment(&self, amount: f64) -> Result<PaymentResult, PaymentError> {
        self.circuit_breaker
            .call_async(|| async {
                let response = self.http_client
                    .post(&format!("{}/payments", self.base_url))
                    .json(&serde_json::json!({ "amount": amount }))
                    .timeout(Duration::from_secs(5))
                    .send().await.map_err(PaymentError::Network)?;

                if !response.status().is_success() {
                    return Err(PaymentError::ServiceError(response.status().as_u16()));
                }
                response.json().await.map_err(PaymentError::Parse)
            })
            .await
            .map_err(|e| match e {
                CircuitError::CircuitOpen => PaymentError::ServiceUnavailable,
                CircuitError::OperationFailed(inner) => inner,
            })
    }
}
```

### Circuit Breaker with Fallback

```rust
pub struct ResilientUserService {
    primary: UserServiceClient,
    fallback_cache: Arc<RwLock<HashMap<u64, User>>>,
    circuit_breaker: Arc<CircuitBreaker>,
}

impl ResilientUserService {
    pub async fn get_user(&self, id: u64) -> Result<User, UserError> {
        let result = self.circuit_breaker.call_async(|| async {
            self.primary.get_user(id).await
        }).await;

        match result {
            Ok(user) => {
                // Cache successful result for fallback
                self.fallback_cache.write().unwrap().insert(id, user.clone());
                Ok(user)
            }
            Err(CircuitError::CircuitOpen) | Err(CircuitError::OperationFailed(_)) => {
                tracing::warn!(user_id = id, "Using cached user data");
                self.fallback_cache.read().unwrap()
                    .get(&id).cloned()
                    .ok_or(UserError::NotFound)
            }
        }
    }
}
```

### Monitoring Circuit Breaker with Prometheus

```rust
use prometheus::{IntGaugeVec, IntCounterVec, Registry};

pub struct MonitoredCircuitBreaker {
    inner: CircuitBreaker,
    name: String,
    state_gauge: IntGaugeVec,
    call_counter: IntCounterVec,
}

impl MonitoredCircuitBreaker {
    pub fn new(name: &str, config: CircuitBreakerConfig, registry: &Registry) -> Self {
        let state_gauge = IntGaugeVec::new(
            prometheus::Opts::new("circuit_breaker_state", "Current state")
                .const_label("circuit", name),
            &["state"],
        ).unwrap();

        let call_counter = IntCounterVec::new(
            prometheus::Opts::new("circuit_breaker_calls", "Call outcomes")
                .const_label("circuit", name),
            &["outcome"],
        ).unwrap();

        registry.register(Box::new(state_gauge.clone())).unwrap();
        registry.register(Box::new(call_counter.clone())).unwrap();

        Self { inner: CircuitBreaker::new(config), name: name.into(), state_gauge, call_counter }
    }

    pub fn call<F, T, E>(&self, operation: F) -> Result<T, CircuitError<E>>
    where F: FnOnce() -> Result<T, E> {
        let result = self.inner.call(operation);

        let state = self.inner.state();
        self.state_gauge.with_label_values(&["closed"])
            .set(if state == CircuitState::Closed { 1 } else { 0 });
        self.state_gauge.with_label_values(&["open"])
            .set(if state == CircuitState::Open { 1 } else { 0 });
        self.state_gauge.with_label_values(&["half_open"])
            .set(if state == CircuitState::HalfOpen { 1 } else { 0 });

        match &result {
            Ok(_) => self.call_counter.with_label_values(&["success"]).inc(),
            Err(CircuitError::CircuitOpen) => self.call_counter.with_label_values(&["rejected"]).inc(),
            Err(CircuitError::OperationFailed(_)) => self.call_counter.with_label_values(&["failure"]).inc(),
        }

        result
    }
}
```

### Circuit Breaker Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn circuit_opens_after_threshold_failures() {
        let config = CircuitBreakerConfig {
            failure_threshold: 3, reset_timeout: Duration::from_secs(30),
            success_threshold: 2,
        };
        let cb = CircuitBreaker::new(config);

        assert!(cb.call(|| Ok::<_, &str>(42)).is_ok());
        assert_eq!(cb.state(), CircuitState::Closed);

        for _ in 0..3 { let _ = cb.call(|| Err::<i32, _>("error")); }
        assert_eq!(cb.state(), CircuitState::Open);

        let result = cb.call(|| Ok::<_, &str>(42));
        assert!(matches!(result, Err(CircuitError::CircuitOpen)));
    }

    #[test]
    fn transitions_to_half_open_after_timeout() {
        let config = CircuitBreakerConfig {
            failure_threshold: 1,
            reset_timeout: Duration::from_millis(10),
            success_threshold: 1,
        };
        let cb = CircuitBreaker::new(config);

        let _ = cb.call(|| Err::<i32, _>("error"));
        assert_eq!(cb.state(), CircuitState::Open);

        std::thread::sleep(Duration::from_millis(20));

        assert!(cb.call(|| Ok::<_, &str>(42)).is_ok());
        assert_eq!(cb.state(), CircuitState::Closed);
    }

    #[test]
    fn half_open_failure_reopens_circuit() {
        let config = CircuitBreakerConfig {
            failure_threshold: 1,
            reset_timeout: Duration::from_millis(10),
            success_threshold: 2,
        };
        let cb = CircuitBreaker::new(config);

        let _ = cb.call(|| Err::<i32, _>("error"));
        std::thread::sleep(Duration::from_millis(20));

        let _ = cb.call(|| Err::<i32, _>("error"));
        assert_eq!(cb.state(), CircuitState::Open);
    }
}
```

### Graceful Shutdown with Workers

```rust
use tokio::sync::Notify;

pub struct ShutdownController {
    notify: Arc<Notify>,
}

impl ShutdownController {
    pub fn new() -> Self { Self { notify: Arc::new(Notify::new()) } }

    pub fn subscribe(&self) -> ShutdownReceiver {
        ShutdownReceiver { notify: self.notify.clone() }
    }

    pub fn shutdown(&self) { self.notify.notify_waiters(); }

    pub async fn wait_for_signal(&self) {
        let ctrl_c = async {
            tokio::signal::ctrl_c().await.expect("Failed to install Ctrl+C handler");
        };

        #[cfg(unix)]
        let terminate = async {
            tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
                .expect("Failed to install SIGTERM handler")
                .recv().await;
        };

        #[cfg(not(unix))]
        let terminate = std::future::pending::<()>();

        tokio::select! {
            _ = ctrl_c => tracing::info!("Received Ctrl+C"),
            _ = terminate => tracing::info!("Received SIGTERM"),
        }

        self.shutdown();
    }
}

#[derive(Clone)]
pub struct ShutdownReceiver {
    notify: Arc<Notify>,
}

impl ShutdownReceiver {
    pub async fn recv(&self) { self.notify.notified().await; }
}

// Worker with shutdown support
async fn worker(id: usize, shutdown: ShutdownReceiver) {
    tracing::info!(worker_id = id, "Worker started");

    loop {
        tokio::select! {
            _ = shutdown.recv() => {
                tracing::info!(worker_id = id, "Shutting down");
                break;
            }
            _ = do_work() => {}
        }
    }
}

#[tokio::main]
async fn main() {
    let shutdown = ShutdownController::new();

    let mut handles = vec![];
    for id in 0..4 {
        let rx = shutdown.subscribe();
        handles.push(tokio::spawn(worker(id, rx)));
    }

    shutdown.wait_for_signal().await;

    let timeout = Duration::from_secs(30);
    for handle in handles {
        let _ = tokio::time::timeout(timeout, handle).await;
    }
    tracing::info!("Shutdown complete");
}
```

### Thread Coordination with AtomicBool and Barrier

For synchronous thread coordination during shutdown:

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::{Arc, Barrier};
use std::thread;

pub struct ThreadPool {
    workers: Vec<thread::JoinHandle<()>>,
    shutdown_flag: Arc<AtomicBool>,
    shutdown_barrier: Arc<Barrier>,
}

impl ThreadPool {
    pub fn new(num_workers: usize) -> Self {
        let shutdown_flag = Arc::new(AtomicBool::new(false));
        let shutdown_barrier = Arc::new(Barrier::new(num_workers + 1));

        let workers: Vec<_> = (0..num_workers)
            .map(|id| {
                let flag = shutdown_flag.clone();
                let barrier = shutdown_barrier.clone();
                thread::spawn(move || {
                    while !flag.load(Ordering::SeqCst) {
                        thread::sleep(Duration::from_millis(100));
                        if flag.load(Ordering::SeqCst) { break; }
                        // Do work...
                    }
                    // Cleanup then signal completion
                    barrier.wait();
                })
            })
            .collect();

        Self { workers, shutdown_flag, shutdown_barrier }
    }

    pub fn shutdown(self) {
        self.shutdown_flag.store(true, Ordering::SeqCst);
        self.shutdown_barrier.wait();  // Wait for all workers
        for handle in self.workers { let _ = handle.join(); }
    }
}
```

### Connection Draining

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use tokio::time::{Duration, Instant};

pub struct ConnectionTracker {
    active: Arc<AtomicUsize>,
}

impl ConnectionTracker {
    pub fn new() -> Self { Self { active: Arc::new(AtomicUsize::new(0)) } }

    pub fn connection_guard(&self) -> ConnectionGuard {
        self.active.fetch_add(1, Ordering::SeqCst);
        ConnectionGuard { counter: self.active.clone() }
    }

    pub fn active_connections(&self) -> usize {
        self.active.load(Ordering::SeqCst)
    }

    /// Wait for all connections to drain with timeout
    pub async fn drain(&self, timeout: Duration) -> bool {
        let deadline = Instant::now() + timeout;

        while self.active.load(Ordering::SeqCst) > 0 {
            if Instant::now() >= deadline {
                tracing::warn!(
                    remaining = self.active.load(Ordering::SeqCst),
                    "Drain timeout exceeded"
                );
                return false;
            }
            tokio::time::sleep(Duration::from_millis(100)).await;
        }

        tracing::info!("All connections drained");
        true
    }
}

pub struct ConnectionGuard {
    counter: Arc<AtomicUsize>,
}

impl Drop for ConnectionGuard {
    fn drop(&mut self) {
        self.counter.fetch_sub(1, Ordering::SeqCst);
    }
}

// Usage in request handler
async fn handle_request(tracker: &ConnectionTracker) {
    let _guard = tracker.connection_guard();
    // Guard automatically decrements on drop
    tokio::time::sleep(Duration::from_millis(100)).await;
}
```

## CAP Theorem and Trade-offs

The CAP theorem states that a distributed system cannot simultaneously provide all three:
- **Consistency (C)**: Every read receives the most recent write
- **Availability (A)**: Every request receives a response (not necessarily the latest)
- **Partition Tolerance (P)**: System continues despite network partitions

Since network partitions are inevitable, you must choose between CP or AP.

### CP System (Consistency over Availability)

```rust
/// CP system: Refuses to serve during partition if quorum unavailable
pub struct ConsistentKVStore {
    data: Arc<RwLock<HashMap<String, String>>>,
    quorum_size: usize,
    available_nodes: Arc<RwLock<Vec<bool>>>,
}

#[derive(Debug, thiserror::Error)]
pub enum ConsistencyError {
    #[error("Quorum not available ({available}/{required} nodes)")]
    QuorumUnavailable { available: usize, required: usize },
    #[error("Key not found")]
    NotFound,
}

impl ConsistentKVStore {
    pub fn new(total_nodes: usize) -> Self {
        let quorum_size = total_nodes / 2 + 1;
        Self {
            data: Arc::new(RwLock::new(HashMap::new())),
            quorum_size,
            available_nodes: Arc::new(RwLock::new(vec![true; total_nodes])),
        }
    }

    fn check_quorum(&self) -> Result<(), ConsistencyError> {
        let nodes = self.available_nodes.read().unwrap();
        let available = nodes.iter().filter(|&&n| n).count();
        if available < self.quorum_size {
            return Err(ConsistencyError::QuorumUnavailable {
                available, required: self.quorum_size,
            });
        }
        Ok(())
    }

    /// Get value — fails if quorum unavailable (CP behavior)
    pub fn get(&self, key: &str) -> Result<String, ConsistencyError> {
        self.check_quorum()?;
        self.data.read().unwrap().get(key).cloned()
            .ok_or(ConsistencyError::NotFound)
    }

    /// Set value — fails if quorum unavailable (CP behavior)
    pub fn set(&self, key: String, value: String) -> Result<(), ConsistencyError> {
        self.check_quorum()?;
        self.data.write().unwrap().insert(key, value);
        Ok(())
    }

    pub fn partition_node(&self, node_id: usize) {
        if let Some(node) = self.available_nodes.write().unwrap().get_mut(node_id) {
            *node = false;
        }
    }

    pub fn recover_node(&self, node_id: usize) {
        if let Some(node) = self.available_nodes.write().unwrap().get_mut(node_id) {
            *node = true;
        }
    }
}

#[cfg(test)]
mod cp_tests {
    use super::*;

    #[test]
    fn refuses_operations_without_quorum() {
        let store = ConsistentKVStore::new(3); // Quorum = 2

        store.set("key".into(), "value".into()).unwrap();
        assert_eq!(store.get("key").unwrap(), "value");

        // Partition 2 nodes — below quorum
        store.partition_node(0);
        store.partition_node(1);

        assert!(matches!(store.get("key"), Err(ConsistencyError::QuorumUnavailable { .. })));
    }
}
```

### AP System (Availability over Consistency)

```rust
/// AP system: Always serves requests, may return stale data
pub struct AvailableKVStore {
    node_id: String,
    local_data: Arc<RwLock<HashMap<String, VersionedValue>>>,
}

#[derive(Clone, Debug)]
pub struct VersionedValue {
    pub value: String,
    pub version: u64,
    pub timestamp: u64,
}

impl AvailableKVStore {
    pub fn new(node_id: &str) -> Self {
        Self {
            node_id: node_id.to_string(),
            local_data: Arc::new(RwLock::new(HashMap::new())),
        }
    }

    /// Always succeeds with local data (AP behavior)
    pub fn get(&self, key: &str) -> Option<VersionedValue> {
        self.local_data.read().unwrap().get(key).cloned()
    }

    /// Always succeeds locally (AP behavior)
    pub fn set(&self, key: String, value: String) -> VersionedValue {
        let mut data = self.local_data.write().unwrap();
        let version = data.get(&key).map(|v| v.version + 1).unwrap_or(1);
        let versioned = VersionedValue {
            value, version,
            timestamp: std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH).unwrap()
                .as_millis() as u64,
        };
        data.insert(key, versioned.clone());
        versioned
    }

    /// Merge data from another node (eventual consistency)
    /// Uses last-write-wins conflict resolution
    pub fn merge(&self, key: String, remote_value: VersionedValue) {
        let mut data = self.local_data.write().unwrap();
        let should_update = match data.get(&key) {
            None => true,
            Some(local) => remote_value.timestamp > local.timestamp,
        };
        if should_update { data.insert(key, remote_value); }
    }
}

#[cfg(test)]
mod ap_tests {
    use super::*;

    #[test]
    fn always_available_during_partition() {
        let node_a = AvailableKVStore::new("A");
        let node_b = AvailableKVStore::new("B");

        // Both nodes write independently during partition
        node_a.set("counter".into(), "10".into());
        node_b.set("counter".into(), "20".into());

        assert_eq!(node_a.get("counter").unwrap().value, "10");
        assert_eq!(node_b.get("counter").unwrap().value, "20");

        // After partition heals, merge resolves conflicts
        let value_b = node_b.get("counter").unwrap();
        node_a.merge("counter".into(), value_b);
        assert_eq!(node_a.get("counter").unwrap().value, "20");
    }
}
```

### Choosing Between CP and AP

| Use Case | Choose | Reason |
|----------|--------|--------|
| Financial transactions | CP | Data integrity critical |
| Inventory management | CP | Overselling is costly |
| User sessions | AP | Availability more important |
| Social media feeds | AP | Stale data acceptable |
| Configuration management | CP | Consistency required |
| Caching layer | AP | Stale data tolerable |

## Consensus Algorithms

### Raft Overview

Raft is a consensus algorithm designed for understandability:
1. **Leader Election**: One node elected leader, handles all client requests
2. **Log Replication**: Leader replicates log entries to followers
3. **Safety**: Only nodes with up-to-date logs can become leader

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum RaftState { Follower, Candidate, Leader }

pub struct RaftNode {
    pub id: u64,
    pub state: RaftState,
    pub current_term: u64,
    pub voted_for: Option<u64>,
    pub log: Vec<LogEntry>,
    pub commit_index: u64,
}

#[derive(Debug, Clone)]
pub struct LogEntry { pub term: u64, pub index: u64, pub command: String }

#[derive(Debug)]
pub struct RequestVote {
    pub term: u64,
    pub candidate_id: u64,
    pub last_log_index: u64,
    pub last_log_term: u64,
}

#[derive(Debug)]
pub struct RequestVoteResponse {
    pub term: u64,
    pub vote_granted: bool,
}

#[derive(Debug)]
pub struct AppendEntries {
    pub term: u64,
    pub leader_id: u64,
    pub prev_log_index: u64,
    pub prev_log_term: u64,
    pub entries: Vec<LogEntry>,
    pub leader_commit: u64,
}

#[derive(Debug)]
pub struct AppendEntriesResponse {
    pub term: u64,
    pub success: bool,
}

impl RaftNode {
    pub fn new(id: u64) -> Self {
        Self { id, state: RaftState::Follower, current_term: 0,
               voted_for: None, log: vec![], commit_index: 0 }
    }

    pub fn handle_request_vote(&mut self, request: RequestVote) -> RequestVoteResponse {
        if request.term > self.current_term {
            self.current_term = request.term;
            self.state = RaftState::Follower;
            self.voted_for = None;
        }

        if request.term < self.current_term {
            return RequestVoteResponse { term: self.current_term, vote_granted: false };
        }

        let can_vote = self.voted_for.is_none()
            || self.voted_for == Some(request.candidate_id);
        let last_log_term = self.log.last().map(|e| e.term).unwrap_or(0);
        let last_log_index = self.log.len() as u64;
        let log_ok = request.last_log_term > last_log_term
            || (request.last_log_term == last_log_term
                && request.last_log_index >= last_log_index);

        let vote_granted = can_vote && log_ok;
        if vote_granted { self.voted_for = Some(request.candidate_id); }

        RequestVoteResponse { term: self.current_term, vote_granted }
    }
}
```

### Using raft-rs

For production use, use the `raft-rs` crate:

```toml
[dependencies]
raft = "0.7"
```

```rust
use raft::{prelude::*, storage::MemStorage, Config, RawNode};

fn create_raft_node(id: u64, peers: Vec<u64>) -> RawNode<MemStorage> {
    let config = Config {
        id,
        election_tick: 10,
        heartbeat_tick: 3,
        max_size_per_msg: 1024 * 1024,
        max_inflight_msgs: 256,
        ..Default::default()
    };

    let storage = MemStorage::new();

    if peers.is_empty() {
        let mut snapshot = Snapshot::default();
        snapshot.mut_metadata().index = 0;
        snapshot.mut_metadata().term = 0;
        snapshot.mut_metadata().mut_conf_state().voters = vec![id];
        storage.wl().apply_snapshot(snapshot).unwrap();
    }

    RawNode::new(&config, storage, &slog::Logger::root(slog::Discard, slog::o!())).unwrap()
}
```

## Distributed Tracing with OpenTelemetry

### Tracer Setup

```rust
use opentelemetry::{global, trace::TracerProvider as _};
use opentelemetry_sdk::{
    runtime,
    trace::{RandomIdGenerator, Sampler, TracerProvider},
    Resource,
};
use opentelemetry_otlp::WithExportConfig;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing(service_name: &str) -> Result<(), Box<dyn std::error::Error>> {
    let exporter = opentelemetry_otlp::new_exporter()
        .tonic()
        .with_endpoint("http://localhost:4317");

    let tracer_provider = TracerProvider::builder()
        .with_batch_exporter(exporter.build_span_exporter()?, runtime::Tokio)
        .with_sampler(Sampler::AlwaysOn)
        .with_id_generator(RandomIdGenerator::default())
        .with_resource(Resource::new(vec![
            opentelemetry::KeyValue::new("service.name", service_name.to_string()),
            opentelemetry::KeyValue::new("service.version", env!("CARGO_PKG_VERSION")),
        ]))
        .build();

    global::set_tracer_provider(tracer_provider.clone());

    let otel_layer = tracing_opentelemetry::layer()
        .with_tracer(tracer_provider.tracer(service_name));

    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer().json())
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(otel_layer)
        .init();

    Ok(())
}

pub fn shutdown_tracing() {
    global::shutdown_tracer_provider();
}
```

### Context Propagation Across Services

```rust
use opentelemetry::{global, propagation::Extractor, trace::{SpanKind, Tracer}, Context};

struct HeaderExtractor<'a>(&'a axum::http::HeaderMap);

impl<'a> Extractor for HeaderExtractor<'a> {
    fn get(&self, key: &str) -> Option<&str> {
        self.0.get(key).and_then(|v| v.to_str().ok())
    }
    fn keys(&self) -> Vec<&str> {
        self.0.keys().map(|k| k.as_str()).collect()
    }
}

/// Middleware to extract trace context from incoming requests
pub async fn trace_context_middleware(
    headers: axum::http::HeaderMap,
    request: axum::extract::Request,
    next: axum::middleware::Next,
) -> axum::response::Response {
    let parent_context = global::get_text_map_propagator(|propagator| {
        propagator.extract(&HeaderExtractor(&headers))
    });

    let tracer = global::tracer("http-server");
    let span = tracer
        .span_builder(format!("{} {}", request.method(), request.uri().path()))
        .with_kind(SpanKind::Server)
        .start_with_context(&tracer, &parent_context);

    let cx = Context::current_with_span(span);
    let _guard = cx.attach();

    let current_span = cx.span();
    current_span.set_attribute(opentelemetry::KeyValue::new(
        "http.method", request.method().to_string(),
    ));
    current_span.set_attribute(opentelemetry::KeyValue::new(
        "http.url", request.uri().to_string(),
    ));

    let response = next.run(request).await;

    current_span.set_attribute(opentelemetry::KeyValue::new(
        "http.status_code", response.status().as_u16() as i64,
    ));

    response
}
```

### Propagating Context to Downstream Services

```rust
use opentelemetry::{global, propagation::Injector, Context};

struct HeaderInjector<'a>(&'a mut reqwest::header::HeaderMap);

impl<'a> Injector for HeaderInjector<'a> {
    fn set(&mut self, key: &str, value: String) {
        if let Ok(name) = reqwest::header::HeaderName::from_bytes(key.as_bytes()) {
            if let Ok(val) = reqwest::header::HeaderValue::from_str(&value) {
                self.0.insert(name, val);
            }
        }
    }
}

pub async fn call_downstream_service(
    client: &reqwest::Client,
    url: &str,
) -> Result<String, reqwest::Error> {
    let mut headers = reqwest::header::HeaderMap::new();

    // Inject current trace context into outgoing headers
    let cx = Context::current();
    global::get_text_map_propagator(|propagator| {
        propagator.inject_context(&cx, &mut HeaderInjector(&mut headers));
    });

    client.get(url).headers(headers).send().await?.text().await
}
```

### Adding Span Events and Attributes

```rust
use opentelemetry::trace::{Span, Status, TraceContextExt};

async fn process_order(order_id: &str) -> Result<(), OrderError> {
    let cx = opentelemetry::Context::current();
    let span = cx.span();

    span.set_attribute(opentelemetry::KeyValue::new("order.id", order_id.to_string()));
    span.add_event("validating_order", vec![]);

    validate_order(order_id).await?;

    span.add_event("charging_payment", vec![
        opentelemetry::KeyValue::new("payment.method", "credit_card"),
    ]);

    match charge_payment(order_id).await {
        Ok(transaction_id) => {
            span.set_attribute(opentelemetry::KeyValue::new(
                "payment.transaction_id", transaction_id,
            ));
            span.add_event("payment_successful", vec![]);
        }
        Err(e) => {
            span.set_status(Status::error(e.to_string()));
            span.record_error(&e);
            return Err(e);
        }
    }

    span.add_event("order_completed", vec![]);
    Ok(())
}
```

## TCP Server/Client

### Length-Prefixed Binary Protocol

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
pub enum KvMessage {
    Get(Vec<u8>),
    Put((Vec<u8>, Vec<u8>)),
    Del(Vec<u8>),
    Success(bool),
    ReturnValue(Option<Vec<u8>>),
}

impl KvMessage {
    pub fn package(&self) -> Vec<u8> {
        let data = bincode::serialize(&self).unwrap();
        let mut buf = Vec::with_capacity(4 + data.len());
        buf.extend_from_slice(&(data.len() as u32).to_be_bytes());
        buf.extend_from_slice(&data);
        buf
    }
}

async fn read_message(stream: &mut TcpStream) -> std::io::Result<Vec<u8>> {
    let mut len_buf = [0u8; 4];
    stream.read_exact(&mut len_buf).await?;
    let len = u32::from_be_bytes(len_buf) as usize;
    let mut buffer = vec![0u8; len];
    stream.read_exact(&mut buffer).await?;
    Ok(buffer)
}
```

### TCP Server

```rust
#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:9000").await.unwrap();
    let state = Arc::new(Mutex::new(HashMap::<Vec<u8>, Vec<u8>>::new()));

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        let state = state.clone();
        tokio::spawn(async move { process(socket, state).await });
    }
}

async fn process(mut socket: TcpStream, state: Arc<Mutex<HashMap<Vec<u8>, Vec<u8>>>>) {
    let buffer = match read_message(&mut socket).await {
        Ok(b) => b,
        Err(_) => return,
    };
    let message: KvMessage = bincode::deserialize(&buffer).unwrap();
    let response = handle_message(message, &state);
    let _ = socket.write_all(&response.package()).await;
}
```

## TLS/HTTPS

### Server-Side HTTPS (Actix + OpenSSL)

```rust
use actix_web::{web, App, HttpServer};
use openssl::ssl::{SslAcceptor, SslFiletype, SslMethod};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let mut builder = SslAcceptor::mozilla_intermediate(SslMethod::tls()).unwrap();
    builder.set_private_key_file("key.pem", SslFiletype::PEM).unwrap();
    builder.set_certificate_chain_file("cert.pem").unwrap();

    HttpServer::new(|| App::new().route("/api/data", web::post().to(handler)))
        .bind_openssl("0.0.0.0:8443", builder)?
        .run().await
}
```

### Client-Side HTTPS (native-tls)

```rust
use native_tls::TlsConnector;
use std::io::{Read, Write};
use std::net::TcpStream;

fn https_client() -> Result<(), Box<dyn std::error::Error>> {
    let connector = TlsConnector::builder().build()?;
    let stream = TcpStream::connect("example.com:443")?;
    let mut stream = connector.connect("example.com", stream)?;

    stream.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")?;
    let mut response = String::new();
    stream.read_to_string(&mut response)?;
    Ok(())
}
```

### Certificate Generation (Development)

```bash
openssl genrsa -out key.pem 2048
openssl req -new -x509 -key key.pem -out cert.pem -days 365
```

## Inter-Context Communication

### gRPC with Tonic

```rust
// Define service in proto file, then use generated code
use tonic::transport::Channel;
use user_proto::user_service_client::UserServiceClient;
use user_proto::{GetUserRequest, GetUserResponse};

pub struct UserServiceGrpcClient {
    client: UserServiceClient<Channel>,
}

impl UserServiceGrpcClient {
    pub async fn connect(endpoint: &str) -> Result<Self, tonic::transport::Error> {
        let client = UserServiceClient::connect(endpoint.to_string()).await?;
        Ok(Self { client })
    }

    pub async fn get_user(&mut self, user_id: &str) -> Result<UserResponse, GrpcError> {
        let request = tonic::Request::new(GetUserRequest {
            user_id: user_id.to_string(),
        });

        let response = self.client.get_user(request).await
            .map_err(|e| GrpcError::Transport(e.to_string()))?;

        let inner = response.into_inner();
        Ok(UserResponse {
            user_id: inner.user_id,
            username: inner.username,
            email: inner.email,
        })
    }
}

#[derive(Debug, thiserror::Error)]
pub enum GrpcError {
    #[error("Transport error: {0}")] Transport(String),
    #[error("Not found")] NotFound,
}

// Server implementation
use tonic::{Request, Response, Status};
use user_proto::user_service_server::{UserService, UserServiceServer};

#[derive(Default)]
pub struct UserServiceImpl {}

#[tonic::async_trait]
impl UserService for UserServiceImpl {
    async fn get_user(
        &self, request: Request<GetUserRequest>,
    ) -> Result<Response<GetUserResponse>, Status> {
        let user_id = request.into_inner().user_id;

        // Fetch from repository
        Ok(Response::new(GetUserResponse {
            user_id,
            username: "example".to_string(),
            email: "example@test.com".to_string(),
        }))
    }
}

// Starting the server
pub async fn start_grpc_server(addr: &str) -> Result<(), Box<dyn std::error::Error>> {
    let service = UserServiceImpl::default();
    tonic::transport::Server::builder()
        .add_service(UserServiceServer::new(service))
        .serve(addr.parse()?)
        .await?;
    Ok(())
}
```

### RabbitMQ with Lapin

```rust
use lapin::{
    options::*, types::FieldTable, BasicProperties, Channel, Connection, ConnectionProperties,
};
use futures_util::stream::StreamExt;

// Publisher
pub struct RabbitMqPublisher {
    channel: Channel,
    exchange: String,
}

impl RabbitMqPublisher {
    pub async fn connect(uri: &str, exchange: &str) -> Result<Self, lapin::Error> {
        let conn = Connection::connect(uri, ConnectionProperties::default()).await?;
        let channel = conn.create_channel().await?;

        channel.exchange_declare(
            exchange,
            lapin::ExchangeKind::Topic,
            ExchangeDeclareOptions::default(),
            FieldTable::default(),
        ).await?;

        Ok(Self { channel, exchange: exchange.to_string() })
    }

    pub async fn publish<T: serde::Serialize>(
        &self, routing_key: &str, message: &T,
    ) -> Result<(), lapin::Error> {
        let payload = serde_json::to_vec(message).unwrap();

        self.channel.basic_publish(
            &self.exchange, routing_key,
            BasicPublishOptions::default(),
            &payload,
            BasicProperties::default()
                .with_content_type("application/json".into())
                .with_delivery_mode(2),  // Persistent
        ).await?;

        Ok(())
    }
}

// Consumer
pub struct RabbitMqConsumer {
    channel: Channel,
    queue_name: String,
}

impl RabbitMqConsumer {
    pub async fn connect(
        uri: &str, exchange: &str, queue: &str, routing_key: &str,
    ) -> Result<Self, lapin::Error> {
        let conn = Connection::connect(uri, ConnectionProperties::default()).await?;
        let channel = conn.create_channel().await?;

        channel.queue_declare(queue, QueueDeclareOptions::default(), FieldTable::default()).await?;
        channel.queue_bind(queue, exchange, routing_key,
            QueueBindOptions::default(), FieldTable::default()).await?;

        Ok(Self { channel, queue_name: queue.to_string() })
    }

    pub async fn consume<F, Fut>(&self, handler: F) -> Result<(), lapin::Error>
    where
        F: Fn(Vec<u8>) -> Fut + Send + 'static,
        Fut: std::future::Future<Output = Result<(), Box<dyn std::error::Error>>> + Send,
    {
        let mut consumer = self.channel.basic_consume(
            &self.queue_name, "consumer",
            BasicConsumeOptions::default(), FieldTable::default(),
        ).await?;

        while let Some(delivery) = consumer.next().await {
            if let Ok(delivery) = delivery {
                match handler(delivery.data.clone()).await {
                    Ok(()) => delivery.ack(BasicAckOptions::default()).await?,
                    Err(e) => {
                        eprintln!("Handler error: {}", e);
                        delivery.nack(BasicNackOptions { requeue: true, ..Default::default() }).await?;
                    }
                }
            }
        }
        Ok(())
    }
}

// Usage
#[derive(Debug, Serialize, Deserialize)]
struct OrderCreatedEvent {
    order_id: String,
    customer_id: String,
    total: u64,
}

async fn publish_order_created(publisher: &RabbitMqPublisher, order_id: &str) {
    let event = OrderCreatedEvent {
        order_id: order_id.to_string(),
        customer_id: "customer123".to_string(),
        total: 9999,
    };
    publisher.publish("orders.created", &event).await.unwrap();
}
```

### Dead-Letter Queue Pattern

```rust
pub async fn setup_queue_with_dlq(channel: &lapin::Channel) -> Result<(), lapin::Error> {
    // Declare dead-letter exchange
    channel.exchange_declare(
        "dlx", lapin::ExchangeKind::Direct,
        ExchangeDeclareOptions::default(), FieldTable::default(),
    ).await?;

    // Dead-letter queue
    channel.queue_declare("failed_messages",
        QueueDeclareOptions::default(), FieldTable::default()).await?;
    channel.queue_bind("failed_messages", "dlx", "failed",
        QueueBindOptions::default(), FieldTable::default()).await?;

    // Main queue with DLQ settings
    let mut args = FieldTable::default();
    args.insert("x-dead-letter-exchange".into(), "dlx".into());
    args.insert("x-dead-letter-routing-key".into(), "failed".into());
    args.insert("x-message-ttl".into(), 60000i32.into());

    channel.queue_declare("main_queue", QueueDeclareOptions::default(), args).await?;
    Ok(())
}

// Consumer that rejects to DLQ after max retries
pub async fn consume_with_retry_limit(
    channel: &lapin::Channel, queue: &str, max_retries: u32,
) -> Result<(), lapin::Error> {
    let mut consumer = channel.basic_consume(
        queue, "consumer", BasicConsumeOptions::default(), FieldTable::default(),
    ).await?;

    while let Some(delivery) = consumer.next().await {
        if let Ok(delivery) = delivery {
            let retry_count = delivery.properties.headers()
                .as_ref()
                .and_then(|h| h.inner().get("x-retry-count"))
                .and_then(|v| v.as_long_int())
                .unwrap_or(0) as u32;

            match process_delivery(&delivery.data).await {
                Ok(()) => delivery.ack(BasicAckOptions::default()).await?,
                Err(_) if retry_count < max_retries => {
                    delivery.nack(BasicNackOptions { requeue: true, ..Default::default() }).await?;
                }
                Err(_) => {
                    // Max retries — reject to DLQ
                    delivery.nack(BasicNackOptions { requeue: false, ..Default::default() }).await?;
                }
            }
        }
    }
    Ok(())
}
```

### QoS/Prefetch for Flow Control

```rust
pub async fn setup_consumer_with_qos(
    channel: &lapin::Channel, queue: &str,
) -> Result<lapin::Consumer, lapin::Error> {
    // Consumer won't receive more than N unacked messages
    channel.basic_qos(10, BasicQosOptions::default()).await?;

    channel.basic_consume(
        queue, "consumer_tag",
        BasicConsumeOptions::default(), FieldTable::default(),
    ).await
}
```

### Kafka with rdkafka

```rust
use rdkafka::config::ClientConfig;
use rdkafka::producer::{FutureProducer, FutureRecord};
use rdkafka::consumer::{StreamConsumer, Consumer};
use rdkafka::Message;

// Producer
pub struct KafkaEventPublisher {
    producer: FutureProducer,
    topic: String,
}

impl KafkaEventPublisher {
    pub fn new(brokers: &str, topic: &str) -> Self {
        let producer: FutureProducer = ClientConfig::new()
            .set("bootstrap.servers", brokers)
            .set("message.timeout.ms", "5000")
            .create().expect("Producer creation failed");

        Self { producer, topic: topic.to_string() }
    }

    pub async fn publish(&self, key: &str, payload: &[u8]) -> Result<(), KafkaError> {
        let record = FutureRecord::to(&self.topic).key(key).payload(payload);
        self.producer.send(record, Duration::from_secs(5)).await
            .map_err(|(e, _)| KafkaError::Send(e))?;
        Ok(())
    }
}

// Consumer
pub async fn consume_events(brokers: &str, group_id: &str, topic: &str) {
    let consumer: StreamConsumer = ClientConfig::new()
        .set("bootstrap.servers", brokers)
        .set("group.id", group_id)
        .set("auto.offset.reset", "earliest")
        .create().expect("Consumer creation failed");

    consumer.subscribe(&[topic]).expect("Subscription failed");

    loop {
        match consumer.recv().await {
            Ok(message) => {
                if let Some(payload) = message.payload() {
                    println!("Received: {:?}", std::str::from_utf8(payload));
                }
            }
            Err(e) => eprintln!("Kafka error: {}", e),
        }
    }
}
```

### Manual Offset Commits (At-Least-Once)

```rust
use rdkafka::consumer::{CommitMode, Consumer, StreamConsumer};

pub async fn consume_with_manual_commit(brokers: &str, group_id: &str, topic: &str) {
    let consumer: StreamConsumer = ClientConfig::new()
        .set("bootstrap.servers", brokers)
        .set("group.id", group_id)
        .set("enable.auto.commit", "false")
        .set("auto.offset.reset", "earliest")
        .create().expect("Consumer creation failed");

    consumer.subscribe(&[topic]).expect("Subscription failed");

    loop {
        match consumer.recv().await {
            Ok(message) => {
                if let Err(e) = process_kafka_message(&message).await {
                    eprintln!("Processing failed: {}", e);
                    continue;  // Don't commit — will be redelivered
                }

                if let Err(e) = consumer.commit_message(&message, CommitMode::Async) {
                    eprintln!("Commit failed: {}", e);
                }
            }
            Err(e) => eprintln!("Kafka error: {}", e),
        }
    }
}
```

### Batch Processing with Periodic Commits

```rust
pub async fn consume_batched(
    consumer: &StreamConsumer, batch_size: usize, commit_interval: Duration,
) {
    let mut batch = Vec::with_capacity(batch_size);
    let mut last_commit = std::time::Instant::now();

    loop {
        match tokio::time::timeout(Duration::from_millis(100), consumer.recv()).await {
            Ok(Ok(message)) => {
                batch.push(message);

                if batch.len() >= batch_size || last_commit.elapsed() >= commit_interval {
                    process_kafka_batch(&batch).await;
                    if let Some(last) = batch.last() {
                        let _ = consumer.commit_message(last, CommitMode::Async);
                    }
                    batch.clear();
                    last_commit = std::time::Instant::now();
                }
            }
            Ok(Err(e)) => eprintln!("Kafka error: {}", e),
            Err(_) => {
                if !batch.is_empty() && last_commit.elapsed() >= commit_interval {
                    process_kafka_batch(&batch).await;
                    if let Some(last) = batch.last() {
                        let _ = consumer.commit_message(last, CommitMode::Async);
                    }
                    batch.clear();
                    last_commit = std::time::Instant::now();
                }
            }
        }
    }
}
```

### Communication Strategy Trade-offs

| Strategy | Coupling | Latency | Consistency | Fault Tolerance | Best For |
|----------|----------|---------|-------------|-----------------|----------|
| **gRPC** | Medium | Low | Strong | Low | Internal microservices, real-time queries |
| **REST** | Medium | Low-Medium | Strong | Low | External APIs, simple CRUD |
| **Message Queue** | Low | High | Eventual | High | Event-driven workflows, decoupled services |
| **Kafka Streaming** | Low | Medium | Eventual | Very High | Event sourcing, high-throughput data |
| **Shared Database** | **Very High** | Low | Strong | Low | **AVOID** — anti-pattern |

**Shared Database Warning:**
- Creates tight coupling between services
- Schema changes break multiple services
- Unclear data ownership
- Testing in isolation becomes impossible
- Use read replicas or event-driven sync instead

## Protocol Versioning

### Versioned Message Enums

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TextMessageV1 {
    pub sender: String,
    pub recipient: String,
    pub content: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TextMessageV2 {
    pub sender: String,
    pub recipient: String,
    pub content: String,
    pub timestamp: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ChatMessageVersioned {
    V1(TextMessageV1),
    V2(TextMessageV2),
}

impl ChatMessageVersioned {
    pub fn to_v2(self) -> TextMessageV2 {
        match self {
            ChatMessageVersioned::V1(v1) => TextMessageV2 {
                sender: v1.sender, recipient: v1.recipient, content: v1.content,
                timestamp: 0,
            },
            ChatMessageVersioned::V2(v2) => v2,
        }
    }
}
```

### Version Field in Protocol Header

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProtocolHeader {
    pub version: u16,
    pub message_type: u16,
    pub payload_length: u32,
}

fn handle_message(msg: ProtocolMessage) -> Result<(), Box<dyn std::error::Error>> {
    match (msg.header.version, msg.header.message_type) {
        (1, 1) => { let text: TextMessageV1 = msg.decode_payload()?; handle_v1(text); }
        (2, 1) => { let text: TextMessageV2 = msg.decode_payload()?; handle_v2(text); }
        (v, t) => return Err(format!("Unknown version {} type {}", v, t).into()),
    }
    Ok(())
}
```

### Schema Evolution with Serde Attributes

```rust
#[derive(Serialize, Deserialize)]
struct UserProfileV2 {
    user_id: u64,
    username: String,
    email: Option<String>,
    active: bool,

    // New field with default — compatible with old data
    #[serde(default)]
    is_premium: bool,

    // Renamed field — accepts old name during deserialization
    #[serde(alias = "user_level")]
    account_tier: Option<String>,

    // Skip serializing if default
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    tags: Vec<String>,
}
```

## Storage Tiers

| Tier | Speed | Use Case |
|------|-------|----------|
| **Hot (Redis)** | Sub-millisecond | Caches, active sessions |
| **Warm (Postgres)** | Milliseconds | Primary databases |
| **Cold (S3/Archive)** | Seconds-minutes | Old data, backups |

## Redis Commands Reference

| Command | Description |
|---------|-------------|
| `LPUSH key value` | Push to front of queue |
| `RPUSH key value` | Push to back of queue |
| `BLPOP key timeout` | Blocking pop from front |
| `HSET key field value` | Set hash field |
| `HMGET key f1 f2...` | Get multiple hash fields |
| `SET key value EX ttl` | Set with expiry |
| `DEL key` | Delete key |
| `LLEN key` | Queue length |

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: async basics, error handling, serde essentials
- **[async-concurrency.md](async-concurrency.md)** — Tokio runtime, channels, graceful shutdown, Tower services
- **[architecture.md](architecture.md)** — Workspace design, application layering, nanoservices pattern
- **[web-apis.md](web-apis.md)** — Axum/Actix handlers, middleware, HTTP layer
- **[database.md](database.md)** — SQLx connection pools, caching strategies
- **[deployment.md](deployment.md)** — Docker Compose, CI/CD, observability
- **[domain-patterns.md](domain-patterns.md)** — Bounded contexts, inter-context communication, event sourcing
- **[unsafe-ffi.md](unsafe-ffi.md)** — Binary protocol serialization, byte manipulation
