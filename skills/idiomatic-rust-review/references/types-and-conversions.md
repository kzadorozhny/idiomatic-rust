# Types & Conversions

Sources: [The Ultimate Guide to Rust Newtypes](https://www.howtocodeit.com/articles/ultimate-guide-rust-newtypes), [Convenient and idiomatic conversions in Rust](https://ricardomartins.cc/2016/08/03/convenient_and_idiomatic_conversions_in_rust), [Taking string arguments in Rust](http://xion.io/post/code/rust-string-args.html), [When should I use String vs &str?](https://steveklabnik.com/writing/when-should-i-use-string-vs-str/), [Creating a Rust function that returns a &str or String](https://hermanradtke.com/2015/05/29/creating-a-rust-function-that-returns-string-or-str.html/), [Rust Patterns: Enums Instead Of Booleans](https://blakesmith.me/2019/05/07/rust-patterns-enums-instead-of-booleans.html), [Math with distances in Rust](https://www.ferrisellis.com/content/rust-implementing-units-for-types/), Rust API Guidelines (C-CONV).

---

## TYPE-01 (MUST) — Newtype interchangeable primitives that represent different domain concepts

Two `u64`s representing a user ID and a duration in milliseconds should not be swappable. Newtypes are zero-cost (`#[repr(transparent)]`) and catch entire classes of bugs at compile time.

```rust
// BAD
fn schedule(user_id: u64, retry_after_ms: u64) { … }
schedule(retry_after_ms, user_id);  // compiles, ships, pages on-call

// GOOD
#[derive(Copy, Clone, Debug, Eq, Hash, PartialEq)]
#[repr(transparent)]
pub struct UserId(pub u64);

#[derive(Copy, Clone, Debug)]
#[repr(transparent)]
pub struct Millis(pub u64);

fn schedule(user_id: UserId, retry_after: Millis) { … }
```

When the validation is non-trivial (email, port range, non-empty string), make the field private and expose a fallible constructor:

```rust
pub struct Port(u16);
impl Port {
    pub fn new(n: u16) -> Result<Self, OutOfRange> {
        if (1024..49152).contains(&n) { Ok(Self(n)) } else { Err(OutOfRange) }
    }
    pub fn get(self) -> u16 { self.0 }
}
```

Now anywhere the code has a `Port`, the value is known-valid.

## TYPE-02 (SHOULD) — Enums instead of booleans for parameters

A function with three `bool` parameters has 8 call-site combinations and zero readability. An enum is self-documenting and extensible.

```rust
// BAD
fn render(html: bool, dark_mode: bool, compress: bool) { … }
render(true, false, true);  // ???

// GOOD
enum Format { Html, Plain }
enum Theme  { Light, Dark }
enum Compression { On, Off }
fn render(format: Format, theme: Theme, compression: Compression) { … }
render(Format::Html, Theme::Light, Compression::On);
```

Bonus: callers who need a fourth format don't break source compatibility — they just gain a variant.

## TYPE-03 (SHOULD) — `&str` parameters, `String` return values

Default rule: take the cheapest type that lets the caller pass what they have; return the type the caller is most likely to want to own.

```rust
// BAD
fn greet(name: String) -> &str { … }     // forces caller to allocate, lifetime headache on return

// GOOD
fn greet(name: &str) -> String { … }
```

Variants:

- Accept *both* owned and borrowed with `impl Into<String>` when you'll store it.
- Accept `impl AsRef<str>` when you'll only read it but want ergonomic call sites.
- Return `Cow<'_, str>` when sometimes you can borrow and sometimes you must allocate.

```rust
pub fn normalize(s: &str) -> Cow<'_, str> {
    if s.chars().all(|c| !c.is_uppercase()) {
        Cow::Borrowed(s)            // no allocation when already normalized
    } else {
        Cow::Owned(s.to_lowercase())
    }
}
```

## TYPE-04 (SHOULD) — `impl AsRef<Path>` for file-path parameters

```rust
// BAD
fn open_log(path: PathBuf) -> io::Result<File> { … }
open_log(PathBuf::from("/var/log/app"));   // explicit conversion

// GOOD
fn open_log(path: impl AsRef<Path>) -> io::Result<File> { … }
open_log("/var/log/app")?;                 // &str
open_log(p)?;                              // &Path
open_log(buf)?;                            // PathBuf
```

Same pattern for `impl AsRef<str>`, `impl AsRef<[u8]>`. The trait is implemented for the obvious types and conversion is zero-cost.

## TYPE-05 (SHOULD) — Implement `From`, not `Into`

`From<A> for B` automatically gives you `Into<B> for A`. The reverse isn't true. Always implement `From` — it composes better and shows up first in docs.

```rust
// GOOD
impl From<u32> for UserId {
    fn from(n: u32) -> Self { UserId(n as u64) }
}
// Now both `UserId::from(7u32)` and `let x: UserId = 7u32.into()` work.
```

For fallible conversions, do the same with `TryFrom`.

## TYPE-06 (SHOULD) — `TryFrom` for fallible conversions, not custom `parse_*`

Once `TryFrom` exists, the `?` operator and trait-based generics work everywhere. Custom `from_string` / `parse_email` methods scatter the same logic.

```rust
// BAD
impl Email {
    pub fn parse(s: &str) -> Result<Self, EmailError> { … }
}

// GOOD
impl TryFrom<&str> for Email {
    type Error = EmailError;
    fn try_from(s: &str) -> Result<Self, Self::Error> { … }
}
impl FromStr for Email {  // bonus: enables `"x@y".parse::<Email>()`
    type Err = EmailError;
    fn from_str(s: &str) -> Result<Self, Self::Err> { Self::try_from(s) }
}
```

## TYPE-07 (MUST) — Don't `impl<T> From<T> for MyError`; it conflicts and surprises

A blanket `From` on your error type breaks downstream attempts to add their own `From` impls and hides where errors actually originate. Use specific `From` impls for the underlying error types only.

## TYPE-08 (SHOULD) — Prefer `Vec<T>` slice parameters as `&[T]`

```rust
// BAD
fn sum(v: Vec<i64>) -> i64 { v.iter().sum() }
sum(vec![1, 2, 3]);    // forced allocation
// sum(&array);        // doesn't compile

// GOOD
fn sum(v: &[i64]) -> i64 { v.iter().sum() }
sum(&[1, 2, 3]);
sum(&my_vec);
sum(&my_array[..]);
```

## TYPE-09 (SHOULD) — `impl IntoIterator<Item = T>` over `Vec<T>` when consuming

Lets the caller pass arrays, iterators, or pre-built `Vec`s without converting.

```rust
// BAD
fn bulk_insert(items: Vec<Item>) { … }

// GOOD
fn bulk_insert(items: impl IntoIterator<Item = Item>) { … }
bulk_insert([item1, item2]);
bulk_insert(stream.map(parse));
bulk_insert(existing_vec);
```

## TYPE-10 (SHOULD) — Newtypes deserve `Debug`, `Clone` (if cheap), and sometimes `Display`

Default derives for a value newtype:

```rust
#[derive(Debug, Clone, Copy, Eq, Hash, Ord, PartialEq, PartialOrd)]
pub struct Order(u64);
```

`Display` is for *user-facing* formatting. `Debug` is for developer-facing. Don't conflate them. A `UserId` probably wants `Display` to print as `"user#42"`, and `Debug` to print as `UserId(42)`.

## TYPE-11 (MAY) — Derive `serde::Serialize`/`Deserialize` only on data that crosses serialization boundaries

Sprinkling `#[derive(Serialize, Deserialize)]` everywhere couples your internal types to your wire format. Keep a separate DTO layer and `impl From` between them when the public/wire shape differs from the domain.

## TYPE-12 (MUST) — Never use floating-point for money

`f64` can't represent `0.1` exactly. Use `i64` cents, `rust_decimal::Decimal`, or a newtype around an integer denomination.

```rust
// BAD
let total: f64 = items.iter().map(|i| i.price).sum();

// GOOD
struct Cents(i64);
let total: Cents = items.iter().map(|i| i.price).fold(Cents(0), |a, b| Cents(a.0 + b.0));
```

## TYPE-13 (SHOULD) — Avoid `String` fields when an enum will do

```rust
// BAD
struct Job { status: String }   // "pending" / "Running" / "DONE  " — typos compile

// GOOD
enum Status { Pending, Running, Done, Failed { reason: String } }
struct Job { status: Status }
```

## TYPE-14 (MAY) — Use `NonZeroU32` and friends when zero is invalid

`Option<NonZeroU32>` is the same size as `u32` (niche optimization), and `NonZeroU32::new(0)` returns `None`. Free safety + free space.
