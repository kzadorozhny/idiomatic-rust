# Documentation & Naming

Sources: [Guide on how to write documentation for a Rust crate](https://blog.guillaume-gomez.fr/articles/2020-03-12+Guide+on+how+to+write+documentation+for+a+Rust+crate), [Teaching libraries through good documentation](https://deterministic.space/teaching-libraries.html), [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) (C-WORD-ORDER, C-CASE, C-CRATE-DOC), [Canonical's Rust Best Practices](https://canonical.github.io/rust-best-practices/).

---

## DOC-01 (MUST) — Every public item gets a doc comment

`pub fn`, `pub struct`, `pub enum`, `pub trait`, `pub const` — all need a `///` block. Crate-level docs go in `//!` at the top of `lib.rs`. Enforce with `#![deny(missing_docs)]` in libraries.

```rust
/// A persistent key-value store backed by `sled`.
///
/// `Store` is cheap to clone — it wraps an `Arc` internally.
#[derive(Clone)]
pub struct Store { … }
```

## DOC-02 (MUST) — Doc the first line as a single-sentence summary

`cargo doc` shows only the first sentence in the index. Make it count. Start with a verb in third person ("Returns…", "Reads…", "Constructs…").

```rust
// BAD
/// This function is for getting users from the database.
///   ↑ "this function is for" is filler.
pub fn get_users() -> … { … }

// GOOD
/// Returns all users registered in the database.
pub fn get_users() -> … { … }
```

## DOC-03 (SHOULD) — `# Errors` section on every fallible public function

```rust
/// Connects to the given address with the configured TLS settings.
///
/// # Errors
///
/// Returns [`ConnectError::Tls`] if the TLS handshake fails, or
/// [`ConnectError::Io`] if the underlying socket cannot be opened.
pub fn connect(&self, addr: SocketAddr) -> Result<Conn, ConnectError> { … }
```

Enforced by `clippy::missing_errors_doc`.

## DOC-04 (SHOULD) — `# Panics` section on anything that can panic

Even if "it shouldn't in practice." If the function calls `unwrap`, `expect`, indexes a slice, or divides by a runtime value, document the panic condition.

```rust
/// Returns the median of the slice.
///
/// # Panics
///
/// Panics if `values` is empty.
pub fn median(values: &[f64]) -> f64 { … }
```

Better: change the signature to `Option<f64>` and remove the panic. Documenting the panic is the second-best option.

## DOC-05 (MUST) — `# Safety` section on every `pub unsafe fn`

Non-negotiable. The safety section documents what the caller must guarantee.

```rust
/// Reads `len` bytes from the raw pointer.
///
/// # Safety
///
/// - `ptr` must be valid for reads of `len` bytes.
/// - `ptr` must be properly aligned for `u8`.
/// - The memory at `ptr` must not be mutated for the duration of the returned slice.
pub unsafe fn read_raw(ptr: *const u8, len: usize) -> &'static [u8] { … }
```

Enforced by `clippy::missing_safety_doc`.

## DOC-06 (SHOULD) — Include `# Examples` with working doctests

Doctests are also unit tests. They catch API drift.

```rust
/// Truncates the string to at most `max` characters.
///
/// # Examples
///
/// ```
/// # use mycrate::truncate;
/// assert_eq!(truncate("hello, world", 5), "hello");
/// assert_eq!(truncate("hi", 5), "hi");
/// ```
pub fn truncate(s: &str, max: usize) -> &str { … }
```

Use `# ` to hide setup lines that distract from the example.

## DOC-07 (SHOULD) — Use intra-doc links

`[`MyType`]`, `[`MyType::method`]`, `[`crate::module::Thing`]` get rendered as live links and break the build when the target moves.

```rust
/// See also [`Client::connect`] and [`ConnectError`].
```

## DOC-08 (MAY) — Crate-level `//!` doc explains the *concept*, not the API

The crate root is the elevator pitch: what problem does this crate solve, what's the headline example, what's the mental model. Detailed API docs go on individual items.

```rust
//! # mycrate
//!
//! A small DSL for declaratively configuring HTTP clients.
//!
//! ```rust
//! let client = mycrate::Client::builder()
//!     .timeout(Duration::from_secs(5))
//!     .build()?;
//! ```
//!
//! See [`Client`] for the main entry point.
```

## DOC-09 (SHOULD) — Document non-obvious invariants and trade-offs

Doc comments are not just for "what the function returns." Use them for:
- Complexity (`O(n log n)` time, `O(1)` space).
- Allocation behavior ("does not allocate if `Cow::Borrowed`").
- Thread-safety ("safe to call concurrently from multiple threads").
- Cancellation safety in async functions.

## DOC-10 (MUST) — Naming: snake_case for items, CamelCase for types

| Item | Convention |
| --- | --- |
| Modules, functions, variables, methods, fields | `snake_case` |
| Types, traits, enum variants, type parameters | `CamelCase` |
| Constants, statics | `SCREAMING_SNAKE_CASE` |
| Lifetimes | `'lowercase_short` |
| Macros | `snake_case!` |
| Crate names | `snake_case` (avoid `-`; the crate name in `Cargo.toml` uses `-`, in code it's `_`) |

`Clippy` and `rustfmt` enforce most of this; flag deviations.

## DOC-11 (SHOULD) — Word order: getters drop `get_`

`get_x()` is C-style. In Rust, `x()` returns a reference, `x_mut()` returns mutable, `into_x()` consumes, `set_x()` sets.

```rust
// BAD
fn get_name(&self) -> &str { … }
fn get_name_mut(&mut self) -> &mut String { … }
fn set_name(&mut self, n: String) { … }

// GOOD
fn name(&self) -> &str { … }
fn name_mut(&mut self) -> &mut String { … }
fn set_name(&mut self, n: String) { … }
```

Exception: when "get" disambiguates (`get_or_insert`).

## DOC-12 (SHOULD) — Type names: noun phrases. Trait names: capabilities or relationships

```rust
// Types
struct HttpClient;      // GOOD — noun
struct Connect;         // BAD  — verb

// Traits
trait Read { … }        // GOOD — capability
trait Reader { … }      // OK   — agent noun (still common)
trait IsConnectable { … } // BAD — predicate; prefer `Connect`
```

The standard library convention: traits that describe behavior are verbs (`Read`, `Write`, `Iterator`, `Display`). Traits that describe relationships are nouns (`AsRef`, `From`).

## DOC-13 (SHOULD) — Avoid abbreviations except universally-known ones

`url`, `http`, `db`, `ctx` (in narrow scopes), `cfg`, `tmp`, `id` are fine. `usr_mgr_svc` is not.

```rust
// BAD
fn proc_evt(e: &Evt) { … }

// GOOD
fn process_event(event: &Event) { … }
```

## DOC-14 (MAY) — Use `_unused` prefix, not just `_`, when the variable has meaning

```rust
// OK in a closure with no body
let _ = items.iter().for_each(|_| {});

// BAD — loses information
fn handle(_: Request) { … }

// GOOD
fn handle(_request: Request) { … }
```

The `_request` name still shows up in error messages and IDE help.

## DOC-15 (SHOULD) — README + `cargo doc` should agree

The README often duplicates the crate-root doc. Either share it (with `#![doc = include_str!("../README.md")]`) or accept the maintenance cost and keep them in sync.
