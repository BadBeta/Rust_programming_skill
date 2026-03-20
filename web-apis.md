# Web APIs in Rust

Axum (primary), Actix Web, Rocket, Hyper (low-level), routing, extractors, middleware, error handling, authentication (Argon2, JWT, Basic Auth), sqlx database integration, reqwest HTTP client, WebSocket, CORS, static assets, SPA routing, and API versioning.

## Rules for Web APIs (LLM)

1. **ALWAYS use extractors for request parsing** — never manually parse request bodies or query strings; use `Json<T>`, `Query<T>`, `Path<T>` which validate and return proper error responses automatically
2. **NEVER put business logic in handlers** — handlers should extract, delegate to a service/domain layer, and convert the result to a response; this keeps handlers testable and swappable between frameworks
3. **ALWAYS return structured error responses** — implement `IntoResponse` for your error type; never return bare `StatusCode` or string errors from production APIs
4. **ALWAYS use `#[serde(deny_unknown_fields)]` on input types** — catches client typos and prevents silent data loss from misspelled fields
5. **NEVER use `allow_any_origin()` in production CORS** — specify exact allowed origins; permissive CORS is a security vulnerability
6. **ALWAYS use Argon2 for password hashing** — never bcrypt or scrypt; Argon2 won the Password Hashing Competition and is the current standard
7. **ALWAYS validate JWT expiration** — never disable `exp` validation; set reasonable token lifetimes (15min access, 7d refresh)
8. **PREFER `State<Arc<T>>` over `Extension<T>` for shared state** — `State` is type-checked at compile time and extracted via the type system; `Extension` panics at runtime if missing

### Common Mistakes (BAD/GOOD)

**Bare status code errors:**
```rust
// BAD: no error body, client gets empty response
async fn get_user(Path(id): Path<i64>) -> Result<Json<User>, StatusCode> {
    let user = find_user(id).await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(user))
}
```

```rust
// GOOD: structured error with message
async fn get_user(
    Path(id): Path<i64>,
    State(state): State<Arc<AppState>>,
) -> Result<Json<User>, AppError> {
    let user = state.user_service.find(id).await?;
    Ok(Json(user))
}

// AppError implements IntoResponse with JSON error body
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg.clone()),
            AppError::Internal(e) => (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()),
        };
        (status, Json(serde_json::json!({"error": message}))).into_response()
    }
}
```

**Business logic in handlers:**
```rust
// BAD: handler does everything — untestable, framework-coupled
async fn create_order(Json(input): Json<CreateOrderInput>) -> impl IntoResponse {
    let total = input.items.iter().map(|i| i.price * i.qty as f64).sum::<f64>();
    let tax = total * 0.08;
    sqlx::query!("INSERT INTO orders ...").execute(&pool).await.unwrap();
    // 50 more lines of business logic...
}
```

```rust
// GOOD: handler delegates to domain layer
async fn create_order(
    State(state): State<Arc<AppState>>,
    Json(input): Json<CreateOrderInput>,
) -> Result<Json<OrderResponse>, AppError> {
    let order = state.order_service.create(input).await?;
    Ok(Json(OrderResponse::from(order)))
}
```

**Missing input validation:**
```rust
// BAD: trusts client input blindly
#[derive(Deserialize)]
struct CreateUser { username: String, email: String }
```

```rust
// GOOD: validates at the boundary
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct CreateUser {
    #[serde(deserialize_with = "validate_username")]
    username: String,
    #[serde(deserialize_with = "validate_email")]
    email: String,
}
```

**Hardcoded CORS in production:**
```rust
// BAD: allows any origin
let cors = CorsLayer::permissive();
```

```rust
// GOOD: explicit origins
let cors = CorsLayer::new()
    .allow_origin("https://app.example.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers([CONTENT_TYPE, AUTHORIZATION]);
```

### Section Index

| Section | Topics |
|---------|--------|
| [Framework Comparison](#framework-comparison) | Axum vs Actix vs Rocket vs Hyper decision table |
| [Axum](#axum) | Extractors, routing, state, middleware, Tower layers, error handling |
| [Actix Web](#actix-web) | Handlers, extractors, middleware, state, websockets |
| [Rocket](#rocket) | Request guards, fairings, managed state |
| [Hyper](#hyper-low-level) | Raw HTTP, Service trait, connection handling |
| [HTTP Method Semantics](#http-method-semantics) | GET/POST/PUT/PATCH/DELETE conventions |
| [API Versioning](#api-versioning) | URL, header, content-type versioning strategies |
| [Cross-Framework Consistency](#cross-framework-consistency) | Shared patterns across frameworks |
| [Authentication](#authentication) | JWT, Argon2, sessions, middleware guards, refresh tokens |
| [Pagination](#pagination) | Cursor-based, offset-based, response format |
| [reqwest HTTP Client](#reqwest-http-client) | Client reuse, retry, JSON, multipart, streaming |
| [CORS Configuration](#cors-configuration) | CorsLayer, preflight, allowed origins |
| [WebSockets](#websockets) | Axum WS, Actix WS, message handling, broadcast |
| [Authenticated WebSockets](#authenticated-websockets) | JWT validation on upgrade, user-attributed messages |
| [Multi-Room Chat](#multi-room-chat-with-persistence) | Room management, DashMap channels, save-then-broadcast, history loading |
| [WebSocket Heartbeat & Reconnection](#websocket-heartbeat--reconnection) | Ping/pong, dead connection cleanup, missed messages |
| [WebRTC Signaling](#webrtc-signaling-server) | SDP exchange, ICE candidates, peer-to-peer vs SFU, STUN/TURN |
| [WHIP/WHEP Signaling](#whipwhep--http-based-webrtc-signaling) | HTTP POST signaling, connect/ICE/restart endpoints, when HTTP vs WS |
| [Static Asset Serving](#static-asset-serving) | ServeDir, SPA fallback, compression |
| [Security Checklist](#security-checklist) | Input validation, rate limiting, HTTPS, headers |

## Framework Comparison

| Aspect | Axum | Actix Web | Rocket | Hyper |
|--------|------|-----------|--------|-------|
| Style | Tower-based, modular | Actor-based, mature | Batteries-included | Low-level HTTP |
| Ecosystem | Tower/tokio native | Own actor system | Custom | Raw HTTP |
| State | `State<T>` | `web::Data<T>` | `&State<T>` | Manual |
| Extractors | `FromRequest` trait | `FromRequest` trait | Request guards | Manual parsing |
| Error handling | `IntoResponse` | `ResponseError` | `Status` enum | Manual |
| Runtime | tokio (required) | tokio | tokio | tokio |
| Middleware | Tower layers | Built-in middleware | Fairings | Tower/manual |
| Best for | **New projects** | Production APIs | Rapid prototyping | Custom protocols |
| Recommendation | **Preferred** — Tower-native, maintained by tokio team | Mature, battle-tested, high performance | Best developer ergonomics | Maximum control |

## Axum

### Basic Server Setup

```rust
use axum::{
    routing::{get, post, delete, patch},
    Router, Json,
    extract::{Path, Query, State},
    http::StatusCode,
    response::IntoResponse,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    db: sqlx::PgPool,
    jwt_secret: String,
}

#[tokio::main]
async fn main() {
    let pool = sqlx::PgPool::connect(&std::env::var("DATABASE_URL").unwrap())
        .await
        .unwrap();

    let state = Arc::new(AppState {
        db: pool,
        jwt_secret: std::env::var("JWT_SECRET").unwrap(),
    });

    let app = Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/{id}", get(get_user).delete(delete_user).patch(update_user))
        .route("/health", get(|| async { "ok" }))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### JSON Extraction and Response

```rust
#[derive(Deserialize)]
struct CreateUserInput {
    username: String,
    email: String,
}

#[derive(Serialize)]
struct UserResponse {
    id: i64,
    username: String,
    email: String,
}

// Axum uses tuples for (StatusCode, body) responses
async fn create_user(
    State(state): State<Arc<AppState>>,
    Json(input): Json<CreateUserInput>,
) -> Result<(StatusCode, Json<UserResponse>), ApiError> {
    let user = sqlx::query_as!(
        UserResponse,
        "INSERT INTO users (username, email) VALUES ($1, $2) RETURNING id, username, email",
        input.username,
        input.email,
    )
    .fetch_one(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(user)))
}

async fn get_user(
    State(state): State<Arc<AppState>>,
    Path(id): Path<i64>,
) -> Result<Json<UserResponse>, ApiError> {
    let user = sqlx::query_as!(
        UserResponse,
        "SELECT id, username, email FROM users WHERE id = $1",
        id,
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(ApiError::NotFound)?;

    Ok(Json(user))
}
```

### Custom Extractor

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

pub struct AuthUser {
    pub user_id: String,
}

impl<S: Send + Sync> FromRequestParts<S> for AuthUser {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        let token = parts
            .headers
            .get("Authorization")
            .and_then(|v| v.to_str().ok())
            .map(|s| s.trim_start_matches("Bearer "))
            .ok_or((StatusCode::UNAUTHORIZED, "Missing authorization"))?;

        let claims = decode_jwt(token)
            .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid token"))?;

        Ok(AuthUser { user_id: claims.sub })
    }
}

// Use in handler — extraction happens automatically
// If extraction fails, handler never runs — error response returned
async fn protected(
    user: AuthUser,
    Json(body): Json<CreateUserInput>,
) -> impl IntoResponse {
    // user.user_id is available
    StatusCode::OK
}
```

### Router Composition

```rust
fn api_routes() -> Router<Arc<AppState>> {
    Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/{id}", get(get_user).delete(delete_user).patch(update_user))
}

fn auth_routes() -> Router<Arc<AppState>> {
    Router::new()
        .route("/login", post(login))
        .route("/register", post(register))
        .route("/logout", post(logout))
}

fn admin_routes() -> Router<Arc<AppState>> {
    Router::new()
        .route("/stats", get(admin_stats))
        .route("/users", get(admin_list_users))
}

fn app(state: Arc<AppState>) -> Router {
    Router::new()
        .nest("/api/v1", api_routes())
        .nest("/auth", auth_routes())
        .nest("/admin", admin_routes())
        .route("/health", get(|| async { "ok" }))
        .with_state(state)
}
```

### Middleware with Tower

```rust
use tower_http::{
    cors::CorsLayer,
    trace::TraceLayer,
    compression::CompressionLayer,
    timeout::TimeoutLayer,
};
use axum::middleware;
use std::time::Duration;

let app = Router::new()
    .route("/api/users", get(list_users))
    .layer(TraceLayer::new_for_http())     // Request/response logging
    .layer(CompressionLayer::new())         // Gzip/brotli compression
    .layer(TimeoutLayer::new(Duration::from_secs(30))) // Request timeout
    .layer(CorsLayer::permissive())         // Restrict in production!
    .with_state(state);

// Custom middleware function
async fn auth_middleware(
    headers: axum::http::HeaderMap,
    request: axum::extract::Request,
    next: middleware::Next,
) -> Result<axum::response::Response, StatusCode> {
    let token = headers
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // Validate token...
    verify_token(token).map_err(|_| StatusCode::UNAUTHORIZED)?;

    Ok(next.run(request).await)
}

// Apply middleware to specific routes
let protected_routes = Router::new()
    .route("/protected", get(handler))
    .route("/admin", get(admin_handler))
    .layer(middleware::from_fn(auth_middleware));

let app = Router::new()
    .route("/public", get(public_handler))
    .merge(protected_routes)
    .with_state(state);
```

### Middleware with State Access

```rust
// from_fn_with_state — access AppState in middleware
async fn rate_limit_middleware(
    State(state): State<Arc<AppState>>,
    request: axum::extract::Request,
    next: middleware::Next,
) -> Result<axum::response::Response, StatusCode> {
    let ip = request.extensions().get::<ConnectInfo<SocketAddr>>()
        .map(|ci| ci.0.ip().to_string())
        .unwrap_or_default();

    if !state.rate_limiter.check(&ip).await {
        return Err(StatusCode::TOO_MANY_REQUESTS);
    }
    Ok(next.run(request).await)
}

let app = Router::new()
    .route("/api/data", get(handler))
    .layer(middleware::from_fn_with_state(state.clone(), rate_limit_middleware))
    .with_state(state);
```

### map_request / map_response Middleware

Simpler than `from_fn` when you only need to transform the request or response:

```rust
use axum::middleware::{map_request, map_response};

// Add a request ID to every request
async fn add_request_id(mut request: axum::extract::Request) -> axum::extract::Request {
    let id = uuid::Uuid::new_v4().to_string();
    request.headers_mut().insert("x-request-id", id.parse().unwrap());
    request
}

// Add timing header to every response
async fn add_timing(response: axum::response::Response) -> axum::response::Response {
    // Response transformations
    response
}

let app = Router::new()
    .route("/api/data", get(handler))
    .layer(map_request(add_request_id))
    .layer(map_response(add_timing));
```

### HandleErrorLayer — Making Fallible Services Infallible

Tower services can return errors, but axum requires infallible responses. `HandleErrorLayer` bridges this gap:

```rust
use tower::ServiceBuilder;
use axum::error_handling::HandleErrorLayer;

let app = Router::new()
    .route("/api/data", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(HandleErrorLayer::new(|err: tower::timeout::error::Elapsed| async move {
                (StatusCode::REQUEST_TIMEOUT, "Request timed out".to_string())
            }))
            .layer(tower::timeout::TimeoutLayer::new(Duration::from_secs(30)))
    );
```

### Error Handling

```rust
use axum::response::{IntoResponse, Response};
use serde_json::json;

#[derive(Debug, thiserror::Error)]
enum ApiError {
    #[error("not found")]
    NotFound,
    #[error("bad request: {0}")]
    BadRequest(String),
    #[error("unauthorized")]
    Unauthorized,
    #[error("conflict: {0}")]
    Conflict(String),
    #[error("internal error")]
    Internal(#[from] anyhow::Error),
    #[error("database error")]
    Database(#[from] sqlx::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, msg) = match &self {
            ApiError::NotFound => (StatusCode::NOT_FOUND, "not found"),
            ApiError::BadRequest(m) => (StatusCode::BAD_REQUEST, m.as_str()),
            ApiError::Unauthorized => (StatusCode::UNAUTHORIZED, "unauthorized"),
            ApiError::Conflict(m) => (StatusCode::CONFLICT, m.as_str()),
            ApiError::Internal(_) | ApiError::Database(_) => {
                // Log internal errors — don't expose details to client
                tracing::error!(error = %self);
                (StatusCode::INTERNAL_SERVER_ERROR, "internal error")
            }
        };
        (status, Json(json!({ "error": msg }))).into_response()
    }
}

// Now handlers can use Result<T, ApiError>
async fn create_user(
    State(state): State<Arc<AppState>>,
    Json(input): Json<CreateUserInput>,
) -> Result<(StatusCode, Json<UserResponse>), ApiError> {
    // sqlx errors auto-convert via #[from]
    let user = sqlx::query_as!(/* ... */)
        .fetch_one(&state.db)
        .await?;

    Ok((StatusCode::CREATED, Json(user)))
}
```

## Actix Web

### Basic Server Setup

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Responder};

async fn greet(req: actix_web::HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", name)
}

#[actix_web::main]  // NOT #[tokio::main] — Actix has its own runtime wrapper
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
    })
    .workers(4)
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### View Factory Pattern

Organize routes into factories for modular configuration:

```rust
use actix_web::web::{ServiceConfig, get, post, delete, patch, scope};

// Module-level factory
pub fn user_actions_factory(app: &mut ServiceConfig) {
    app.service(
        scope("/api/v1")
            .route("users", get().to(list_users))
            .route("users/{id}", get().to(get_user))
            .route("users", post().to(create_user))
            .route("users/{id}", delete().to(delete_user))
            .route("users/{id}", patch().to(update_user))
    );
}

pub fn auth_factory(app: &mut ServiceConfig) {
    app.service(
        scope("/auth")
            .route("login", post().to(login))
            .route("register", post().to(register))
    );
}

// Top-level factory chains module factories
pub fn views_factory(app: &mut ServiceConfig) {
    user_actions_factory(app);
    auth_factory(app);
}

// Main uses configure()
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let pool = create_pool().await;
    let state = web::Data::new(AppState { db: pool });

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .configure(views_factory)
    })
    .workers(4)
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### JSON Extraction

```rust
use actix_web::{web::Json, HttpResponse};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
pub struct CreateRequest {
    pub title: String,
    pub status: String,
}

#[derive(Serialize)]
pub struct ApiResponse {
    pub items: Vec<Item>,
}

// Json<T> automatically deserializes request body.
// Returns 400 Bad Request if deserialization fails.
pub async fn create(body: Json<CreateRequest>) -> HttpResponse {
    let item = body.into_inner();  // Extract owned value

    HttpResponse::Created().json(ApiResponse {
        items: vec![process(item)],
    })
}
```

### URL Parameter and Query Extraction

```rust
use actix_web::{HttpRequest, HttpResponse, web};

// Path parameter extraction
pub async fn get_by_name(req: HttpRequest) -> HttpResponse {
    let name = match req.match_info().get("name") {
        Some(name) => name,
        None => return HttpResponse::BadRequest().json("Name parameter required"),
    };

    match fetch_item(name).await {
        Ok(item) => HttpResponse::Ok().json(item),
        Err(e) => HttpResponse::NotFound().json(e.to_string()),
    }
}

// Using typed path extractor
pub async fn get_by_id(path: web::Path<(u64,)>) -> HttpResponse {
    let id = path.into_inner().0;
    // ...
    HttpResponse::Ok().finish()
}
```

### Custom Extractors (FromRequest)

Implement `FromRequest` for custom extraction logic:

```rust
use actix_web::{
    dev::Payload,
    FromRequest,
    HttpRequest,
};
use futures::future::{Ready, ok, err};

pub struct HeaderToken {
    pub token: String,
}

impl FromRequest for HeaderToken {
    type Error = NanoServiceError;
    type Future = Ready<Result<HeaderToken, NanoServiceError>>;

    fn from_request(req: &HttpRequest, _: &mut Payload) -> Self::Future {
        let token = match req.headers().get("Authorization") {
            Some(value) => {
                match value.to_str() {
                    Ok(s) => s.trim_start_matches("Bearer ").to_string(),
                    Err(_) => return err(NanoServiceError::new(
                        "Invalid token format".to_string(),
                        NanoServiceErrorStatus::Unauthorized,
                    )),
                }
            }
            None => return err(NanoServiceError::new(
                "Authorization header missing".to_string(),
                NanoServiceErrorStatus::Unauthorized,
            )),
        };

        ok(HeaderToken { token })
    }
}

// Use in handler — extraction runs automatically
pub async fn protected_endpoint(
    token: HeaderToken,
    body: Json<UpdateRequest>,
) -> HttpResponse {
    // token.token contains the extracted value
    // If extraction fails, handler never runs — error response returned
    HttpResponse::Ok().json("Success")
}
```

### ResponseError for Custom Errors

Implement `ResponseError` to convert errors to HTTP responses:

```rust
use actix_web::{HttpResponse, http::StatusCode, error::ResponseError};
use std::fmt;

#[derive(Debug)]
pub enum NanoServiceErrorStatus {
    NotFound,
    BadRequest,
    Unauthorized,
    Forbidden,
    Conflict,
    Unknown,
}

#[derive(Debug)]
pub struct NanoServiceError {
    pub message: String,
    pub status: NanoServiceErrorStatus,
}

impl NanoServiceError {
    pub fn new(message: String, status: NanoServiceErrorStatus) -> Self {
        Self { message, status }
    }
}

impl fmt::Display for NanoServiceError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.message)
    }
}

impl ResponseError for NanoServiceError {
    fn status_code(&self) -> StatusCode {
        match self.status {
            NanoServiceErrorStatus::NotFound => StatusCode::NOT_FOUND,
            NanoServiceErrorStatus::BadRequest => StatusCode::BAD_REQUEST,
            NanoServiceErrorStatus::Unauthorized => StatusCode::UNAUTHORIZED,
            NanoServiceErrorStatus::Forbidden => StatusCode::FORBIDDEN,
            NanoServiceErrorStatus::Conflict => StatusCode::CONFLICT,
            NanoServiceErrorStatus::Unknown => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }

    fn error_response(&self) -> HttpResponse {
        HttpResponse::build(self.status_code()).json(&self.message)
    }
}

// Now handlers can use Result<T, NanoServiceError>
pub async fn get_item(
    req: HttpRequest,
) -> Result<HttpResponse, NanoServiceError> {
    let name = req
        .match_info()
        .get("name")
        .ok_or(NanoServiceError::new(
            "Name required".to_string(),
            NanoServiceErrorStatus::BadRequest,
        ))?;

    let item = fetch_item(name).await?;  // Errors auto-convert to responses
    Ok(HttpResponse::Ok().json(item))
}
```

### Shared State (web::Data)

```rust
use actix_web::web::Data;
use std::sync::Arc;

pub struct AppState {
    pub db_pool: sqlx::PgPool,
    pub cache: Arc<Cache>,
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let state = Data::new(AppState {
        db_pool: create_pool().await,
        cache: Arc::new(Cache::new()),
    });

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())  // Shared across all workers
            .configure(views_factory)
    })
    .workers(4)
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

// Access in handlers
async fn handler(state: Data<AppState>) -> HttpResponse {
    let conn = state.db_pool.acquire().await.unwrap();
    // ...
    HttpResponse::Ok().finish()
}
```

## Rocket

### Basic Server Setup

```rust
#[macro_use]
extern crate rocket;

#[get("/")]
fn index() -> &'static str {
    "Hello, world!"
}

#[get("/<name>")]
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[launch]
fn rocket() -> _ {
    rocket::build().mount("/", routes![index, greet])
}
```

### JSON Handling

```rust
use rocket::serde::{json::Json, Deserialize, Serialize};

#[derive(Deserialize)]
#[serde(crate = "rocket::serde")]
pub struct CreateRequest<'r> {
    title: &'r str,
    status: &'r str,
}

#[derive(Serialize)]
#[serde(crate = "rocket::serde")]
pub struct ApiResponse {
    success: bool,
}

#[post("/create", data = "<body>")]
async fn create(body: Json<CreateRequest<'_>>) -> Json<ApiResponse> {
    // Process body.title, body.status
    Json(ApiResponse { success: true })
}
```

### Request Guards

```rust
use rocket::{
    request::{self, FromRequest, Request},
    http::Status,
};

pub struct AuthToken(String);

#[rocket::async_trait]
impl<'r> FromRequest<'r> for AuthToken {
    type Error = &'static str;

    async fn from_request(
        req: &'r Request<'_>,
    ) -> request::Outcome<Self, Self::Error> {
        match req.headers().get_one("Authorization") {
            Some(token) => request::Outcome::Success(
                AuthToken(token.trim_start_matches("Bearer ").to_string()),
            ),
            None => request::Outcome::Error((Status::Unauthorized, "Missing token")),
        }
    }
}

#[get("/protected")]
fn protected(token: AuthToken) -> &'static str {
    "Access granted"
}
```

### Database and State

```rust
use std::sync::Arc;

#[get("/users/<id>")]
async fn get_user(
    id: u64,
    db: &rocket::State<Arc<sqlx::PgPool>>,
) -> Result<Json<User>, Status> {
    match fetch_user(db, id).await {
        Ok(user) => Ok(Json(user)),
        Err(_) => Err(Status::NotFound),
    }
}

#[rocket::main]
async fn main() -> Result<(), rocket::Error> {
    let pool = create_pool().await;
    rocket::build()
        .manage(Arc::new(pool))
        .mount("/api", routes![get_user])
        .launch()
        .await?;
    Ok(())
}
```

## Hyper (Low-Level)

### Basic HTTP Server

```rust
use hyper::{body::Bytes, server::conn::http1, service::service_fn, Request, Response};
use hyper_util::rt::TokioIo;
use http_body_util::Full;
use std::convert::Infallible;
use tokio::net::TcpListener;

async fn handle(
    req: Request<hyper::body::Incoming>,
) -> Result<Response<Full<Bytes>>, Infallible> {
    println!("Request: {} {}", req.method(), req.uri());
    Ok(Response::new(Full::new(Bytes::from("Hello, World!"))))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (stream, _) = listener.accept().await?;
        let io = TokioIo::new(stream);

        tokio::spawn(async move {
            if let Err(err) = http1::Builder::new()
                .serve_connection(io, service_fn(handle))
                .await
            {
                eprintln!("Error: {:?}", err);
            }
        });
    }
}
```

### Manual Body Extraction

```rust
use hyper::{body::Incoming, Request};
use http_body_util::BodyExt;
use serde::de::DeserializeOwned;

pub async fn extract_json<T: DeserializeOwned>(
    req: Request<Incoming>,
) -> Result<T, Box<dyn std::error::Error>> {
    // Collect the full body (streaming → single buffer)
    let whole_body = req
        .collect()
        .await?
        .aggregate();

    // Deserialize from reader
    let parsed: T = serde_json::from_reader(whole_body.reader())?;
    Ok(parsed)
}
```

### Manual Routing

```rust
use hyper::{Method, Request, Response, StatusCode};

async fn router(
    req: Request<Incoming>,
) -> Result<Response<Full<Bytes>>, Infallible> {
    let path: Vec<&str> = req
        .uri()
        .path()
        .split('/')
        .filter(|s| !s.is_empty())
        .collect();

    match (req.method(), path.as_slice()) {
        (&Method::GET, ["api", "v1", "items"]) => get_all_items(req).await,
        (&Method::GET, ["api", "v1", "items", id]) => get_item(req, id).await,
        (&Method::POST, ["api", "v1", "items"]) => create_item(req).await,
        (&Method::DELETE, ["api", "v1", "items", id]) => delete_item(req, id).await,
        _ => Ok(Response::builder()
            .status(StatusCode::NOT_FOUND)
            .body(Full::new(Bytes::from("Not Found")))
            .unwrap()),
    }
}
```

## HTTP Method Semantics

| Method | Purpose | Idempotent | Safe | Has Body |
|--------|---------|------------|------|----------|
| GET | Retrieve resource | Yes | Yes | No (by convention) |
| POST | Create resource | No | No | Yes |
| PUT | Replace entire resource | Yes | No | Yes |
| PATCH | Partial update | No | No | Yes |
| DELETE | Remove resource | Yes | No | No (by convention) |
| OPTIONS | Describe communication options | Yes | Yes | No |
| HEAD | Like GET but no body | Yes | Yes | No |

## API Versioning

### URL Path Versioning (Recommended)

```rust
// Explicit in URL — easiest to reason about
pub fn v1_routes(app: &mut ServiceConfig) {
    app.service(
        scope("/api/v1")
            .route("/items", get().to(v1::list_items))
            .route("/items", post().to(v1::create_item)),
    );
}

pub fn v2_routes(app: &mut ServiceConfig) {
    app.service(
        scope("/api/v2")
            .route("/items", get().to(v2::list_items))  // New response format
            .route("/items", post().to(v2::create_item)),
    );
}

// Support both versions during migration
pub fn views_factory(app: &mut ServiceConfig) {
    v1_routes(app); // Deprecated but still supported
    v2_routes(app); // Current version
}

// Axum equivalent
fn app() -> Router {
    Router::new()
        .nest("/api/v1", v1_api_routes())
        .nest("/api/v2", v2_api_routes())
}
```

## Cross-Framework Consistency

Keep core logic framework-agnostic — wrap with thin framework adapters:

```rust
// Core function — no framework dependencies
pub async fn create_item_core(
    pool: &sqlx::PgPool,
    item: ItemInput,
) -> Result<Item, CoreError> {
    validate(&item)?;
    let saved = repository::save(pool, item).await?;
    Ok(saved)
}

// Actix wrapper
pub async fn create_actix(
    state: web::Data<AppState>,
    body: Json<ItemInput>,
) -> Result<HttpResponse, NanoServiceError> {
    let item = create_item_core(&state.db, body.into_inner()).await?;
    Ok(HttpResponse::Created().json(item))
}

// Axum wrapper
pub async fn create_axum(
    State(state): State<Arc<AppState>>,
    Json(body): Json<ItemInput>,
) -> impl IntoResponse {
    match create_item_core(&state.db, body).await {
        Ok(item) => (StatusCode::CREATED, Json(item)).into_response(),
        Err(e) => e.into_response(),
    }
}

// Rocket wrapper
#[post("/items", data = "<body>")]
pub async fn create_rocket(
    db: &rocket::State<Arc<sqlx::PgPool>>,
    body: Json<ItemInput>,
) -> Result<Json<Item>, Status> {
    create_item_core(db, body.into_inner())
        .await
        .map(Json)
        .map_err(|_| Status::BadRequest)
}
```

## Authentication

### Password Hashing with Argon2

Argon2 is the recommended algorithm for password hashing (winner of the Password Hashing Competition).

```toml
[dependencies]
argon2 = { version = "0.5", features = ["password-hash"] }
uuid = { version = "1.8", features = ["serde", "v4"] }
rand = "0.8"
```

```rust
use argon2::{Argon2, PasswordHasher, PasswordVerifier, password_hash::{SaltString, PasswordHash}};

/// Hash a password for storage
pub fn hash_password(password: &str) -> Result<String, argon2::password_hash::Error> {
    // Generate random salt
    let salt = SaltString::generate(&mut rand::thread_rng());

    // Hash with Argon2 (default parameters are secure)
    Ok(Argon2::default()
        .hash_password(password.as_bytes(), &salt)?
        .to_string())
}

/// Verify a password against a stored hash
pub fn verify_password(
    password: &str,
    hash: &str,
) -> Result<bool, argon2::password_hash::Error> {
    let parsed = PasswordHash::new(hash)?;
    Ok(Argon2::default()
        .verify_password(password.as_bytes(), &parsed)
        .is_ok())
}
```

#### Why Salt and Hash?

**Hashing** converts passwords to fixed-length strings:
- **Deterministic**: Same input always produces same hash
- **Irreversible**: Cannot reverse hash to get original password
- **Collision-resistant**: Different inputs produce different hashes

**Salting** adds a unique random value before hashing:
- **Uniqueness**: Same password produces different hashes for different users
- **Prevents rainbow tables**: Pre-computed hash tables become useless
- **Protection against brute-force**: Makes attacks computationally expensive per hash

### User Model with Password Hashing

```rust
use sqlx::FromRow;
use serde::{Serialize, Deserialize};

// Full user from database — includes password hash
#[derive(Debug, Clone, FromRow)]
pub struct User {
    pub id: i32,
    pub email: String,
    pub password: String, // Argon2 hash
    pub unique_id: String,
}

impl User {
    pub fn verify_password(&self, password: &str) -> Result<bool, AuthError> {
        verify_password(password, &self.password)
            .map_err(|e| AuthError::VerificationFailed(e.to_string()))
    }
}

// New user creation with automatic hashing
pub struct NewUser {
    pub email: String,
    pub password: String, // This will be the hash
    pub unique_id: String,
}

impl NewUser {
    pub fn new(email: String, password: String) -> Result<Self, AuthError> {
        let unique_id = uuid::Uuid::new_v4().to_string();
        let hash = hash_password(&password)
            .map_err(|e| AuthError::HashingFailed(e.to_string()))?;

        Ok(NewUser {
            email,
            password: hash,
            unique_id,
        })
    }
}

// TrimmedUser — never expose password hashes outside the auth service
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TrimmedUser {
    pub id: i32,
    pub email: String,
    pub unique_id: String,
    // No password field!
}

impl From<User> for TrimmedUser {
    fn from(user: User) -> Self {
        TrimmedUser {
            id: user.id,
            email: user.email,
            unique_id: user.unique_id,
        }
    }
}
```

### JWT Token Management

```toml
[dependencies]
jsonwebtoken = "9.3"
chrono = "0.4"
```

```rust
use jsonwebtoken::{
    encode, decode, Header, Algorithm,
    EncodingKey, DecodingKey, Validation,
};
use serde::{Serialize, Deserialize};

/// JWT claims — data encoded in the token
#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,  // Subject (user ID or unique_id)
    pub exp: i64,     // Expiration timestamp
    pub iat: i64,     // Issued at timestamp
}

impl Claims {
    pub fn new(user_id: String, duration_hours: i64) -> Self {
        let now = chrono::Utc::now();
        Self {
            sub: user_id,
            iat: now.timestamp(),
            exp: (now + chrono::Duration::hours(duration_hours)).timestamp(),
        }
    }

    pub fn is_expired(&self) -> bool {
        chrono::Utc::now().timestamp() > self.exp
    }
}

/// Create a JWT token
pub fn create_token(
    claims: &Claims,
    secret: &str,
) -> Result<String, jsonwebtoken::errors::Error> {
    encode(
        &Header::default(),
        claims,
        &EncodingKey::from_secret(secret.as_ref()),
    )
}

/// Verify and decode a JWT token
pub fn verify_token(
    token: &str,
    secret: &str,
) -> Result<Claims, jsonwebtoken::errors::Error> {
    let data = decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret.as_ref()),
        &Validation::new(Algorithm::HS256),
    )?;
    Ok(data.claims)
}
```

### Basic Auth Credential Extraction

For login endpoints using HTTP Basic Authentication:

```rust
use base64::{Engine, engine::general_purpose};

#[derive(Debug)]
pub struct BasicCredentials {
    pub email: String,
    pub password: String,
}

pub fn extract_basic_auth(
    authorization_header: &str,
) -> Result<BasicCredentials, AuthError> {
    // Check for "Basic " prefix
    if !authorization_header.starts_with("Basic ") {
        return Err(AuthError::InvalidFormat("Invalid authorization scheme".into()));
    }

    // Decode Base64
    let base64_credentials = &authorization_header[6..];
    let decoded = general_purpose::STANDARD
        .decode(base64_credentials)
        .map_err(|_| AuthError::InvalidFormat("Invalid Base64 encoding".into()))?;

    // Convert to string
    let credentials = String::from_utf8(decoded)
        .map_err(|_| AuthError::InvalidFormat("Invalid UTF-8 in credentials".into()))?;

    // Split on first colon (password may contain colons)
    let parts: Vec<&str> = credentials.splitn(2, ':').collect();
    if parts.len() != 2 {
        return Err(AuthError::InvalidFormat("Invalid credentials format".into()));
    }

    Ok(BasicCredentials {
        email: parts[0].to_string(),
        password: parts[1].to_string(),
    })
}
```

### Complete Login Flow

```rust
// Core login function — framework-agnostic
pub async fn login_core(
    pool: &sqlx::PgPool,
    email: &str,
    password: &str,
    jwt_secret: &str,
) -> Result<String, AuthError> {
    // Get user from database
    let user = sqlx::query_as!(
        User,
        "SELECT id, email, password, unique_id FROM users WHERE email = $1",
        email,
    )
    .fetch_optional(pool)
    .await
    .map_err(|e| AuthError::DatabaseError(e.to_string()))?
    .ok_or(AuthError::InvalidCredentials)?;

    // Verify password
    if !user.verify_password(password)? {
        return Err(AuthError::InvalidCredentials);
    }

    // Generate JWT with unique_id
    let claims = Claims::new(user.unique_id, 24); // 24-hour token
    let token = create_token(&claims, jwt_secret)
        .map_err(|e| AuthError::TokenError(e.to_string()))?;

    Ok(token)
}

// Axum handler
async fn login(
    State(state): State<Arc<AppState>>,
    Json(input): Json<LoginInput>,
) -> Result<Json<TokenResponse>, ApiError> {
    let token = login_core(&state.db, &input.email, &input.password, &state.jwt_secret)
        .await
        .map_err(|_| ApiError::Unauthorized)?;

    Ok(Json(TokenResponse { token }))
}

// Actix handler using Basic Auth
async fn login_basic(
    req: actix_web::HttpRequest,
    state: web::Data<AppState>,
) -> Result<HttpResponse, NanoServiceError> {
    let auth_header = req
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or(NanoServiceError::new(
            "Missing Authorization header".into(),
            NanoServiceErrorStatus::Unauthorized,
        ))?;

    let credentials = extract_basic_auth(auth_header)?;
    let token = login_core(
        &state.db_pool,
        &credentials.email,
        &credentials.password,
        &state.jwt_secret,
    )
    .await?;

    Ok(HttpResponse::Ok().json(token))
}
```

### Database Transaction for User Creation

```rust
use sqlx::PgPool;

pub async fn create_user(
    pool: &PgPool,
    email: String,
    password: String,
) -> Result<User, AuthError> {
    // Create new user with hashed password
    let new_user = NewUser::new(email, password)?;

    // Insert into database
    let user = sqlx::query_as::<_, User>(
        r#"
        INSERT INTO users (email, password, unique_id)
        VALUES ($1, $2, $3)
        RETURNING id, email, password, unique_id
        "#,
    )
    .bind(&new_user.email)
    .bind(&new_user.password)
    .bind(&new_user.unique_id)
    .fetch_one(pool)
    .await
    .map_err(|e| AuthError::DatabaseError(e.to_string()))?;

    Ok(user)
}
```

### User Session Schema (SQL)

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR NOT NULL UNIQUE,
    password VARCHAR NOT NULL,     -- Argon2 hash
    unique_id VARCHAR NOT NULL UNIQUE  -- For JWT and session management
);

-- Indexes for login lookups
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_unique_id ON users(unique_id);
```

### Auth Error Type

```rust
#[derive(Debug, thiserror::Error)]
pub enum AuthError {
    #[error("Password hashing failed: {0}")]
    HashingFailed(String),
    #[error("Password verification failed: {0}")]
    VerificationFailed(String),
    #[error("Invalid credentials")]
    InvalidCredentials,
    #[error("Token error: {0}")]
    TokenError(String),
    #[error("Invalid format: {0}")]
    InvalidFormat(String),
    #[error("Database error: {0}")]
    DatabaseError(String),
}
```

### FromRequest Implementation for Token Extraction

#### Actix Web

```rust
use actix_web::{
    dev::Payload,
    FromRequest,
    HttpRequest,
};
use futures::future::{Ready, ok, err};

impl FromRequest for HeaderToken {
    type Error = NanoServiceError;
    type Future = Ready<Result<HeaderToken, NanoServiceError>>;

    fn from_request(req: &HttpRequest, _: &mut Payload) -> Self::Future {
        let token_str = match req.headers().get("token") {
            Some(value) => match value.to_str() {
                Ok(s) => s,
                Err(_) => return err(NanoServiceError::new(
                    "Invalid token format".into(),
                    NanoServiceErrorStatus::Unauthorized,
                )),
            },
            None => return err(NanoServiceError::new(
                "Token not found in header".into(),
                NanoServiceErrorStatus::Unauthorized,
            )),
        };

        // Decode the JWT
        match verify_token(token_str, &get_jwt_secret()) {
            Ok(claims) => ok(HeaderToken {
                unique_id: claims.sub,
            }),
            Err(_) => err(NanoServiceError::new(
                "Invalid token".into(),
                NanoServiceErrorStatus::Unauthorized,
            )),
        }
    }
}

// Usage in handler — token automatically extracted
pub async fn protected_endpoint(
    token: HeaderToken,
    body: actix_web::web::Json<RequestBody>,
) -> Result<HttpResponse, NanoServiceError> {
    // token.unique_id is available
    // If extraction fails, handler never runs
    Ok(HttpResponse::Ok().json("Success"))
}
```

#### Axum

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

pub struct HeaderToken {
    pub unique_id: String,
}

impl<S: Send + Sync> FromRequestParts<S> for HeaderToken {
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        let token_str = parts
            .headers
            .get("Authorization")
            .and_then(|v| v.to_str().ok())
            .map(|s| s.trim_start_matches("Bearer "))
            .ok_or((StatusCode::UNAUTHORIZED, "Missing token"))?;

        let claims = verify_token(token_str, &get_jwt_secret())
            .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid token"))?;

        Ok(HeaderToken {
            unique_id: claims.sub,
        })
    }
}
```

## Pagination

```rust
#[derive(Deserialize)]
struct PaginationParams {
    #[serde(default = "default_page")]
    page: u32,
    #[serde(default = "default_limit")]
    limit: u32,
}

fn default_page() -> u32 { 1 }
fn default_limit() -> u32 { 20 }

impl PaginationParams {
    fn offset(&self) -> u32 {
        (self.page.saturating_sub(1)) * self.limit
    }
    fn limit(&self) -> u32 {
        self.limit.min(100) // Cap at 100 items per page
    }
}

#[derive(Serialize)]
struct Paginated<T: Serialize> {
    items: Vec<T>,
    page: u32,
    total: u64,
    has_next: bool,
}

async fn list_users(
    State(state): State<Arc<AppState>>,
    Query(params): Query<PaginationParams>,
) -> Result<Json<Paginated<UserResponse>>, ApiError> {
    let total: (i64,) = sqlx::query_as("SELECT COUNT(*) FROM users")
        .fetch_one(&state.db)
        .await?;

    let users = sqlx::query_as!(
        UserResponse,
        "SELECT id, username, email FROM users ORDER BY id LIMIT $1 OFFSET $2",
        params.limit() as i64,
        params.offset() as i64,
    )
    .fetch_all(&state.db)
    .await?;

    Ok(Json(Paginated {
        has_next: (params.offset() as u64 + users.len() as u64) < total.0 as u64,
        items: users,
        page: params.page,
        total: total.0 as u64,
    }))
}
```

## reqwest HTTP Client

**Key principle:** Create one `Client` and reuse it. `Client` already uses `Arc` internally — cloning is cheap. Each `Client` maintains its own connection pool, so creating one per request wastes connections.

### ClientBuilder Configuration

```rust
use reqwest::Client;
use std::time::Duration;

// Production-grade client configuration
let client = Client::builder()
    // Timeouts (always set all three)
    .timeout(Duration::from_secs(30))              // Total request deadline
    .connect_timeout(Duration::from_secs(5))       // Connection phase only
    .read_timeout(Duration::from_secs(10))         // Per-read operation

    // Connection pool
    .pool_idle_timeout(Duration::from_secs(90))    // How long to keep idle connections
    .pool_max_idle_per_host(10)                    // Max idle connections per host

    // TCP tuning
    .tcp_keepalive(Duration::from_secs(15))        // Detect dead connections
    .tcp_nodelay(true)                             // Disable Nagle (default: true)

    // Defaults
    .default_headers({
        let mut headers = reqwest::header::HeaderMap::new();
        headers.insert("User-Agent", "my-service/1.0".parse().unwrap());
        headers
    })

    // Redirect policy
    .redirect(reqwest::redirect::Policy::limited(5))  // Max 5 redirects (default: 10)

    // Compression (enabled by default with features)
    .gzip(true)
    .brotli(true)

    .build()
    .expect("Failed to build HTTP client");
```

### API Client Wrapper

```rust
pub struct ApiClient {
    client: reqwest::Client,
    base_url: String,
}

impl ApiClient {
    pub fn new(base_url: String) -> Self {
        let client = reqwest::Client::builder()
            .timeout(Duration::from_secs(30))
            .connect_timeout(Duration::from_secs(5))
            .pool_idle_timeout(Duration::from_secs(90))
            .tcp_keepalive(Duration::from_secs(15))
            .build()
            .unwrap();
        Self { client, base_url }
    }

    pub async fn get<T: serde::de::DeserializeOwned>(
        &self,
        path: &str,
    ) -> Result<T, reqwest::Error> {
        self.client
            .get(format!("{}{}", self.base_url, path))
            .send()
            .await?
            .error_for_status()?
            .json()
            .await
    }

    pub async fn post<B: serde::Serialize, T: serde::de::DeserializeOwned>(
        &self,
        path: &str,
        body: &B,
    ) -> Result<T, reqwest::Error> {
        self.client
            .post(format!("{}{}", self.base_url, path))
            .json(body)
            .send()
            .await?
            .error_for_status()?
            .json()
            .await
    }

    pub async fn delete(&self, path: &str) -> Result<(), reqwest::Error> {
        self.client
            .delete(format!("{}{}", self.base_url, path))
            .send()
            .await?
            .error_for_status()?;
        Ok(())
    }
}
```

## CORS Configuration

### Axum (Production)

```rust
use tower_http::cors::{CorsLayer, AllowOrigin, AllowMethods, AllowHeaders};
use axum::http::{Method, HeaderName};

// Production: restrict origins
let cors = CorsLayer::new()
    .allow_origin(AllowOrigin::exact("https://myapp.com".parse().unwrap()))
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers([
        HeaderName::from_static("content-type"),
        HeaderName::from_static("authorization"),
    ])
    .max_age(std::time::Duration::from_secs(3600));

let app = Router::new()
    .route("/", get(index))
    .layer(cors);

// Development: allow anything
let cors_dev = CorsLayer::permissive();
```

### Actix Web

```rust
use actix_cors::Cors;
use actix_web::http::header;

// Development
let cors = Cors::default()
    .allow_any_origin()
    .allow_any_method()
    .allow_any_header();

// Production
let cors = Cors::default()
    .allowed_origin("https://myapp.com")
    .allowed_methods(vec!["GET", "POST", "PUT", "DELETE"])
    .allowed_headers(vec![header::AUTHORIZATION, header::CONTENT_TYPE])
    .max_age(3600);

HttpServer::new(|| {
    App::new()
        .wrap(cors)
        .configure(views_factory)
});
```

### Rocket (Custom Fairing)

```rust
use rocket::fairing::{Fairing, Info, Kind};
use rocket::http::Header;
use rocket::{Request, Response};

pub struct CORS;

#[rocket::async_trait]
impl Fairing for CORS {
    fn info(&self) -> Info {
        Info {
            name: "Add CORS headers",
            kind: Kind::Response,
        }
    }

    async fn on_response<'r>(
        &self,
        _request: &'r Request<'_>,
        response: &mut Response<'r>,
    ) {
        response.set_header(Header::new("Access-Control-Allow-Origin", "*"));
        response.set_header(Header::new(
            "Access-Control-Allow-Methods",
            "POST, GET, PUT, DELETE, OPTIONS",
        ));
        response.set_header(Header::new("Access-Control-Allow-Headers", "*"));
        response.set_header(Header::new("Access-Control-Allow-Credentials", "true"));
    }
}

// Attach to server
let rocket = rocket::build()
    .mount("/", routes![/* ... */])
    .attach(CORS);
```

### Hyper (Manual Headers)

```rust
use hyper::{Response, header};

fn add_cors_headers<T>(mut response: Response<T>) -> Response<T> {
    let headers = response.headers_mut();
    headers.insert(
        header::ACCESS_CONTROL_ALLOW_ORIGIN,
        "*".parse().unwrap(),
    );
    headers.insert(
        header::ACCESS_CONTROL_ALLOW_METHODS,
        "GET, POST, PUT, DELETE, OPTIONS".parse().unwrap(),
    );
    headers.insert(
        header::ACCESS_CONTROL_ALLOW_HEADERS,
        "*".parse().unwrap(),
    );
    response
}

// Apply to every response in your handler
async fn handle(
    req: Request<Incoming>,
) -> Result<Response<Full<Bytes>>, Infallible> {
    let response = // ... build response
    Ok(add_cors_headers(response))
}
```

## WebSockets

### When to Use WebSockets

| Use Case | Example |
|----------|---------|
| Push notifications | Live alerts, system events |
| Real-time updates | Chat, collaborative editing |
| Data streaming | Stock prices, sensor data |
| Gaming | Multiplayer state sync |
| Task completion | Notify when background job finishes |

### WebSocket Flow

```
Client                    Server
   |                         |
   |--- HTTP Upgrade ------->|
   |<-- 101 Switching -------|
   |                         |
   |<====== Messages =======>|  (bidirectional, persistent)
   |                         |
   |--- Close Frame -------->|
   |<-- Close Frame ---------|
```

### Axum WebSocket

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::IntoResponse,
    routing::get,
    Router,
};

async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        match msg {
            Ok(Message::Text(text)) => {
                if socket.send(Message::Text(text)).await.is_err() {
                    break;
                }
            }
            Ok(Message::Binary(bin)) => {
                if socket.send(Message::Binary(bin)).await.is_err() {
                    break;
                }
            }
            Ok(Message::Ping(ping)) => {
                if socket.send(Message::Pong(ping)).await.is_err() {
                    break;
                }
            }
            Ok(Message::Close(_)) | Err(_) => break,
            _ => {}
        }
    }
}

fn app() -> Router {
    Router::new().route("/ws", get(ws_handler))
}
```

### Actix WebSocket with actix-ws

```rust
use actix_web::{web, rt, HttpRequest, HttpResponse, Error};
use actix_ws::AggregatedMessage;
use futures::StreamExt;

async fn websocket_handler(
    req: HttpRequest,
    stream: web::Payload,
) -> Result<HttpResponse, Error> {
    // Upgrade HTTP connection to WebSocket
    let (res, mut session, stream) = actix_ws::handle(&req, stream)?;

    // Configure continuation handling
    let mut stream = stream
        .aggregate_continuations()
        .max_continuation_size(2_usize.pow(20)); // 1MB max

    // Spawn handler task
    rt::spawn(async move {
        while let Some(msg) = stream.next().await {
            match msg {
                Ok(AggregatedMessage::Text(text)) => {
                    session.text(text).await.unwrap();
                }
                Ok(AggregatedMessage::Binary(bin)) => {
                    session.binary(bin).await.unwrap();
                }
                Ok(AggregatedMessage::Ping(msg)) => {
                    session.pong(&msg).await.unwrap();
                }
                Ok(AggregatedMessage::Close(_)) => break,
                _ => {}
            }
        }
    });

    Ok(res)
}

// Route configuration
fn configure(cfg: &mut web::ServiceConfig) {
    cfg.route("/ws", web::get().to(websocket_handler));
}
```

Dependencies:
```toml
[dependencies]
actix-web = "4.9"
actix-ws = "0.3"
futures = "0.3"
```

### tokio-tungstenite (Framework-Independent)

For applications without a web framework:

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio_tungstenite::{accept_async, tungstenite::Message};
use futures::{StreamExt, SinkExt};

async fn handle_connection(stream: TcpStream) {
    let ws_stream = accept_async(stream).await.unwrap();
    let (mut write, mut read) = ws_stream.split();

    while let Some(msg) = read.next().await {
        match msg {
            Ok(Message::Text(text)) => {
                write.send(Message::Text(text)).await.unwrap();
            }
            Ok(Message::Binary(bin)) => {
                write.send(Message::Binary(bin)).await.unwrap();
            }
            Ok(Message::Close(_)) => break,
            _ => {}
        }
    }
}

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:9000").await.unwrap();

    while let Ok((stream, _)) = listener.accept().await {
        tokio::spawn(handle_connection(stream));
    }
}
```

Dependencies:
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = "0.21"
futures = "0.3"
```

### WebSocket with Shared State (Chat Room)

```rust
use std::sync::Arc;
use tokio::sync::broadcast;
use futures::{SinkExt, StreamExt};

struct ChatState {
    broadcast: broadcast::Sender<String>,
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<Arc<ChatState>>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}

async fn handle_socket(mut socket: WebSocket, state: Arc<ChatState>) {
    let mut rx = state.broadcast.subscribe();
    let (mut sink, mut stream) = socket.split();

    tokio::select! {
        // Forward broadcasts to this client
        _ = async {
            while let Ok(msg) = rx.recv().await {
                if sink.send(Message::Text(msg)).await.is_err() {
                    break;
                }
            }
        } => {}
        // Forward client messages to broadcast
        _ = async {
            while let Some(Ok(Message::Text(text))) = stream.next().await {
                let _ = state.broadcast.send(text);
            }
        } => {}
    }
}
```

### Client JavaScript

```javascript
const ws = new WebSocket('ws://localhost:8080/ws');

ws.onopen = () => {
    console.log('Connected');
    ws.send('Hello from client');
};

ws.onmessage = (event) => {
    console.log('Received:', event.data);
};

ws.onclose = () => console.log('Disconnected');
ws.onerror = (err) => console.error('Error:', err);
```

### Authenticated WebSockets

Validate the user's JWT **before** upgrading the HTTP connection to WebSocket. Once upgraded, there's no HTTP layer to reject unauthorized users.

```rust
use axum::{
    extract::{ws::{WebSocket, WebSocketUpgrade}, State, Query},
    response::IntoResponse,
};

#[derive(Deserialize)]
struct WsParams {
    token: String,  // JWT passed as query param (WebSocket API can't set headers)
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    Query(params): Query<WsParams>,
    State(state): State<Arc<AppState>>,
) -> Result<impl IntoResponse, ApiError> {
    // Validate JWT BEFORE upgrade — reject unauthorized with HTTP 401
    let claims = state.jwt_service.validate(&params.token)
        .map_err(|_| ApiError::Unauthorized)?;

    let user_id = claims.sub;
    Ok(ws.on_upgrade(move |socket| handle_authenticated_socket(socket, state, user_id)))
}

async fn handle_authenticated_socket(
    socket: WebSocket,
    state: Arc<AppState>,
    user_id: UserId,
) {
    // Now user_id is known — attribute messages, enforce permissions
    let (mut sink, mut stream) = socket.split();

    while let Some(Ok(Message::Text(text))) = stream.next().await {
        let msg = ChatMessage {
            sender: user_id,
            content: text,
            sent_at: Utc::now(),
        };
        let _ = state.broadcast.send(msg);
    }
}
```

**Why query param for token?** The browser WebSocket API (`new WebSocket(url)`) doesn't support custom headers. The standard workaround is `ws://host/ws?token=<jwt>`. For native clients, subprotocol headers work too.

```javascript
// Client: pass JWT in URL
const token = localStorage.getItem('jwt');
const ws = new WebSocket(`ws://localhost:8080/ws?token=${token}`);
```

### Multi-Room Chat with Persistence

Real chat apps need multiple rooms, message attribution, and history loading.

**Room management with DashMap:**

```rust
use dashmap::DashMap;
use tokio::sync::broadcast;
use uuid::Uuid;

type RoomId = Uuid;

#[derive(Clone, Debug, serde::Serialize, serde::Deserialize)]
pub struct ChatMessage {
    pub id: Uuid,
    pub room_id: RoomId,
    pub sender_id: UserId,
    pub sender_name: String,
    pub content: String,
    pub sent_at: chrono::DateTime<chrono::Utc>,
}

pub struct ChatState {
    rooms: DashMap<RoomId, broadcast::Sender<ChatMessage>>,
    pool: sqlx::PgPool,
}

impl ChatState {
    pub fn get_or_create_room(&self, room_id: RoomId) -> broadcast::Sender<ChatMessage> {
        self.rooms
            .entry(room_id)
            .or_insert_with(|| broadcast::channel(256).0)
            .clone()
    }

    pub fn subscribe(&self, room_id: RoomId) -> broadcast::Receiver<ChatMessage> {
        self.get_or_create_room(room_id).subscribe()
    }
}
```

**Save-then-broadcast pattern — persist before delivering:**

```rust
async fn handle_room_socket(
    socket: WebSocket,
    state: Arc<ChatState>,
    room_id: RoomId,
    user: AuthUser,
) {
    let tx = state.get_or_create_room(room_id);
    let mut rx = tx.subscribe();
    let (mut sink, mut stream) = socket.split();

    tokio::select! {
        // Outbound: forward room messages to this client
        _ = async {
            while let Ok(msg) = rx.recv().await {
                let json = serde_json::to_string(&msg).unwrap();
                if sink.send(Message::Text(json)).await.is_err() {
                    break;
                }
            }
        } => {}
        // Inbound: persist then broadcast
        _ = async {
            while let Some(Ok(Message::Text(text))) = stream.next().await {
                let msg = ChatMessage {
                    id: Uuid::new_v4(),
                    room_id,
                    sender_id: user.id,
                    sender_name: user.name.clone(),
                    content: text,
                    sent_at: chrono::Utc::now(),
                };
                // Persist FIRST — if DB write fails, don't broadcast
                if let Err(e) = save_message(&state.pool, &msg).await {
                    tracing::error!(error = %e, "failed to persist message");
                    continue;  // Skip broadcast on persistence failure
                }
                let _ = tx.send(msg);
            }
        } => {}
    }
}

async fn save_message(pool: &sqlx::PgPool, msg: &ChatMessage) -> Result<(), sqlx::Error> {
    sqlx::query!(
        "INSERT INTO messages (id, room_id, sender_id, content, sent_at) VALUES ($1, $2, $3, $4, $5)",
        msg.id, msg.room_id, msg.sender_id.0, msg.content, msg.sent_at
    )
    .execute(pool)
    .await?;
    Ok(())
}
```

**Load chat history on connect (cursor-based pagination):**

```rust
/// Load messages before a cursor timestamp, newest first, limited to page_size
async fn load_history(
    pool: &sqlx::PgPool,
    room_id: RoomId,
    before: Option<chrono::DateTime<chrono::Utc>>,
    page_size: i64,
) -> Result<Vec<ChatMessage>, sqlx::Error> {
    let before = before.unwrap_or(chrono::Utc::now());
    sqlx::query_as!(
        ChatMessage,
        r#"SELECT id, room_id, sender_id, sender_name, content, sent_at
           FROM messages
           WHERE room_id = $1 AND sent_at < $2
           ORDER BY sent_at DESC
           LIMIT $3"#,
        room_id, before, page_size
    )
    .fetch_all(pool)
    .await
}

// REST endpoint for history (called before WebSocket connect)
async fn get_history(
    Path(room_id): Path<RoomId>,
    Query(params): Query<HistoryParams>,
    State(state): State<Arc<ChatState>>,
) -> Result<Json<Vec<ChatMessage>>, ApiError> {
    let messages = load_history(&state.pool, room_id, params.before, params.limit.unwrap_or(50))
        .await?;
    Ok(Json(messages))
}

// Router setup
fn chat_routes() -> Router<Arc<AppState>> {
    Router::new()
        .route("/rooms/{room_id}/ws", get(ws_room_handler))
        .route("/rooms/{room_id}/messages", get(get_history))
}
```

**Why cursor-based (not offset)?** Chat history is append-heavy. With offset pagination, new messages shift all offsets, causing duplicates or gaps. Cursor pagination (`WHERE sent_at < $cursor`) is stable regardless of new inserts.

### WebSocket Heartbeat & Reconnection

Detect dead connections with periodic ping/pong and clean up resources.

**Server-side ping interval:**

```rust
use tokio::time::{interval, Duration};

async fn handle_socket_with_heartbeat(mut socket: WebSocket, state: Arc<AppState>) {
    let mut heartbeat = interval(Duration::from_secs(30));
    let (mut sink, mut stream) = socket.split();
    let mut last_pong = Instant::now();

    loop {
        tokio::select! {
            // Send ping every 30 seconds
            _ = heartbeat.tick() => {
                if last_pong.elapsed() > Duration::from_secs(90) {
                    tracing::warn!("client unresponsive, closing");
                    break;  // No pong in 3 intervals — connection is dead
                }
                if sink.send(Message::Ping(vec![].into())).await.is_err() {
                    break;
                }
            }
            // Handle incoming messages
            msg = stream.next() => {
                match msg {
                    Some(Ok(Message::Pong(_))) => {
                        last_pong = Instant::now();
                    }
                    Some(Ok(Message::Text(text))) => {
                        // Handle application message
                        handle_text_message(&text, &state).await;
                    }
                    Some(Ok(Message::Close(_))) | None | Some(Err(_)) => break,
                    _ => {}
                }
            }
        }
    }
    // Clean up: remove from connection table, notify room of departure
    state.connection_table.release(&user_id);
}
```

**Missed message delivery on reconnect:**

When a client reconnects, it sends its last-seen message timestamp. The server loads missed messages from the database and sends them before subscribing to the live broadcast:

```rust
async fn handle_reconnect(
    socket: &mut WebSocket,
    pool: &PgPool,
    room_id: RoomId,
    last_seen: chrono::DateTime<chrono::Utc>,
) -> Result<(), ApiError> {
    // Load messages the client missed while disconnected
    let missed = sqlx::query_as!(
        ChatMessage,
        "SELECT * FROM messages WHERE room_id = $1 AND sent_at > $2 ORDER BY sent_at ASC",
        room_id, last_seen
    )
    .fetch_all(pool)
    .await?;

    // Replay missed messages before subscribing to live stream
    for msg in missed {
        let json = serde_json::to_string(&msg).unwrap();
        socket.send(Message::Text(json)).await
            .map_err(|_| ApiError::Internal)?;
    }
    Ok(())
}
```

**Client reconnection with exponential backoff:**

```javascript
class ReconnectingWebSocket {
    constructor(url) {
        this.url = url;
        this.baseDelay = 1000;
        this.maxDelay = 30000;
        this.attempt = 0;
        this.lastSeen = null;
        this.connect();
    }

    connect() {
        const url = this.lastSeen
            ? `${this.url}&last_seen=${this.lastSeen}`
            : this.url;
        this.ws = new WebSocket(url);

        this.ws.onopen = () => { this.attempt = 0; };
        this.ws.onmessage = (e) => {
            const msg = JSON.parse(e.data);
            this.lastSeen = msg.sent_at;
            this.onMessage(msg);
        };
        this.ws.onclose = () => {
            const delay = Math.min(this.baseDelay * 2 ** this.attempt, this.maxDelay);
            const jitter = delay * (0.5 + Math.random() * 0.5);
            this.attempt++;
            setTimeout(() => this.connect(), jitter);
        };
    }
}
```

### WebRTC Signaling Server

For peer-to-peer video/audio streaming (webcam, screen share), the Rust server acts as a **signaling server** — it relays connection metadata between peers but never touches the actual media stream.

**Architecture decision:**

| Approach | Server Role | When to Use |
|----------|------------|-------------|
| **Peer-to-peer (P2P)** | Relay SDP + ICE only | 1:1 video calls, small groups (≤4) |
| **SFU (Selective Forwarding)** | Receives + forwards media | Group calls (5-50), recording |
| **MCU (Multipoint Control)** | Decodes + re-encodes | Large rooms, transcoding needs |

For most apps: **start with P2P + signaling server.** The browser handles all media (capture, encode, transmit) via its built-in WebRTC stack. The Rust server only forwards signaling messages over WebSocket.

**Signaling flow:**

```
Caller                   Signaling Server              Callee
  |                           |                           |
  |-- Offer (SDP) ---------->|                           |
  |                           |-- Offer (SDP) ---------->|
  |                           |                           |
  |                           |<-- Answer (SDP) ---------|
  |<-- Answer (SDP) ---------|                           |
  |                           |                           |
  |-- ICE Candidate -------->|                           |
  |                           |-- ICE Candidate -------->|
  |                           |<-- ICE Candidate ---------|
  |<-- ICE Candidate ---------|                           |
  |                           |                           |
  |<============= P2P Media Stream =====================>|
  |  (direct, bypasses server — UDP/SRTP)                |
```

**Signaling protocol messages:**

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum SignalingMessage {
    /// SDP offer from caller
    Offer { sdp: String, target_user: UserId },
    /// SDP answer from callee
    Answer { sdp: String, target_user: UserId },
    /// ICE candidate for NAT traversal
    IceCandidate { candidate: String, sdp_mid: Option<String>, target_user: UserId },
    /// Call initiation
    CallRequest { target_user: UserId, media_type: MediaType },
    /// Call response
    CallResponse { accepted: bool, target_user: UserId },
    /// Hang up
    HangUp { target_user: UserId },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum MediaType { Audio, Video, ScreenShare }
```

**Signaling server — route messages between users via WebSocket:**

```rust
use dashmap::DashMap;
use tokio::sync::mpsc;

/// Maps user IDs to their WebSocket sender
type UserConnections = DashMap<UserId, mpsc::UnboundedSender<SignalingMessage>>;

pub struct SignalingState {
    connections: UserConnections,
}

async fn handle_signaling_socket(
    socket: WebSocket,
    state: Arc<SignalingState>,
    user_id: UserId,
) {
    let (mut sink, mut stream) = socket.split();
    let (tx, mut rx) = mpsc::unbounded_channel::<SignalingMessage>();

    // Register this user's sender
    state.connections.insert(user_id, tx);

    tokio::select! {
        // Forward signaling messages TO this user
        _ = async {
            while let Some(msg) = rx.recv().await {
                let json = serde_json::to_string(&msg).unwrap();
                if sink.send(Message::Text(json)).await.is_err() {
                    break;
                }
            }
        } => {}
        // Route signaling messages FROM this user to the target
        _ = async {
            while let Some(Ok(Message::Text(text))) = stream.next().await {
                if let Ok(msg) = serde_json::from_str::<SignalingMessage>(&text) {
                    let target = match &msg {
                        SignalingMessage::Offer { target_user, .. }
                        | SignalingMessage::Answer { target_user, .. }
                        | SignalingMessage::IceCandidate { target_user, .. }
                        | SignalingMessage::CallRequest { target_user, .. }
                        | SignalingMessage::CallResponse { target_user, .. }
                        | SignalingMessage::HangUp { target_user } => target_user,
                    };
                    if let Some(target_tx) = state.connections.get(target) {
                        let _ = target_tx.send(msg);
                    }
                }
            }
        } => {}
    }

    // Cleanup on disconnect
    state.connections.remove(&user_id);
}
```

**STUN/TURN configuration:**

The server doesn't handle STUN/TURN directly — these are separate infrastructure. The client needs STUN to discover its public IP and TURN as a relay fallback when direct P2P fails (corporate firewalls, symmetric NAT).

```javascript
// Client WebRTC setup
const pc = new RTCPeerConnection({
    iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },           // Free public STUN
        {
            urls: 'turn:turn.example.com:3478',              // Self-hosted TURN
            username: 'user',
            credential: 'pass',
        },
    ],
});

// Get webcam stream
const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
stream.getTracks().forEach(track => pc.addTrack(track, stream));

// Create and send offer via signaling WebSocket
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);
signalingWs.send(JSON.stringify({ type: 'Offer', sdp: offer.sdp, target_user: calleeId }));
```

**Rust crates for WebRTC:**
- **Signaling only** (recommended for P2P): No special crate needed — just WebSocket or HTTP message routing as shown above
- **`str0m`** (recommended for server-side): Sans-I/O WebRTC library — no internal threads, no async runtime dependency. Used by [atm0s-media-server](https://github.com/8xFF/atm0s-media-server) (production decentralized SFU). Sans-I/O design makes protocol logic testable without network I/O (see [async-concurrency.md](async-concurrency.md#sans-io-pattern))
- **`webrtc-rs`** (`webrtc = "0.11"`): Full async WebRTC stack — more batteries-included than `str0m` but couples to tokio runtime. Good for quick prototypes
- **TURN server**: Use `coturn` (C, battle-tested) for production TURN relay — don't build your own

### WHIP/WHEP — HTTP-Based WebRTC Signaling

An alternative to WebSocket signaling: use standard HTTP POST endpoints for SDP/ICE exchange. This is the model used by production systems like atm0s-media-server and is now standardized (WHIP: RFC 9725, WHEP: draft).

**Why HTTP over WebSocket for signaling?**
- Stateless — no persistent connection to manage for signaling
- Works through CDNs, load balancers, and API gateways without special WebSocket support
- Easier to secure (standard HTTP auth middleware)
- Signaling is request-response, not streaming — HTTP fits the pattern naturally

```rust
use axum::{extract::{Path, State}, Json, routing::post, Router};

#[derive(Deserialize)]
struct ConnectRequest {
    sdp: String,        // SDP offer from client
    token: String,      // JWT for authentication
}

#[derive(Serialize)]
struct ConnectResponse {
    sdp: String,        // SDP answer from server
    conn_id: String,    // Connection ID for subsequent ICE/restart calls
    ice_lite: bool,     // Whether server uses ICE lite
}

#[derive(Deserialize)]
struct IceCandidateRequest {
    candidate: String,
    sdp_mid: Option<String>,
    sdp_m_line_index: Option<u16>,
}

// POST /webrtc/connect — initial SDP offer/answer exchange
async fn connect(
    State(state): State<Arc<MediaState>>,
    Json(req): Json<ConnectRequest>,
) -> Result<Json<ConnectResponse>, ApiError> {
    let claims = state.secure.decode_token(&req.token)
        .map_err(|_| ApiError::Forbidden("invalid token"))?;

    let conn_id = generate_connection_id();
    let (tx, rx) = tokio::sync::oneshot::channel();

    // Send to media core via channel, await SDP answer
    state.rpc_tx.send(RpcReq::Connect { sdp: req.sdp, conn_id, reply: tx })
        .await.map_err(|_| ApiError::Internal)?;

    let answer = rx.await.map_err(|_| ApiError::Internal)?;
    Ok(Json(ConnectResponse { sdp: answer.sdp, conn_id: conn_id.to_string(), ice_lite: true }))
}

// POST /webrtc/:conn_id/ice-candidate — trickle ICE
async fn ice_candidate(
    Path(conn_id): Path<String>,
    State(state): State<Arc<MediaState>>,
    Json(req): Json<IceCandidateRequest>,
) -> Result<StatusCode, ApiError> {
    let conn_id: u64 = conn_id.parse().map_err(|_| ApiError::BadRequest("invalid conn_id".into()))?;
    state.rpc_tx.send(RpcReq::RemoteIce { conn_id, candidate: req })
        .await.map_err(|_| ApiError::Internal)?;
    Ok(StatusCode::NO_CONTENT)
}

// POST /webrtc/:conn_id/restart-ice — ICE restart with new SDP
async fn restart_ice(
    Path(conn_id): Path<String>,
    State(state): State<Arc<MediaState>>,
    Json(req): Json<ConnectRequest>,
) -> Result<Json<ConnectResponse>, ApiError> {
    // Same flow as connect, but for existing connection
    let conn_id: u64 = conn_id.parse().map_err(|_| ApiError::BadRequest("invalid conn_id".into()))?;
    let claims = state.secure.decode_token(&req.token)
        .map_err(|_| ApiError::Forbidden("invalid token"))?;

    let (tx, rx) = tokio::sync::oneshot::channel();
    state.rpc_tx.send(RpcReq::RestartIce { conn_id, sdp: req.sdp, reply: tx })
        .await.map_err(|_| ApiError::Internal)?;

    let answer = rx.await.map_err(|_| ApiError::Internal)?;
    Ok(Json(ConnectResponse { sdp: answer.sdp, conn_id: conn_id.to_string(), ice_lite: true }))
}

fn webrtc_routes() -> Router<Arc<MediaState>> {
    Router::new()
        .route("/connect", post(connect))
        .route("/{conn_id}/ice-candidate", post(ice_candidate))
        .route("/{conn_id}/restart-ice", post(restart_ice))
}
```

**When to use which signaling approach:**

| Approach | Use When | Example |
|----------|----------|---------|
| **WebSocket signaling** | P2P calls between users, real-time presence needed alongside signaling | Chat app with video calling |
| **HTTP POST (WHIP/WHEP)** | Server-mediated streams, broadcasting, SFU architecture | Live streaming, media servers, conferencing |

## Static Asset Serving

### Embedding Frontend with rust-embed

Embed frontend assets (HTML, JS, CSS, images) directly into the Rust binary for single-file deployment:

```rust
use rust_embed::RustEmbed;
use std::path::Path;

#[derive(RustEmbed)]
#[folder = "frontend/dist"]
struct FrontendAssets;
```

Dependencies:
```toml
[dependencies]
rust-embed = "8.3"
mime_guess = "2.0"
```

### SPA Catch-All Routing

For single-page applications, route all non-API requests to the frontend:

```rust
// Axum
async fn spa_fallback() -> impl IntoResponse {
    match FrontendAssets::get("index.html") {
        Some(content) => (
            [(axum::http::header::CONTENT_TYPE, "text/html")],
            content.data.to_vec(),
        )
            .into_response(),
        None => StatusCode::NOT_FOUND.into_response(),
    }
}

// Serve static assets by path
async fn static_asset(Path(path): Path<String>) -> impl IntoResponse {
    match FrontendAssets::get(&path) {
        Some(content) => {
            let mime = mime_guess::from_path(&path)
                .first_or_octet_stream()
                .to_string();
            (
                [(axum::http::header::CONTENT_TYPE, mime)],
                content.data.to_vec(),
            )
                .into_response()
        }
        None => StatusCode::NOT_FOUND.into_response(),
    }
}

let app = Router::new()
    .nest("/api", api_routes())
    .route("/assets/*path", get(static_asset))
    .fallback(spa_fallback);
```

```rust
// Actix Web
async fn catch_all(req: HttpRequest) -> impl Responder {
    // API routes should 404 if not matched
    if req.path().starts_with("/api/") {
        return HttpResponse::NotFound().finish();
    }

    // Check if requesting a known static file
    let path = req.path().trim_start_matches('/');
    if let Some(content) = FrontendAssets::get(path) {
        let mime = mime_guess::from_path(path)
            .first_or_octet_stream()
            .to_string();
        return HttpResponse::Ok()
            .content_type(mime)
            .append_header(("Cache-Control", "public, max-age=604800"))
            .body(content.data.to_vec());
    }

    // Default: serve index.html for SPA client-side routing
    match FrontendAssets::get("index.html") {
        Some(content) => HttpResponse::Ok()
            .content_type("text/html")
            .body(content.data.to_vec()),
        None => HttpResponse::NotFound().finish(),
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .configure(api_routes)
            .default_service(web::route().to(catch_all))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

## Frontend Token Storage

Tokens should be stored in localStorage or httpOnly cookies:

```javascript
// Login API call
export const login = async (email, password) => {
    const authToken = btoa(`${email}:${password}`);  // Base64 encode

    const response = await fetch('/api/v1/auth/login', {
        method: 'GET',
        headers: {
            'Authorization': `Basic ${authToken}`,
            'Content-Type': 'application/json',
        },
    });

    return response.json();  // JWT token
};

// Store token
localStorage.setItem('token', token);

// Include token in subsequent requests
const response = await fetch('/api/v1/items', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(requestBody),
});
```

## Security Checklist

1. **Use Argon2 for passwords** — winner of Password Hashing Competition, recommended over bcrypt/scrypt
2. **HTTPS only** — JWT and Basic Auth are insecure over HTTP
3. **Validate all input** — use serde + custom validation
4. **Rate limit auth endpoints** — prevent brute-force attacks
5. **Never log passwords** — use redacting newtype
6. **Set JWT expiration** — don't disable `exp` validation in production
7. **Restrict CORS in production** — never use `allow_any_origin()` in prod
8. **Use httpOnly cookies** when possible — for XSS protection
9. **Store JWT secrets securely** — use long random strings, rotate periodically
10. **Never expose password hashes** — use TrimmedUser pattern for API responses

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: ownership, traits, error handling, serde essentials, async basics
- **[error-handling.md](error-handling.md)** — thiserror/anyhow for API error types, multi-layer error translation
- **[serde-serialization.md](serde-serialization.md)** — Custom serialization, enum representations, field attributes for API types
- **[async-concurrency.md](async-concurrency.md)** — Tower service pattern, graceful shutdown, backpressure, tokio runtime
- **[database.md](database.md)** — SQLx queries, connection pools, migrations for API backends
- **[testing.md](testing.md)** — API integration testing with reqwest, async test patterns
- **[deployment.md](deployment.md)** — Docker builds, CI/CD, observability for web services
