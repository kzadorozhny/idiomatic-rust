# Error Handling

Sources: [Error Handling in Rust (BurntSushi)](https://burntsushi.net/rust-error-handling/), [Context-preserving error handling](https://kazlauskas.me/entries/errors), [Designing Error Types in Rust Libraries](https://d34dl0ck.me/rust-bites-designing-error-types-in-rust-libraries/), [To panic or not to panic](https://www.ncameron.org/blog/to-panic-or-not-to-panic/), [Three Kinds Of Unwrap](https://zkrising.com/writing/three-unwraps/), [Practical guide to Error Handling in Rust](https://dev-state.com/posts/error_handling/), [Wrapping errors in Rust](https://edgl.dev/blog/wrapping-errors-in-rust/), Rust API Guidelines.

---

## ERR-01 (MUST) — `Result<T, E>` for recoverable errors, `panic!` for bugs

A panic means "this should be impossible — fix the program." A `Result::Err` means "this can happen at runtime and the caller should decide." Anything that depends on user input, the filesystem, the network, or untrusted data is `Result`.

```rust
// BAD — a malformed config file kills the process
let port: u16 = config_str.parse().unwrap();

// GOOD
let port: u16 = config_str.parse()
    .map_err(|e| ConfigError::InvalidPort { value: config_str.into(), source: e })?;
```

## ERR-02 (MUST) — No `unwrap` / `expect` on fallible operations in library code

A library that panics on input it disagrees with is broken. The exceptions:

1. **Tests, examples, benchmarks** — fine.
2. **Compile-time guarantees the type system can't see** — use `.expect("invariant: ...")` and document the invariant in the message.
3. **`main()` in a binary** — prefer returning `Result` so the runtime error formatting kicks in.

```rust
// BAD — panic exposed in a library
pub fn parse_packet(bytes: &[u8]) -> Packet {
    Packet { kind: bytes[0], len: bytes[1] }  // panics on short input
}

// GOOD
pub fn parse_packet(bytes: &[u8]) -> Result<Packet, ParseError> {
    let kind = *bytes.first().ok_or(ParseError::Truncated)?;
    let len  = *bytes.get(1).ok_or(ParseError::Truncated)?;
    Ok(Packet { kind, len })
}
```

## ERR-03 (MUST) — Use `?` to propagate, not `match … return Err(e)`

```rust
// BAD
let bytes = match read_file(p) {
    Ok(b) => b,
    Err(e) => return Err(e.into()),
};

// GOOD
let bytes = read_file(p)?;
```

`?` calls `From::from` on the error, so a single `#[from]` in your error enum makes propagation transparent.

## ERR-04 (SHOULD) — Library errors: enum with `thiserror`

Callers must be able to *match on* what went wrong. Use a concrete enum, derive `thiserror::Error`, and chain causes with `#[source]`.

```rust
// GOOD
#[derive(Debug, thiserror::Error)]
pub enum LoadError {
    #[error("config file not found at {path}")]
    NotFound { path: PathBuf },

    #[error("config file is malformed")]
    Parse(#[from] toml::de::Error),

    #[error("I/O error reading {path}")]
    Io {
        path: PathBuf,
        #[source]
        source: std::io::Error,
    },
}
```

Anti-patterns to flag:

- `pub fn ... -> Result<T, Box<dyn Error>>` in a library — opaque.
- `pub fn ... -> anyhow::Result<T>` in a library — same problem, just nicer ergonomics for the wrong audience.
- A single mega-enum `Error { Other(String) }` — callers can't distinguish failure modes.

## ERR-05 (SHOULD) — Application errors: `anyhow` (or `eyre`) with `.context()`

In a binary, you typically log the error and exit. `anyhow` makes that fluent:

```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let cfg = load_config(&path)
        .with_context(|| format!("loading config from {}", path.display()))?;
    let server = Server::bind(cfg.addr)
        .context("binding server")?;
    server.run().context("serving requests")?;
    Ok(())
}
```

Every `?` boundary is an opportunity for `.context()`. Bare `?` with no context yields error chains that read like "I/O error: I/O error: not found" — useless.

## ERR-06 (SHOULD) — Don't swallow errors with `.ok()` or `let _ =`

```rust
// BAD — error silently discarded
let _ = file.flush();

// GOOD — make the choice explicit and audited
if let Err(e) = file.flush() {
    tracing::warn!(?e, "flush failed; data may be lost");
}
```

The only legitimate `let _ = ...` is when the function returns `Result` purely for compatibility (e.g., `writeln!` to a `String` that can't fail) and you genuinely don't care.

## ERR-07 (SHOULD) — `Option::ok_or_else` over `ok_or` when the error allocates

`ok_or(e)` evaluates `e` eagerly even when the `Option` is `Some`.

```rust
// BAD — formats the string every call, even when found
map.get(&k).ok_or(format!("key not found: {k}"))?;

// GOOD
map.get(&k).ok_or_else(|| format!("key not found: {k}"))?;
```

Same applies to `unwrap_or` vs `unwrap_or_else`, and to `.map_err(|e| heavy(e))` vs cheap closures.

## ERR-08 (MAY) — `Result<T, E>` is more idiomatic than `Option<T>` for "this might fail with a reason"

Use `Option<T>` for "this value is sometimes absent and that's normal" (lookup, parsing optional fields). Use `Result<T, E>` when there's a *reason* for the absence the caller might want to inspect or log.

```rust
// Marginal — caller has no idea why
fn find_user(id: UserId) -> Option<User> { … }

// Better when "not found" is one of several outcomes
fn load_user(id: UserId) -> Result<User, LoadError> { … }
```

## ERR-09 (SHOULD) — Document `# Errors` and `# Panics` in rustdoc

Every public fallible function should have an `# Errors` section. Every function that *can* panic — even if "shouldn't in practice" — needs `# Panics`.

```rust
/// Reads the entire file as UTF-8.
///
/// # Errors
///
/// Returns [`LoadError::Io`] if the file cannot be opened, or
/// [`LoadError::Parse`] if the contents are not valid UTF-8.
///
/// # Panics
///
/// Panics if `path` contains an interior NUL byte.
pub fn load(path: impl AsRef<Path>) -> Result<String, LoadError> { … }
```

`clippy::missing_errors_doc` and `clippy::missing_panics_doc` enforce this.

## ERR-10 (MUST) — Panic-safety in `unsafe` and `Drop`

A panic that unwinds across an FFI boundary is UB. A panic in a `Drop` impl while already unwinding aborts the program. Flag any `unsafe extern "C" fn` that doesn't either `catch_unwind` or use `#[unwind(abort)]`-equivalent settings, and any `Drop` that calls fallible code without handling panics.

## ERR-11 (MAY) — Avoid `.unwrap_or_default()` when it hides bugs

`u64::default()` is `0`. `String::default()` is `""`. If `0` or `""` is a valid value that means "absent," that's fine. If a `0` would silently corrupt the calculation, prefer to surface the error.

## ERR-12 (SHOULD) — `From` impls for error conversions, not manual `.map_err`

```rust
// VERBOSE
fn load() -> Result<Cfg, LoadError> {
    let s = std::fs::read_to_string(p).map_err(LoadError::Io)?;
    let c: Cfg = toml::from_str(&s).map_err(LoadError::Parse)?;
    Ok(c)
}

// IDIOMATIC — let `?` do the conversion
#[derive(thiserror::Error, Debug)]
enum LoadError {
    #[error(transparent)] Io(#[from] std::io::Error),
    #[error(transparent)] Parse(#[from] toml::de::Error),
}

fn load(p: &Path) -> Result<Cfg, LoadError> {
    let s = std::fs::read_to_string(p)?;
    let c: Cfg = toml::from_str(&s)?;
    Ok(c)
}
```

Caveat: when context matters (which file? which line?), keep the explicit `.map_err` to attach it.

## Quick reference: when to use what

| Situation | Choice |
| --- | --- |
| Bug, broken invariant | `panic!` / `unreachable!` / `debug_assert!` |
| Recoverable failure, caller wants to match | `Result<T, MyError>` (enum) |
| Recoverable failure, caller just logs | `anyhow::Result<T>` |
| Optional value, no reason needed | `Option<T>` |
| Compile-time-known impossibility | `.expect("explain why")` |
| Test or example | `.unwrap()` is fine |
