# API Design

Sources: [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/), [Elegant Library APIs in Rust](https://deterministic.space/elegant-apis-in-rust.html), [Rust traits for developer friendly libraries](https://benashford.github.io/blog/2015/05/24/rust-traits-for-developer-friendly-libraries/), [Nine Rules for Elegant Rust Library APIs](https://www.youtube.com/watch?v=6-8-9ZV-2WQ), [Ergonomic APIs for hard problems](https://www.youtube.com/watch?v=Phk0C-kLlho), [Teaching libraries through good documentation](https://deterministic.space/teaching-libraries.html), [Tricks of the Trait](https://www.youtube.com/watch?v=7DOYtnCXucw).

---

## API-01 (MUST) — Derive `Debug` on every public type

Without `Debug`, callers can't `println!("{:?}", x)`, `assert_eq!` in tests, or use logging macros. Cost: nearly nothing. Default to deriving it.

```rust
// BAD
pub struct Config { /* fields */ }

// GOOD
#[derive(Debug)]
pub struct Config { /* fields */ }
```

For types with sensitive fields, implement `Debug` manually and redact those fields rather than skipping the trait entirely.

## API-02 (SHOULD) — Derive the standard quartet where it makes sense

For value-like types: `Debug, Clone, PartialEq, Eq` is the baseline. Add `Hash` if it'll be a map key, `PartialOrd, Ord` if it'll be sorted, `Copy` only if all fields are `Copy` *and* copying is conceptually cheap.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct UserId(String);
```

## API-03 (SHOULD) — Builder pattern for >3 parameters or many optional fields

A function with 7 parameters where 5 are `Option<T>` is a builder hiding from itself.

```rust
// BAD
pub fn new(host: &str, port: u16, tls: bool, timeout: Option<Duration>,
           retries: Option<u8>, pool_size: Option<usize>, user_agent: Option<String>) -> Client { … }

// GOOD
let client = Client::builder()
    .host("example.com")
    .port(443)
    .tls(true)
    .timeout(Duration::from_secs(30))
    .build()?;
```

For libraries, the [`bon`](https://docs.rs/bon) and `derive_builder` crates generate this. Hand-rolling is fine for small APIs.

## API-04 (SHOULD) — `#[must_use]` on functions whose return value carries meaning

`Result`, `Future`, and types whose construction has no side effect benefit from `#[must_use]`. The compiler warns when callers discard the return.

```rust
#[must_use = "this `Result` may be an `Err` variant, which should be handled"]
pub fn save(&self) -> Result<(), SaveError> { … }
```

Common cases to flag:
- A builder method that returns `self` (the build is wasted if dropped).
- A function returning a `MustUseDrop`-style RAII guard.
- A `Result` that *isn't* `()` and clearly indicates work to do.

## API-05 (SHOULD) — Return concrete types from public functions; take generic inputs

```rust
// BAD — caller has to name an opaque type or `Box<dyn>`
pub fn iter() -> impl Iterator<Item = &str> { … }

// GOOD — concrete struct, fully documented, can implement traits later
pub fn iter(&self) -> Iter<'_> { Iter { … } }
```

Exception: `impl Trait` in return position is fine for combinator-style helpers where naming the concrete type would be painful (closures, iterator chains). But know you're closing the door to future trait impls on that return type.

## API-06 (MUST) — Don't expose internal types in public APIs

If your public function returns `internal::FooParser`, every consumer is now coupled to that path. Wrap or re-export.

```rust
// BAD — leaks impl detail
pub fn parser() -> crate::internal::ParserImpl { … }

// GOOD
pub use crate::internal::ParserImpl as Parser;
pub fn parser() -> Parser { … }
```

## API-07 (SHOULD) — Use `#[non_exhaustive]` on public enums and structs that may grow

```rust
#[non_exhaustive]
pub enum Event { Connected, Disconnected, MessageReceived(String) }
```

Now adding `Reconnecting` later isn't a breaking change — downstream `match`es are forced to include a wildcard arm.

## API-08 (SHOULD) — Sealed traits when you don't want downstream impls

```rust
mod sealed { pub trait Sealed {} }

pub trait MyTrait: sealed::Sealed { /* methods */ }

impl sealed::Sealed for MyType {}
impl MyTrait for MyType { /* … */ }
```

This lets you add required methods later without a breaking change. Use when the trait is conceptually closed (e.g., "things that are HTTP methods").

## API-09 (SHOULD) — `From`/`TryFrom` for value conversions

Don't invent `to_other_type()`/`from_other_type()` when the standard traits exist. They compose with `?`, `.into()`, and trait bounds.

```rust
// BAD
impl Url { pub fn from_string(s: String) -> Result<Self, _> { … } }

// GOOD
impl TryFrom<String> for Url { type Error = ParseError; fn try_from(s: String) -> … }
impl FromStr for Url     { type Err = ParseError;     fn from_str(s: &str) -> …  }
```

## API-10 (MUST) — Constructors named `new`, fallible constructors named `try_new` or just take `&str`

| Pattern | Name |
| --- | --- |
| Infallible constructor | `Type::new(...)` |
| Fallible constructor   | `Type::try_new(...)` or `TryFrom`/`FromStr` |
| Default value          | `impl Default` |
| Conversion from another type | `From`/`TryFrom` |
| Constructor from raw parts | `from_raw_parts` (often `unsafe`) |

Avoid `Type::create`, `Type::build`, `Type::make` — they hint at builders without committing.

## API-11 (SHOULD) — Re-export everything users need from the crate root

A library where users have to write `use mylib::internal::types::config::Config` is painful. Re-export at the root.

```rust
// lib.rs
pub use config::Config;
pub use client::{Client, ClientBuilder};
pub use error::Error;
```

Document the prelude pattern if there are many: `pub mod prelude { pub use crate::{Client, Config, Error}; }`.

## API-12 (SHOULD) — Use the orphan rule to your advantage, but warn about coherence

In your own crate you can impl your trait for foreign types and foreign traits for your types. But don't impl `Display` for a foreign type — that's downstream's job. Conflicts at link-time are nightmares.

## API-13 (MAY) — `Default` implementations that match `new()`

If `MyType::new()` takes no arguments, also `impl Default`. They should produce the same value. This lets `MyType::default()` work in generic contexts (`HashMap`, `serde`).

```rust
impl Default for Config {
    fn default() -> Self { Self::new() }
}
```

## API-14 (SHOULD) — Don't return `&Vec<T>`; return `&[T]`

Returning `&Vec<T>` leaks the container type and prevents the caller from being container-agnostic.

```rust
// BAD
pub fn items(&self) -> &Vec<Item> { &self.items }

// GOOD
pub fn items(&self) -> &[Item] { &self.items }
```

## API-15 (SHOULD) — Iterators over collections, not collections

Returning an iterator lets callers `take(10)`, `filter`, `collect` into a different container.

```rust
// BAD
pub fn names(&self) -> Vec<String> { self.users.iter().map(|u| u.name.clone()).collect() }

// GOOD
pub fn names(&self) -> impl Iterator<Item = &str> + '_ {
    self.users.iter().map(|u| u.name.as_str())
}
```

## API-16 (MUST) — Mark experimental APIs with `#[doc(hidden)]` or a `__private` module

If you need to expose an item to macros or sister crates but don't want it part of your public API, hide it. Don't have an "internal-but-public" tier nobody can tell apart.

## API-17 (MAY) — Newtype wrappers around third-party types in your public API

If you expose `tokio::sync::Mutex<MyType>` directly, bumping your tokio major version is a breaking change for your users. Wrap it.

## API-18 (SHOULD) — Match the standard library's vocabulary

| Operation | Convention |
| --- | --- |
| Convert without copying | `as_*` (`as_str`, `as_bytes`) |
| Convert by allocating  | `to_*` (`to_string`, `to_owned`) |
| Convert by consuming   | `into_*` (`into_inner`, `into_iter`) |
| Get reference          | `get` / `get_mut` |
| Get or panic           | `[]` indexing |
| Iterate by reference   | `iter()` / `iter_mut()` |
| Iterate consuming      | `into_iter()` |
| Length                 | `len()` (not `length()` or `size()`) |
| Empty test             | `is_empty()` (not `len() == 0`) |

Going against these conventions makes the library feel foreign. Flag mismatches.
