# Deployment and Operations

Build optimization, cross-compilation, containerization, CI/CD pipelines, deployment strategies, structured logging, metrics, health checks, infrastructure, and reverse proxies for production Rust applications.

## Rules for Deployment (LLM)

1. **ALWAYS use `Swatinem/rust-cache@v2` in CI** — handles target, registry, and git caching automatically; manual `actions/cache` misses edge cases and requires path maintenance
2. **ALWAYS add concurrency cancellation to CI workflows** — `concurrency: { group: ..., cancel-in-progress: true }` prevents wasted minutes on superseded commits
3. **ALWAYS use multi-stage Docker builds** — build in a Rust image, copy binary to distroless/scratch; never ship compiler toolchain in production images
4. **ALWAYS set `lto = "fat"` and `codegen-units = 1` in release profile** — up to 20% smaller binaries and better runtime performance at the cost of compile time
5. **ALWAYS run `cargo clippy` and `cargo fmt --check` as separate CI jobs before tests** — fast failures prevent wasting CI time on code that won't pass review
6. **NEVER use `--all-features` in CI without understanding what it enables** — some features are mutually exclusive or platform-specific; use `cargo-hack` for systematic feature testing
7. **ALWAYS test the minimum supported Rust version (MSRV) in CI** — set `rust-version` in `Cargo.toml` and add a dedicated CI job that builds with that version
8. **PREFER `strip = "symbols"` in release profile** — reduces binary size 50-80% with no runtime cost; debug info is rarely useful in production

### Common Mistakes (BAD/GOOD)

**Wrong Docker layer ordering:**
```dockerfile
# BAD: any source change invalidates the dependency cache layer
FROM rust:1.82 AS builder
COPY . .
RUN cargo build --release

# GOOD: cache dependencies separately — only rebuilds app code on source changes
FROM rust:1.82 AS builder
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs && cargo build --release && rm -rf src
COPY src/ src/
RUN touch src/main.rs && cargo build --release
```

**Missing health check endpoint:**
```rust
// BAD: no way for orchestrator to verify readiness — blind restarts on any issue
// (no /health endpoint)

// GOOD: health check verifies actual dependencies, not just "process is alive"
async fn health_check(State(pool): State<PgPool>) -> StatusCode {
    match sqlx::query("SELECT 1").execute(&pool).await {
        Ok(_) => StatusCode::OK,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE,
    }
}
```

**Not stripping binaries:**
```toml
# BAD: shipping 150MB debug binary to production
[profile.release]
# (no strip setting — defaults to false)

# GOOD: strip + LTO = 10-30MB binary
[profile.release]
strip = "symbols"
lto = "fat"
codegen-units = 1
```

**CI without caching:**
```yaml
# BAD: 15+ minute builds on every push — no caching
- uses: actions/checkout@v4
- run: cargo build --release

# GOOD: cached builds complete in 2-3 minutes for incremental changes
- uses: actions/checkout@v4
- uses: Swatinem/rust-cache@v2
- run: cargo build --release
```

### Section Index

| Section | Topics |
|---------|--------|
| [Build Optimization](#build-optimization) | Release profiles, LTO, codegen-units, strip, dev profiles |
| [Cross-Compilation](#cross-compilation) | Target triples, cross, platform-specific code |
| [Containerization](#containerization-docker) | Multi-stage builds, distroless, cargo-chef, layer caching |
| [Distroless Docker](#distroless-docker-with-dynamic-libraries) | Dynamic linking, ldd, scratch images |
| [CI/CD Pipelines](#cicd-pipelines) | GitHub Actions, GitLab CI, rust-cache, cargo-hack, MSRV |
| [Deployment Strategies](#deployment-strategies) | Systemd, rolling updates, blue-green, canary |
| [Structured Logging](#structured-logging) | tracing subscribers, EnvFilter, JSON, layers, spans |
| [Metrics Collection](#metrics-collection) | prometheus, opentelemetry, custom metrics |
| [Health Checks](#health-checks) | Liveness, readiness, dependency verification |
| [AWS Infrastructure](#aws-infrastructure-with-terraform) | EC2, ECS, Terraform, IAM |
| [NGINX Reverse Proxy](#nginx-reverse-proxy-and-https) | TLS termination, proxy_pass, rate limiting |

## Build Optimization

### Release Profile Configuration

Configure `Cargo.toml` for optimized production builds:

```toml
[profile.release]
# Link-Time Optimization - enables cross-crate optimization
# Options: false, true/"fat", "thin"
# "fat" = more aggressive, slower compile, best runtime performance
# "thin" = faster compile, good optimization
lto = "fat"

# Codegen units - fewer units = better optimization, slower compile
# Default is 16 for parallel compilation
# Set to 1 for maximum optimization
codegen-units = 1

# Panic behavior - "abort" vs "unwind"
# "abort" = smaller binary, no stack unwinding overhead
# "unwind" = allows catch_unwind, larger binary
panic = "abort"

# Debug symbols - false removes them entirely
debug = false

# Strip symbols from binary
# Options: "none", "debuginfo", "symbols"
strip = "symbols"

# Optimization level
# 0 = no optimization (debug)
# 1 = basic optimization
# 2 = some optimization
# 3 = full optimization (default for release)
# "s" = optimize for size
# "z" = optimize for size aggressively
opt-level = 3
```

### Size-Optimized Profile

For embedded or size-constrained deployments:

```toml
[profile.release-small]
inherits = "release"
opt-level = "z"
lto = true
codegen-units = 1
panic = "abort"
strip = "symbols"
```

Build with: `cargo build --profile release-small`

### Custom Profiles

```toml
# Fast compilation for CI testing
[profile.ci]
inherits = "dev"
opt-level = 1
debug = false

# Profiling build - optimized but with debug symbols
[profile.profiling]
inherits = "release"
debug = true
strip = "none"
```

### Feature Flags for Production

```toml
[features]
default = ["logging"]
logging = ["tracing", "tracing-subscriber"]
metrics = ["prometheus"]
production = ["logging", "metrics"]

# Disable default features for minimal builds
[dependencies]
serde = { version = "1.0", default-features = false, features = ["derive"] }
tokio = { version = "1", default-features = false, features = ["rt", "macros"] }
```

### Binary Size Reduction

```bash
# Check binary size
ls -lh target/release/myapp

# Analyze what takes space
cargo install cargo-bloat
cargo bloat --release
cargo bloat --release --crates  # By crate

# Further reduction
strip target/release/myapp
```

| Technique | Size Impact |
|-----------|------------|
| `strip = "symbols"` | -50-80% |
| `opt-level = "z"` | -10-30% |
| `panic = "abort"` | -5-10% |
| `lto = "fat"` | -10-20% |
| `codegen-units = 1` | -5-10% |
| Disable default features | Varies |

## Cross-Compilation

### Target Triples

Target triples follow the pattern: `<arch>-<vendor>-<os>-<env>`

Common targets:
| Target | Description |
|--------|-------------|
| `x86_64-unknown-linux-gnu` | 64-bit Linux with glibc |
| `x86_64-unknown-linux-musl` | 64-bit Linux with musl (static) |
| `aarch64-unknown-linux-gnu` | 64-bit ARM Linux |
| `aarch64-apple-darwin` | Apple Silicon macOS |
| `x86_64-pc-windows-msvc` | 64-bit Windows |
| `thumbv7m-none-eabi` | ARM Cortex-M (bare metal) |
| `wasm32-unknown-unknown` | WebAssembly |

### Setting Up Cross-Compilation

```bash
# List available targets
rustup target list

# Add a target
rustup target add x86_64-unknown-linux-musl
rustup target add aarch64-unknown-linux-gnu

# Build for specific target
cargo build --release --target x86_64-unknown-linux-musl
```

### Linker Configuration

Create `.cargo/config.toml` in project root:

```toml
[build]
# Default target (optional)
target = "x86_64-unknown-linux-gnu"

# ARM64 Linux configuration
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"

# ARM32 Linux configuration
[target.armv7-unknown-linux-gnueabihf]
linker = "arm-linux-gnueabihf-gcc"

# Static musl build (no external linker needed for pure Rust)
[target.x86_64-unknown-linux-musl]
rustflags = ["-C", "target-feature=+crt-static"]

# Bare metal ARM Cortex-M
[target.thumbv7m-none-eabi]
rustflags = [
    "-C", "link-arg=--sysroot=/path/to/arm-sysroot",
    "-C", "link-arg=-L/path/to/arm-sysroot/lib",
]
linker = "/path/to/arm-none-eabi-gcc"
```

### Installing Cross-Compilation Toolchains

```bash
# Ubuntu/Debian - ARM64 Linux
sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

# Ubuntu/Debian - ARM32 Linux
sudo apt install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf

# Ubuntu/Debian - musl toolchain
sudo apt install musl-tools

# macOS with Homebrew
brew install filosottile/musl-cross/musl-cross
```

### Using cross for Simplified Cross-Compilation

```bash
# Install cross (uses Docker for toolchains)
cargo install cross

# Cross-compile (automatically handles toolchains)
cross build --release --target aarch64-unknown-linux-gnu
cross build --release --target x86_64-unknown-linux-musl
```

### Handling C Dependencies

For crates with C dependencies (`*-sys` crates):

```toml
# .cargo/config.toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"

[env]
# For OpenSSL
OPENSSL_DIR = "/path/to/cross-compiled-openssl"
OPENSSL_STATIC = "1"

# Alternative: use pure Rust TLS
# Replace openssl with rustls in dependencies
```

## Containerization (Docker)

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM rust:1.82 AS builder

WORKDIR /usr/src/app

# Cache dependencies - copy manifests first
COPY Cargo.toml Cargo.lock ./

# Create dummy source to build dependencies
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release --locked
RUN rm -rf src

# Build actual application
COPY src ./src
RUN touch src/main.rs && cargo build --release --locked

# Runtime stage - minimal image
FROM debian:bookworm-slim AS runtime

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy binary from builder
COPY --from=builder /usr/src/app/target/release/myapp /app/myapp

# Non-root user for security
RUN useradd -r -s /bin/false appuser
RUN chown appuser:appuser /app/myapp
USER appuser

EXPOSE 8080

CMD ["./myapp"]
```

### Alpine/musl Static Binary

```dockerfile
# Build stage with musl target
FROM rust:1.82 AS builder

WORKDIR /usr/src/app

# Add musl target
RUN rustup target add x86_64-unknown-linux-musl

# Cache dependencies
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release --locked --target x86_64-unknown-linux-musl
RUN rm -rf src

# Build application
COPY src ./src
RUN touch src/main.rs && cargo build --release --locked --target x86_64-unknown-linux-musl

# Minimal runtime - Alpine or scratch
FROM alpine:3.19 AS runtime

WORKDIR /app

COPY --from=builder /usr/src/app/target/x86_64-unknown-linux-musl/release/myapp /app/myapp

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
RUN chown appuser:appgroup /app/myapp
USER appuser

EXPOSE 8080

CMD ["./myapp"]
```

### Scratch Image (Smallest Possible)

```dockerfile
FROM rust:1.82 AS builder
WORKDIR /usr/src/app
RUN rustup target add x86_64-unknown-linux-musl
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release --locked --target x86_64-unknown-linux-musl
RUN rm -rf src
COPY src ./src
RUN touch src/main.rs && cargo build --release --locked --target x86_64-unknown-linux-musl

# Scratch - no OS, just the binary
FROM scratch
COPY --from=builder /usr/src/app/target/x86_64-unknown-linux-musl/release/myapp /myapp
# Note: No shell, no debugging tools
ENTRYPOINT ["/myapp"]
```

### Workspace Dockerfile

```dockerfile
FROM rust:1.82 AS builder

WORKDIR /usr/src/app

# Copy workspace manifests
COPY Cargo.toml Cargo.lock ./
COPY server/Cargo.toml server/
COPY shared/Cargo.toml shared/
COPY client/Cargo.toml client/

# Create dummy sources for dependency caching
RUN mkdir -p server/src shared/src client/src \
    && echo "fn main() {}" > server/src/main.rs \
    && echo "" > shared/src/lib.rs \
    && echo "fn main() {}" > client/src/main.rs

RUN cargo build --release --locked -p server
RUN rm -rf server/src shared/src client/src

# Copy actual sources
COPY server/src server/src
COPY shared/src shared/src

RUN touch server/src/main.rs && cargo build --release --locked -p server

FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /usr/src/app/target/release/server /app/server
RUN useradd -r -s /bin/false appuser
USER appuser
EXPOSE 8080
CMD ["./server"]
```

### Build Arguments and Features

```dockerfile
FROM rust:1.82 AS builder

ARG RUST_FEATURES="default"
ARG PROFILE="release"

WORKDIR /usr/src/app

COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --profile ${PROFILE} --locked --features "${RUST_FEATURES}"
RUN rm -rf src

COPY src ./src
RUN touch src/main.rs && cargo build --profile ${PROFILE} --locked --features "${RUST_FEATURES}"

FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /usr/src/app/target/release/myapp /app/myapp
CMD ["./myapp"]
```

Build with features:
```bash
docker build --build-arg RUST_FEATURES="production,metrics" -t myapp .
```

### Docker Compose for Development

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://user:password@db:5432/mydb
      RUST_LOG: info
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
```

## Distroless Docker with Dynamic Libraries

### Understanding Dynamic Library Requirements

Distroless images contain only your application and its runtime dependencies. Use `ldd` to identify required libraries:

```bash
# Identify dynamic libraries needed by your binary
ldd target/release/myapp
```

### Architecture-Specific Distroless Builds

**For x86_64 (Intel/AMD):**

```dockerfile
FROM rust:latest AS build
WORKDIR /app
COPY . .
RUN cargo build --release
# Strip debug symbols for smaller binary
RUN strip /app/target/release/myapp

FROM gcr.io/distroless/cc-debian12

# Copy dynamic linker and required libraries
COPY --chown=1001:1001 --from=build \
    /lib64/ld-linux-x86-64.so.2 \
    /lib64/ld-linux-x86-64.so.2
COPY --chown=1001:1001 --from=build \
    /lib/x86_64-linux-gnu/libssl.so.3 \
    /lib/x86_64-linux-gnu/libssl.so.3
COPY --chown=1001:1001 --from=build \
    /lib/x86_64-linux-gnu/libcrypto.so.3 \
    /lib/x86_64-linux-gnu/libcrypto.so.3
COPY --chown=1001:1001 --from=build \
    /lib/x86_64-linux-gnu/libgcc_s.so.1 \
    /lib/x86_64-linux-gnu/libgcc_s.so.1
COPY --chown=1001:1001 --from=build \
    /lib/x86_64-linux-gnu/libm.so.6 \
    /lib/x86_64-linux-gnu/libm.so.6
COPY --chown=1001:1001 --from=build \
    /lib/x86_64-linux-gnu/libc.so.6 \
    /lib/x86_64-linux-gnu/libc.so.6

# Copy application binary
COPY --from=build /app/target/release/myapp /usr/local/bin/myapp

EXPOSE 8080
CMD ["myapp"]
```

**For AArch64 (ARM64/Apple Silicon):**

```dockerfile
FROM rust:latest AS build
WORKDIR /app
COPY . .
RUN cargo build --release
RUN strip /app/target/release/myapp

FROM gcr.io/distroless/cc-debian12

# ARM64-specific library paths
COPY --chown=1001:1001 --from=build \
    /lib/ld-linux-aarch64.so.1 \
    /lib/ld-linux-aarch64.so.1
COPY --chown=1001:1001 --from=build \
    /lib/aarch64-linux-gnu/libssl.so.3 \
    /lib/aarch64-linux-gnu/libssl.so.3
COPY --chown=1001:1001 --from=build \
    /lib/aarch64-linux-gnu/libcrypto.so.3 \
    /lib/aarch64-linux-gnu/libcrypto.so.3
COPY --chown=1001:1001 --from=build \
    /lib/aarch64-linux-gnu/libgcc_s.so.1 \
    /lib/aarch64-linux-gnu/libgcc_s.so.1
COPY --chown=1001:1001 --from=build \
    /lib/aarch64-linux-gnu/libm.so.6 \
    /lib/aarch64-linux-gnu/libm.so.6
COPY --chown=1001:1001 --from=build \
    /lib/aarch64-linux-gnu/libc.so.6 \
    /lib/aarch64-linux-gnu/libc.so.6

COPY --from=build /app/target/release/myapp /usr/local/bin/myapp

EXPOSE 8080
CMD ["myapp"]
```

### Library Reference

| Library | Purpose |
|---------|---------|
| `ld-linux-*.so` | Dynamic linker/loader |
| `libssl.so.3` | OpenSSL TLS/SSL |
| `libcrypto.so.3` | OpenSSL cryptographic routines |
| `libgcc_s.so.1` | GCC runtime (exception handling) |
| `libm.so.6` | Math functions (sin, cos, sqrt) |
| `libc.so.6` | GNU C Library (system calls, I/O) |

### Multi-Service Image with Command Override

Build one image that runs different services based on the command:

```dockerfile
FROM rust:latest AS build
WORKDIR /app
COPY . .
RUN cargo build --release -p auth_server
RUN cargo build --release -p app_server
RUN cargo build --release -p ingress
RUN strip /app/target/release/auth_server
RUN strip /app/target/release/app_server
RUN strip /app/target/release/ingress

FROM gcr.io/distroless/cc-debian12
# ... copy libraries ...
COPY --from=build /app/target/release/auth_server /usr/local/bin/
COPY --from=build /app/target/release/app_server /usr/local/bin/
COPY --from=build /app/target/release/ingress /usr/local/bin/

EXPOSE 8001 8080 8081
CMD ["ingress"]  # Default, can be overridden
```

```yaml
# docker-compose.yml - override command per service
services:
  auth:
    image: compute-unit
    command: ["auth_server"]
    ports: ["8081:8081"]

  app:
    image: compute-unit
    command: ["app_server"]
    ports: ["8080:8080"]

  ingress:
    image: compute-unit
    command: ["ingress"]
    ports: ["8001:8001"]
```

## CI/CD Pipelines

### GitHub Actions

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# Cancel superseded runs — saves CI minutes
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: true

env:
  CARGO_TERM_COLOR: always
  RUST_BACKTRACE: 1

jobs:
  check:
    name: Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - uses: Swatinem/rust-cache@v2  # Handles target + registry + git automatically

      - name: Check formatting
        run: cargo fmt --all -- --check

      - name: Run clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

      - name: Check compilation
        run: cargo check --all-features

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: check
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable

      - uses: Swatinem/rust-cache@v2

      - name: Run tests
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
        run: cargo test --all-features --verbose

  build:
    name: Build Release
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable

      - name: Cache cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Build release
        run: cargo build --release --locked

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: binary
          path: target/release/myapp

  # Matrix build for multiple Rust versions
  test-matrix:
    name: Test on Rust ${{ matrix.rust }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        rust: [stable, beta, "1.70", "1.75"]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust }}
      - run: cargo test

  # Cross-platform builds
  build-cross:
    name: Build ${{ matrix.target }}
    runs-on: ubuntu-latest
    needs: test
    strategy:
      matrix:
        target:
          - x86_64-unknown-linux-gnu
          - x86_64-unknown-linux-musl
          - aarch64-unknown-linux-gnu
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install cross
        run: cargo install cross

      - name: Build
        run: cross build --release --target ${{ matrix.target }}

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: binary-${{ matrix.target }}
          path: target/${{ matrix.target }}/release/myapp

  docker:
    name: Build and Push Docker
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### MSRV Testing Job

Verify your crate compiles with the minimum supported Rust version:

```yaml
  msrv:
    name: MSRV Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: "1.75"  # Must match rust-version in Cargo.toml
      - uses: Swatinem/rust-cache@v2
      - run: cargo check --workspace --all-features
```

Set in `Cargo.toml`:
```toml
[package]
rust-version = "1.75"
```

### Feature Flag Testing with cargo-hack

Test feature combinations systematically — catches features that break when combined:

```yaml
  features:
    name: Feature Combinations
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - name: Install cargo-hack
        run: cargo install cargo-hack
      - name: Check feature powerset
        run: cargo hack check --feature-powerset --depth 2 --workspace
```

### Semver Compliance Check (Libraries)

For library crates, automatically detect accidental breaking changes:

```yaml
  semver:
    name: Semver Check
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: obi1kenobi/cargo-semver-checks-action@v2
```

### Workspace-Level Lint Configuration

Set consistent lints across all workspace crates:

```toml
# Cargo.toml (workspace root)
[workspace.lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"
rust_2018_idioms = { level = "warn", priority = -1 }

[workspace.lints.clippy]
dbg_macro = "warn"
print_stdout = "warn"
wildcard_imports = "warn"

# In each crate's Cargo.toml:
[lints]
workspace = true
```

### GitLab CI

```yaml
image: rust:1.82

stages:
  - check
  - test
  - build
  - deploy

variables:
  CARGO_HOME: ${CI_PROJECT_DIR}/.cargo
  RUST_BACKTRACE: "1"

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .cargo/registry/
    - .cargo/git/
    - target/

fmt:
  stage: check
  script:
    - rustup component add rustfmt
    - cargo fmt --all -- --check

clippy:
  stage: check
  script:
    - rustup component add clippy
    - cargo clippy --all-targets --all-features -- -D warnings

check:
  stage: check
  script:
    - cargo check --all-features

test:
  stage: test
  services:
    - postgres:16
  variables:
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    POSTGRES_DB: testdb
    DATABASE_URL: postgres://test:test@postgres:5432/testdb
  script:
    - cargo test --all-features --verbose

build:
  stage: build
  script:
    - cargo build --release --locked
  artifacts:
    paths:
      - target/release/myapp
    expire_in: 1 week

build:musl:
  stage: build
  image: rust:1.82-alpine
  before_script:
    - apk add --no-cache musl-dev
    - rustup target add x86_64-unknown-linux-musl
  script:
    - cargo build --release --locked --target x86_64-unknown-linux-musl
  artifacts:
    paths:
      - target/x86_64-unknown-linux-musl/release/myapp
    expire_in: 1 week

docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

deploy:staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - echo "Deploying to staging..."
    # Add deployment commands
  only:
    - main

deploy:production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - echo "Deploying to production..."
  when: manual
  only:
    - main
```

## Deployment Strategies

### Blue-Green Deployment

Maintain two identical environments, deploy to inactive, then switch:

```yaml
# Kubernetes Blue-Green with Service selector switch
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:v1.0.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:v1.1.0
---
# Switch by changing selector
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Change to "green" to switch
  ports:
  - port: 80
    targetPort: 8080
```

### Canary Deployment

Gradually route traffic to new version:

```yaml
# Istio VirtualService for canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 90
    - destination:
        host: myapp
        subset: canary
      weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary
```

### Rolling Update (Kubernetes Default)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods over desired count
      maxUnavailable: 0  # Zero downtime
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:latest
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
```

## Structured Logging

### Setting Up tracing

```rust
use tracing::{info, warn, error, debug, instrument, Level};
use tracing_subscriber::{fmt, prelude::*, EnvFilter};

fn init_logging() {
    // JSON format for production
    let subscriber = tracing_subscriber::registry()
        .with(EnvFilter::from_default_env()
            .add_directive(Level::INFO.into()))
        .with(fmt::layer()
            .json()  // JSON output
            .with_current_span(true)
            .with_span_list(true)
            .with_file(true)
            .with_line_number(true)
            .with_thread_ids(true));

    tracing::subscriber::set_global_default(subscriber)
        .expect("Failed to set subscriber");
}

// Alternative: human-readable for development
fn init_logging_dev() {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .with_target(true)
        .with_thread_ids(true)
        .pretty()
        .init();
}
```

### Using Spans and Events

```rust
use tracing::{info, error, instrument, Span};

// Automatic span creation with #[instrument]
#[instrument(skip(password), fields(user_id))]
async fn login(username: &str, password: &str) -> Result<User, AuthError> {
    info!("Attempting login");

    let user = find_user(username).await?;

    // Record field value discovered during execution
    Span::current().record("user_id", user.id);

    if !verify_password(&user, password) {
        error!(username = %username, "Invalid password");
        return Err(AuthError::InvalidCredentials);
    }

    info!("Login successful");
    Ok(user)
}

// Manual span creation for fine-grained control
async fn process_order(order_id: u64) -> Result<(), OrderError> {
    let span = tracing::info_span!(
        "process_order",
        order_id = %order_id,
        status = tracing::field::Empty,  // Fill in later
    );
    let _guard = span.enter();

    info!("Starting order processing");

    // Update span with discovered information
    span.record("status", "validating");
    validate_order(order_id).await?;

    span.record("status", "charging");
    charge_payment(order_id).await?;

    span.record("status", "completed");
    info!("Order processing complete");

    Ok(())
}
```

### Structured Fields

```rust
use tracing::{info, error, warn, Level};

fn log_request(req: &Request) {
    // Structured fields - machine-parseable
    info!(
        method = %req.method(),
        path = %req.uri().path(),
        user_agent = ?req.headers().get("user-agent"),
        request_id = %req.request_id(),
        "Incoming request"
    );
}

fn log_response(req: &Request, resp: &Response, duration: Duration) {
    let level = if resp.status().is_success() {
        Level::INFO
    } else if resp.status().is_client_error() {
        Level::WARN
    } else {
        Level::ERROR
    };

    tracing::event!(
        level,
        method = %req.method(),
        path = %req.uri().path(),
        status = %resp.status().as_u16(),
        duration_ms = %duration.as_millis(),
        request_id = %req.request_id(),
        "Request completed"
    );
}

fn log_error(error: &AppError, context: &str) {
    error!(
        error_type = %error.error_type(),
        error_code = %error.code(),
        message = %error.message(),
        context = %context,
        backtrace = ?error.backtrace(),
        "Application error occurred"
    );
}
```

### Layer Composition

```rust
use tracing_subscriber::Layer;

// File logging layer
let file = std::fs::File::create("app.log")?;
let file_layer = tracing_subscriber::fmt::layer()
    .json()
    .with_writer(file)
    .with_filter(EnvFilter::new("info"));

// Console layer
let console_layer = tracing_subscriber::fmt::layer()
    .pretty()
    .with_filter(EnvFilter::new("debug"));

tracing_subscriber::registry()
    .with(file_layer)
    .with(console_layer)
    .init();
```

### Request Tracing Middleware

```rust
use axum::{
    middleware::Next,
    http::{Request, Response},
    body::Body,
};
use tracing::{info_span, Instrument};
use uuid::Uuid;

pub async fn tracing_middleware(
    request: Request<Body>,
    next: Next,
) -> Response<Body> {
    let request_id = Uuid::new_v4().to_string();
    let method = request.method().clone();
    let uri = request.uri().clone();

    let span = info_span!(
        "http_request",
        request_id = %request_id,
        method = %method,
        uri = %uri,
        status = tracing::field::Empty,
        latency_ms = tracing::field::Empty,
    );

    async move {
        let start = std::time::Instant::now();

        let response = next.run(request).await;

        let latency = start.elapsed();
        let status = response.status();

        tracing::Span::current()
            .record("status", status.as_u16())
            .record("latency_ms", latency.as_millis() as u64);

        if status.is_success() {
            tracing::info!("Request completed successfully");
        } else {
            tracing::warn!("Request completed with error");
        }

        response
    }
    .instrument(span)
    .await
}
```

### Distributed Tracing

```rust
use opentelemetry::global;
use opentelemetry_sdk::trace::TracerProvider;
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::prelude::*;

fn init_tracing() {
    // Set up OpenTelemetry exporter (e.g., Jaeger)
    let tracer = opentelemetry_jaeger::new_agent_pipeline()
        .with_service_name("my-rust-service")
        .install_simple()
        .expect("Failed to install Jaeger tracer");

    // Combine with tracing subscriber
    let telemetry = OpenTelemetryLayer::new(tracer);

    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer().json())
        .with(telemetry)
        .with(EnvFilter::from_default_env())
        .init();
}

// Propagate trace context in HTTP calls
async fn call_downstream_service(
    client: &reqwest::Client,
    url: &str,
) -> Result<reqwest::Response, reqwest::Error> {
    use tracing_opentelemetry::OpenTelemetrySpanExt;

    let span = tracing::Span::current();
    let context = span.context();

    // Inject trace context into headers
    let mut headers = HeaderMap::new();
    global::get_text_map_propagator(|propagator| {
        propagator.inject_context(&context, &mut HeaderInjector(&mut headers));
    });

    client.get(url)
        .headers(headers)
        .send()
        .await
}
```

## Metrics Collection

### Prometheus Setup

```rust
use prometheus::{
    Counter, CounterVec, Gauge, Histogram, HistogramVec,
    Opts, Registry, TextEncoder, Encoder,
    register_counter_vec, register_histogram_vec, register_gauge,
};
use std::sync::LazyLock;

// Request counter with labels
static HTTP_REQUESTS_TOTAL: LazyLock<CounterVec> = LazyLock::new(|| {
    register_counter_vec!(
        "http_requests_total",
        "Total number of HTTP requests",
        &["method", "endpoint", "status"]
    ).unwrap()
});

// Request duration histogram
static HTTP_REQUEST_DURATION: LazyLock<HistogramVec> = LazyLock::new(|| {
    register_histogram_vec!(
        "http_request_duration_seconds",
        "HTTP request duration in seconds",
        &["method", "endpoint"],
        vec![0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
    ).unwrap()
});

// Active connections gauge
static ACTIVE_CONNECTIONS: LazyLock<Gauge> = LazyLock::new(|| {
    register_gauge!(
        "active_connections",
        "Number of active connections"
    ).unwrap()
});

// Business metrics
static ORDERS_PROCESSED: LazyLock<CounterVec> = LazyLock::new(|| {
    register_counter_vec!(
        "orders_processed_total",
        "Total orders processed",
        &["status"]  // success, failed, cancelled
    ).unwrap()
});

static ORDER_VALUE: LazyLock<Histogram> = LazyLock::new(|| {
    Histogram::with_opts(
        Opts::new("order_value_dollars", "Order value in dollars")
    ).unwrap()
});

// Initialize all metrics
pub fn init_metrics() {
    // Register custom metrics if using custom registry
    let registry = Registry::new();
    registry.register(Box::new(HTTP_REQUESTS_TOTAL.clone())).unwrap();
    registry.register(Box::new(HTTP_REQUEST_DURATION.clone())).unwrap();
    registry.register(Box::new(ACTIVE_CONNECTIONS.clone())).unwrap();
}
```

### Recording Metrics

```rust
use std::time::Instant;

// Middleware for HTTP metrics
pub async fn metrics_middleware(
    request: Request<Body>,
    next: Next,
) -> Response<Body> {
    let method = request.method().to_string();
    let endpoint = request.uri().path().to_string();

    ACTIVE_CONNECTIONS.inc();
    let start = Instant::now();

    let response = next.run(request).await;

    let duration = start.elapsed().as_secs_f64();
    let status = response.status().as_u16().to_string();

    HTTP_REQUESTS_TOTAL
        .with_label_values(&[&method, &endpoint, &status])
        .inc();

    HTTP_REQUEST_DURATION
        .with_label_values(&[&method, &endpoint])
        .observe(duration);

    ACTIVE_CONNECTIONS.dec();

    response
}

// Business logic metrics
fn process_order(order: &Order) -> Result<(), OrderError> {
    let result = do_process_order(order);

    match &result {
        Ok(_) => {
            ORDERS_PROCESSED.with_label_values(&["success"]).inc();
            ORDER_VALUE.observe(order.total_value());
        }
        Err(_) => {
            ORDERS_PROCESSED.with_label_values(&["failed"]).inc();
        }
    }

    result
}
```

### Metrics Endpoint

```rust
use axum::{routing::get, Router, response::IntoResponse};
use prometheus::{TextEncoder, Encoder};

async fn metrics_handler() -> impl IntoResponse {
    let encoder = TextEncoder::new();
    let metric_families = prometheus::gather();
    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer).unwrap();

    (
        [(axum::http::header::CONTENT_TYPE, encoder.format_type())],
        buffer,
    )
}

fn app() -> Router {
    Router::new()
        .route("/metrics", get(metrics_handler))
        .route("/health", get(health_handler))
        // ... other routes
}
```

### Custom Metrics with metrics Crate

```rust
use tracing::instrument;
use metrics::{counter, gauge, histogram};

// Using the `metrics` crate (alternative to prometheus)
#[instrument(skip(order))]
async fn process_order(order: Order) -> Result<OrderResult, OrderError> {
    let start = std::time::Instant::now();

    gauge!("orders_in_flight").increment(1.0);

    let result = do_process(&order).await;

    gauge!("orders_in_flight").decrement(1.0);
    histogram!("order_processing_duration_seconds").record(start.elapsed().as_secs_f64());

    match &result {
        Ok(_) => counter!("orders_total", "status" => "success").increment(1),
        Err(e) => counter!("orders_total", "status" => "error", "error_type" => e.error_type()).increment(1),
    }

    result
}
```

### Alerting Rules (Prometheus/Alertmanager)

```yaml
# prometheus-rules.yaml
groups:
  - name: rust-app-alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP 5xx error rate"
          description: "Error rate is {{ $value | humanizePercentage }} over last 5 minutes"

      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High P95 latency"
          description: "P95 latency is {{ $value }}s"

      # Service down
      - alert: ServiceDown
        expr: up{job="rust-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          process_resident_memory_bytes{job="rust-app"}
          / 1024 / 1024 > 512
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value }}MB"
```

## Health Checks

### Kubernetes-Compatible Health Endpoints

```rust
use axum::{routing::get, Router, Json, http::StatusCode};
use serde::Serialize;
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Debug, Clone)]
pub struct HealthState {
    pub ready: Arc<RwLock<bool>>,
    pub db_pool: PgPool,
    pub redis: RedisPool,
}

#[derive(Serialize)]
struct HealthResponse {
    status: &'static str,
    checks: Option<HealthChecks>,
}

#[derive(Serialize)]
struct HealthChecks {
    database: CheckResult,
    redis: CheckResult,
}

#[derive(Serialize)]
struct CheckResult {
    status: &'static str,
    #[serde(skip_serializing_if = "Option::is_none")]
    error: Option<String>,
}

// Liveness - is the process running and not deadlocked?
async fn liveness() -> Json<HealthResponse> {
    Json(HealthResponse {
        status: "alive",
        checks: None,
    })
}

// Readiness - can we serve traffic?
async fn readiness(
    state: axum::extract::State<Arc<HealthState>>,
) -> Result<Json<HealthResponse>, (StatusCode, Json<HealthResponse>)> {
    // Check if marked ready
    if !*state.ready.read().await {
        return Err((
            StatusCode::SERVICE_UNAVAILABLE,
            Json(HealthResponse {
                status: "not ready",
                checks: None,
            }),
        ));
    }

    // Check database
    let db_check = match sqlx::query("SELECT 1")
        .execute(&state.db_pool)
        .await
    {
        Ok(_) => CheckResult { status: "ok", error: None },
        Err(e) => CheckResult { status: "error", error: Some(e.to_string()) },
    };

    // Check Redis
    let redis_check = match state.redis.get_connection() {
        Ok(mut conn) => {
            match redis::cmd("PING").query::<String>(&mut conn) {
                Ok(_) => CheckResult { status: "ok", error: None },
                Err(e) => CheckResult { status: "error", error: Some(e.to_string()) },
            }
        }
        Err(e) => CheckResult { status: "error", error: Some(e.to_string()) },
    };

    let all_ok = db_check.status == "ok" && redis_check.status == "ok";

    let response = HealthResponse {
        status: if all_ok { "ready" } else { "degraded" },
        checks: Some(HealthChecks {
            database: db_check,
            redis: redis_check,
        }),
    };

    if all_ok {
        Ok(Json(response))
    } else {
        Err((StatusCode::SERVICE_UNAVAILABLE, Json(response)))
    }
}

fn health_routes(state: Arc<HealthState>) -> Router {
    Router::new()
        .route("/health/live", get(liveness))
        .route("/health/ready", get(readiness))
        .with_state(state)
}
```

## AWS Infrastructure with Terraform

### Basic EC2 Instance

```hcl
# main.tf
terraform {
  required_version = ">= 1.9.4"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.62.0"
    }
  }
}

provider "aws" {
  region = "eu-west-2"
}

variable "db_password" {
  description = "Database password"
  sensitive   = true
}

variable "db_username" {
  description = "Database username"
  default     = "appuser"
}

resource "aws_instance" "production_server" {
  ami             = "ami-05ea2888c91c97ca7"  # Amazon Linux 2023
  instance_type   = "t2.medium"
  key_name        = "mykey"
  count           = 1
  user_data       = file("server_setup.sh")
  security_groups = [aws_security_group.webserver.name]

  tags = {
    Name = "production-server-${count.index}"
  }
}

output "ec2_public_ips" {
  value = aws_instance.production_server[*].public_ip
}
```

### RDS PostgreSQL Database

```hcl
# database.tf
resource "aws_db_parameter_group" "postgres_params" {
  name        = "app-postgres-params"
  family      = "postgres16"
  description = "Custom Postgres parameters"

  parameter {
    name  = "rds.force_ssl"
    value = "0"  # Disable forced SSL for development
  }
}

resource "aws_db_instance" "main_db" {
  identifier            = "app-database"
  instance_class        = "db.t3.micro"
  allocated_storage     = 20
  engine                = "postgres"
  engine_version        = "16"
  username              = var.db_username
  password              = var.db_password
  db_name               = "app_db"
  publicly_accessible   = true
  skip_final_snapshot   = true
  parameter_group_name  = aws_db_parameter_group.postgres_params.name

  tags = {
    Name = "production-database"
  }
}

output "db_endpoint" {
  value = aws_db_instance.main_db.endpoint
}
```

### Application Load Balancer with HTTPS

```hcl
# load_balancer.tf
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

data "aws_acm_certificate" "issued" {
  domain   = "example.com"
  statuses = ["ISSUED"]
}

resource "aws_lb_target_group" "app" {
  name        = "app-target-group"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = data.aws_vpc.default.id
  target_type = "instance"

  health_check {
    path                = "/health"
    protocol            = "HTTP"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}

resource "aws_lb_target_group_attachment" "app" {
  count            = length(aws_instance.production_server)
  target_group_arn = aws_lb_target_group.app.arn
  target_id        = aws_instance.production_server[count.index].id
}

resource "aws_lb" "app" {
  name               = "app-load-balancer"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = data.aws_subnets.default.ids
}

# HTTP listener - redirect to HTTPS
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# HTTPS listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = data.aws_acm_certificate.issued.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

### Security Groups

```hcl
# security_groups.tf

# ALB security group - accepts public traffic
resource "aws_security_group" "alb" {
  name        = "alb-security-group"
  description = "Security group for application load balancer"

  # Allow HTTP from anywhere (for redirect)
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow HTTPS from anywhere
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow all outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 security group - only accepts traffic from ALB
resource "aws_security_group" "webserver" {
  name        = "webserver-security-group"
  description = "Security group for web servers"

  # Only allow HTTP from ALB
  ingress {
    description     = "HTTP from ALB"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # Allow SSH for deployment
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Restrict to your IP in production
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Database security group - only from webservers
resource "aws_security_group" "database" {
  name        = "database-security-group"
  description = "Security group for RDS"

  ingress {
    description     = "PostgreSQL from webservers"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.webserver.id]
  }
}
```

### Deployment Script

```bash
#!/bin/bash
# deploy.sh

# Variables
DB_USERNAME=$1
DB_PASSWORD=$2
DOCKER_USER=$3
DOCKER_PASS=$4

# Apply Terraform
terraform init
terraform apply \
  -var="db_username=$DB_USERNAME" \
  -var="db_password=$DB_PASSWORD" \
  -auto-approve

# Extract outputs
terraform output -json > output.json
IP=$(jq -r '.ec2_public_ips.value[0]' output.json)
DB_ENDPOINT=$(jq -r '.db_endpoint.value' output.json)

# Build database URL
DATABASE_URL="postgresql://${DB_USERNAME}:${DB_PASSWORD}@${DB_ENDPOINT}/app_db"

# Copy files to server
scp -o StrictHostKeyChecking=no \
    docker-compose.yml nginx.conf \
    ec2-user@$IP:/home/ec2-user/

# Deploy on server
ssh -o StrictHostKeyChecking=no ec2-user@$IP << EOF
    sudo docker login -u $DOCKER_USER -p $DOCKER_PASS
    export DATABASE_URL=$DATABASE_URL
    sudo DATABASE_URL=\$DATABASE_URL docker-compose up -d
EOF

echo "Deployed to $IP"
```

## NGINX Reverse Proxy and HTTPS

### Basic Reverse Proxy

```nginx
# nginx.conf
events {
    worker_connections 512;
}

http {
    server {
        listen 80;

        location / {
            proxy_pass http://app:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

### HTTPS with Port 80 Redirect

```nginx
# nginx-ssl.conf
events {
    worker_connections 512;
}

http {
    # Redirect HTTP to HTTPS
    server {
        listen 80;
        return 301 https://$host$request_uri;
    }

    # HTTPS server
    server {
        listen 443 ssl http2;

        ssl_certificate     /etc/nginx/ssl/cert.crt;
        ssl_certificate_key /etc/nginx/ssl/cert.key;

        # Security headers
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;

        location / {
            proxy_pass http://app:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### Microservices Routing

```nginx
# nginx-microservices.conf
events {
    worker_connections 512;
}

http {
    server {
        listen 80;

        # Auth service endpoints
        location /api/v1/auth/ {
            proxy_pass http://auth:8081/api/v1/auth/;
        }

        location /api/v1/users/ {
            proxy_pass http://auth:8081/api/v1/users/;
        }

        # Main app endpoints
        location /api/v1/ {
            proxy_pass http://app:8080/api/v1/;
        }

        # Frontend (SPA)
        location / {
            root /usr/share/nginx/html;
            include /etc/nginx/mime.types;
            try_files $uri $uri/ /index.html;
        }
    }
}
```

### Self-Signed Certificate for Development

```bash
# Generate self-signed certificate
openssl req -x509 -days 365 -nodes -newkey rsa:2048 \
    -keyout ./ssl/self.key \
    -out ./ssl/self.crt \
    -subj "/CN=localhost"
```

```yaml
# docker-compose with SSL
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx-ssl.conf:/etc/nginx/nginx.conf
      - ./ssl/self.crt:/etc/nginx/ssl/cert.crt
      - ./ssl/self.key:/etc/nginx/ssl/cert.key
    depends_on:
      - app
```

### Docker Compose with NGINX and Services

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - ingress

  ingress:
    image: myregistry/compute-unit:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://cache:6379
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - postgres
      - cache

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: app_db
    volumes:
      - postgres_data:/var/lib/postgresql/data

  cache:
    image: redis:7-alpine

volumes:
  postgres_data:
```

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: cargo profiles, feature flags, build configuration
- **[architecture.md](architecture.md)** — Workspace design, tracing setup, production patterns
- **[web-apis.md](web-apis.md)** — Web service deployment, health checks, CORS configuration
- **[services.md](services.md)** — Docker Compose multi-service, service discovery, Redis infrastructure
- **[testing.md](testing.md)** — CI/CD test configuration, integration test setup
