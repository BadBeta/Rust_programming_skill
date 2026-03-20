# Rust Programming Skill

A comprehensive Claude Code skill for idiomatic Rust development. Covers ownership, traits, generics, async/await, error handling, serde, tracing, macros, design patterns, and production best practices. Current for Rust 2024 edition.

## Installation

```bash
git clone https://github.com/badbeta/Rust_programming_skill.git ~/.claude/skills/rust-programming
```

The skill auto-activates for any `.rs` file via the `globs: "*.rs"` frontmatter.



## Activation

The skill activates automatically when:
- Working with `.rs` files
- Writing or reviewing Rust code
- Planning Rust project architecture

Only the core SKILL.md get loaded into context at first. Other files are used as needed.



## Better results by adding this to the first project or refactor prompt:

**Do not use agents for planning or implementation. Do it yourself.**

**ALWAYS load the rust-programming skill and read both SKILL.md and architecture.md before starting to plan architecture or writing any code. No exceptions, so do it right now!** The core skill contains essential rules, decision frameworks, and key patterns. Use architecture.md to decide on workspace layout, crate boundaries, trait-based DI, and growing architecture stages. Then consult supporting files:

  - Use language-patterns.md for ownership patterns, Cow\<T\>, borrow
      splitting, entry API, let-else, if-let chains, closure capture
      semantics, and ? operator chains with context
  - Prefer Result/Option with ? over unwrap/expect, iterator chains
      over index loops, borrowing over cloning, and typed errors
      over string errors
  - Read async-concurrency.md when designing async systems, choosing
      channels, using tokio, rayon, or actors, and understanding
      Pin/Unpin
  - Use type-system.md for trait patterns, GATs, const generics,
      type state, sealed traits, and lifetime patterns
  - Use quick-reference.md for String, Vec, HashMap, Iterator, Option,
      Result, File/Path operations, and common trait implementations
      — choose the right method, don't reinvent it
  - Follow error-handling.md for thiserror/anyhow/color-eyre decisions,
      multi-layer error translation, and error conversion chains
  - Follow serde-serialization.md for derive attributes, enum
      representations, custom serialization, and format selection
  - Follow documentation.md for rustdoc conventions, doc tests,
      intra-doc links, and standard sections (Examples, Errors,
      Panics, Safety)
  - Use testing.md for unit, integration, mockall, insta, proptest,
      cargo-fuzz, and async test patterns
  - Use web-apis.md for axum extractors, middleware, WebSocket,
      authentication, and database integration
  - Use deployment.md for cargo profiles, Docker builds, CI/CD,
      cross-compilation, tracing subscribers, and metrics
  - Every public type must derive Debug, every public enum should
      use #[non_exhaustive], every unsafe block needs a // SAFETY:
      comment
