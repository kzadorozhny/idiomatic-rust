# Example: A Worked Review

Showing what an AI reviewer using this skill produces. The input is a small Rust function; the output applies the rules above.

---

## Input

```rust
// from src/cache.rs
use std::sync::{Arc, Mutex};
use std::collections::HashMap;

pub struct Cache {
    inner: Arc<Mutex<HashMap<String, String>>>,
}

impl Cache {
    pub fn new() -> Cache {
        Cache { inner: Arc::new(Mutex::new(HashMap::new())) }
    }

    pub fn get(&self, k: String) -> String {
        let map = self.inner.lock().unwrap();
        let v = map.get(&k).unwrap();
        return v.clone();
    }

    pub fn put(self, k: String, v: String, overwrite: bool) -> bool {
        let mut map = self.inner.lock().unwrap();
        if map.contains_key(&k) == true && overwrite == false {
            return false;
        }
        map.insert(k, v);
        return true;
    }
}
```

---

## Reviewer output

### Summary

The cache works but has several idiomatic issues: it panics on missing keys, takes owned `String`s where `&str` would do, uses a boolean parameter that should be an enum, and has style issues (`== true`, trailing `return`, `&Vec`-style ownership). Also missing `Default`, `Debug`, and any documentation.

### MUST

**[R-1 / ERR-02] cache.rs:14 — `.unwrap()` panic on missing key**

```rust
let v = map.get(&k).unwrap();   // panics if key absent
```

A cache `get` should return `Option<String>`, not panic when the value isn't there. Cache misses are the *whole point* of caches.

```rust
pub fn get(&self, k: &str) -> Option<String> {
    let map = self.inner.lock().unwrap();
    map.get(k).cloned()
}
```

Source: [Three Kinds Of Unwrap](https://zkrising.com/writing/three-unwraps/).

**[API-01 / DOC-01] cache.rs:5 — `Cache` is missing `Debug` and any documentation**

```rust
/// An in-memory string cache, cheaply clonable.
#[derive(Clone, Debug, Default)]
pub struct Cache { … }
```

(Manual `Debug` if you don't want to leak entries.)

**[ANTI-21] cache.rs:21–22 — `== true` / `== false` against booleans**

```rust
if map.contains_key(&k) == true && overwrite == false { … }

// →
if map.contains_key(&k) && !overwrite { … }
```

Caught by `clippy::bool_comparison`.

### SHOULD

**[R-2 / TYPE-03 / OWN-10] cache.rs:13, cache.rs:18 — Owned `String` arguments**

`get(&self, k: String)` forces every caller to allocate. `put(self, k: String, v: String, ...)` is fine for `v` (you'll store it) but the cache shouldn't be consumed.

```rust
pub fn get(&self, k: &str) -> Option<String> { … }
pub fn put(&self, k: impl Into<String>, v: impl Into<String>, mode: WriteMode) -> bool { … }
```

Source: [Taking string arguments in Rust](http://xion.io/post/code/rust-string-args.html).

**[OWN-04] cache.rs:18 — `put` takes `self` by value**

You almost certainly meant `&self`. As written, calling `put` consumes the cache.

```rust
pub fn put(&self, k: String, v: String, mode: WriteMode) -> bool { … }
```

**[R-3 / TYPE-02] cache.rs:18 — Boolean `overwrite` parameter**

`overwrite: bool` doesn't read well at call sites: `cache.put(k, v, true)` — true what? Replace with an enum.

```rust
pub enum WriteMode { Insert, Overwrite }

pub fn put(&self, k: String, v: String, mode: WriteMode) -> bool { … }

// call sites become:
cache.put(k, v, WriteMode::Overwrite);
```

Source: [Rust Patterns: Enums Instead Of Booleans](https://blakesmith.me/2019/05/07/rust-patterns-enums-instead-of-booleans.html).

**[ITER-11] cache.rs:21–24 — `contains_key` + `insert` is two lookups**

```rust
if map.contains_key(&k) && overwrite == false { return false; }
map.insert(k, v);

// →
match map.entry(k) {
    Entry::Occupied(_) if matches!(mode, WriteMode::Insert) => return false,
    Entry::Occupied(mut e) => { e.insert(v); }
    Entry::Vacant(e) => { e.insert(v); }
}
```

One hash lookup instead of two. Caught by `clippy::map_entry`.

**[R-10 / OWN-11] cache.rs:13–15 — Lock guard held longer than needed**

Minor here, but a habit to build: scope the guard to the smallest region.

```rust
pub fn get(&self, k: &str) -> Option<String> {
    let map = self.inner.lock().unwrap();
    map.get(k).cloned()                 // guard dropped on return
}
```

(In this case it's already fine because the function ends; flagging the pattern for future methods.)

### MAY

**[ANTI-19] cache.rs:16, cache.rs:23, cache.rs:26 — Trailing `return` at end of function**

```rust
return v.clone();   // → v.clone()
return false;       // → false (when last expression of an arm)
return true;        // → true
```

**[API-13] cache.rs — Missing `Default`**

`Cache::new()` takes no args, so `impl Default` is a one-liner that lets `Cache` work in generic `Default` contexts and lets users write `Cache::default()`.

```rust
impl Default for Cache {
    fn default() -> Self { Self::new() }
}
```

Or derive `Default` directly:

```rust
#[derive(Clone, Default)]
pub struct Cache { inner: Arc<Mutex<HashMap<String, String>>> }
```

### Strengths

- Wrapping the `HashMap` in `Arc<Mutex<…>>` and *not* exposing it is the right call — callers see `Cache`, not raw locking primitives.
- Using `Arc` makes `Cache` cheaply cloneable so handles can be passed to multiple tasks.
- Method signatures are short and the type takes a focused responsibility.

---

## Putting it all together

```rust
//! A small in-memory string cache.

use std::collections::HashMap;
use std::collections::hash_map::Entry;
use std::sync::{Arc, Mutex};

/// Controls whether `put` replaces an existing entry.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum WriteMode {
    /// Only insert if the key is absent.
    Insert,
    /// Always insert, replacing any existing entry.
    Overwrite,
}

/// An in-memory string cache. Cheap to clone — handles share the underlying map.
#[derive(Clone, Default)]
pub struct Cache {
    inner: Arc<Mutex<HashMap<String, String>>>,
}

impl Cache {
    /// Returns a new, empty cache.
    pub fn new() -> Self {
        Self::default()
    }

    /// Returns a copy of the value for `key`, if present.
    pub fn get(&self, key: &str) -> Option<String> {
        let map = self.inner.lock().expect("cache mutex poisoned");
        map.get(key).cloned()
    }

    /// Inserts `value` under `key`.
    ///
    /// Returns `true` if the cache was modified. In [`WriteMode::Insert`] mode,
    /// returns `false` if the key already exists.
    pub fn put(&self, key: impl Into<String>, value: impl Into<String>, mode: WriteMode) -> bool {
        let mut map = self.inner.lock().expect("cache mutex poisoned");
        match map.entry(key.into()) {
            Entry::Occupied(_) if mode == WriteMode::Insert => false,
            Entry::Occupied(mut e) => { e.insert(value.into()); true }
            Entry::Vacant(e)       => { e.insert(value.into()); true }
        }
    }
}
```

That's the difference between "compiles" and "idiomatic." Every change cites a specific source from the corpus.
