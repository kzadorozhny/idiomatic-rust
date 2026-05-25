# Unsafe & Performance

Sources: [Pitfalls of Safe Rust](https://corrode.dev/blog/pitfalls-of-safe-rust/), [High Assurance Rust](https://highassurance.rs/), [Refactoring Rust Transpiled from C](https://immunant.com/blog/2020/09/transpiled_c_safety/), [The balance between cost, useability and soundness in C bindings](https://web.archive.org/web/20190509123207/https://cobrand.github.io/rust/sdl2/2017/05/07/the-balance-between-soundness-cost-useability.html), [Rust Atomics and Locks](https://marabos.nl/atomics/), Rust API Guidelines.

---

## UNSAFE-01 (MUST) — Every `unsafe` block needs a `// SAFETY:` comment

The comment documents *why* the unsafe operation is sound, citing the invariants the caller must uphold (for `unsafe fn`) or that the surrounding code ensures (for `unsafe {}` blocks).

```rust
// BAD
unsafe { std::slice::from_raw_parts(ptr, len) }

// GOOD
// SAFETY: `ptr` is non-null and points to `len` consecutive initialized `u8`s,
// because it came from `Vec::as_ptr` of a vec we own with `len` elements.
unsafe { std::slice::from_raw_parts(ptr, len) }
```

This is enforced by `clippy::undocumented_unsafe_blocks` at the `pedantic` level, which libraries should consider promoting to `deny`.

## UNSAFE-02 (MUST) — `pub unsafe fn` requires a `# Safety` doc section

See `DOC-05`. The two together (rustdoc `# Safety` + `// SAFETY:` at call sites) form the safety contract.

## UNSAFE-03 (MUST) — `unsafe` blocks should be small

Wrap only the operation that needs it, not the surrounding logic. This makes auditing tractable.

```rust
// BAD — entire function in unsafe
unsafe fn process(ptr: *const u8, len: usize) -> u32 {
    let s = std::slice::from_raw_parts(ptr, len);
    let mut sum = 0;
    for b in s { sum += *b as u32; }
    sum
}

// GOOD — unsafe is one line
fn process(ptr: *const u8, len: usize) -> u32 {
    // SAFETY: caller guarantees `ptr` points to `len` valid bytes.
    let s = unsafe { std::slice::from_raw_parts(ptr, len) };
    s.iter().map(|&b| b as u32).sum()
}
```

## UNSAFE-04 (MUST) — Don't `mem::transmute` when `as`, `cast`, or `bytemuck` will do

`transmute` is the nuclear option. Most cases have safer alternatives:

- Bit reinterpret of POD types → `bytemuck::cast` / `bytemuck::cast_slice`.
- Numeric conversion → `as` (or `TryFrom` if you want overflow detection).
- Pointer cast → `*const T as *const U`.
- `&T` ↔ `&U` of same layout → `&*(p as *const U)` with documented invariants.

Flag any `mem::transmute` that doesn't have a `// SAFETY:` comment explicitly justifying why the alternatives don't apply.

## UNSAFE-05 (MUST) — `unwrap_unchecked` / `get_unchecked` / `*_unchecked` are footguns

These bypass safety checks for performance. Acceptable only with:
1. A `// SAFETY:` comment.
2. A benchmark showing the safe version is actually a bottleneck.
3. A `debug_assert!` of the invariant at the same call site.

```rust
// GOOD if and only if profiled
debug_assert!(idx < v.len());
// SAFETY: bounds checked by caller per the function precondition.
let x = unsafe { *v.get_unchecked(idx) };
```

If the function is hot enough to warrant `_unchecked`, profile it. Otherwise use the safe variant.

## UNSAFE-06 (MUST) — Don't violate aliasing rules with `*mut` ↔ `&mut`

Holding a `&mut T` and a `&T` (or two `&mut T`s) to the same memory is undefined behavior, even if you got there through raw pointers. Tools: `miri`, `cargo +nightly miri test`.

## UNSAFE-07 (SHOULD) — Use `MaybeUninit<T>` for partial initialization, not `mem::uninitialized`

`mem::uninitialized` was deprecated for soundness reasons. `MaybeUninit` is the modern equivalent and lets the type system track initialization state.

```rust
// DEPRECATED & UB-PRONE
let x: T = unsafe { mem::uninitialized() };

// CORRECT
let mut x = MaybeUninit::<T>::uninit();
// ... initialize fields ...
let x = unsafe { x.assume_init() };  // SAFETY: all fields initialized above
```

## UNSAFE-08 (SHOULD) — Encapsulate `unsafe` behind a safe API

If your crate has 50 `unsafe` blocks, the user trusts your crate, not Rust. Concentrate the `unsafe` in a minimal core; expose a safe API on top.

```rust
// Safe wrapper
pub struct Buffer { ptr: NonNull<u8>, len: usize }

impl Buffer {
    pub fn new(size: usize) -> Self { … }
    pub fn write(&mut self, offset: usize, bytes: &[u8]) -> Result<(), Error> {
        if offset + bytes.len() > self.len { return Err(Error::OutOfBounds); }
        // SAFETY: bounds checked above.
        unsafe { ptr::copy_nonoverlapping(bytes.as_ptr(), self.ptr.as_ptr().add(offset), bytes.len()); }
        Ok(())
    }
}
```

## UNSAFE-09 (MUST) — FFI: don't unwind across the boundary

A Rust panic that crosses an `extern "C"` boundary is UB. Use `std::panic::catch_unwind` or compile with `panic = "abort"`.

```rust
#[no_mangle]
pub extern "C" fn do_thing() -> i32 {
    std::panic::catch_unwind(|| {
        // Rust code that might panic
        compute()
    }).unwrap_or(-1)
}
```

## UNSAFE-10 (SHOULD) — Performance: profile before optimizing

Idiomatic Rust is usually fast. Before reviewing for performance:
1. Has the author run `cargo build --release`? (Debug builds are 10–100× slower.)
2. Has the author profiled? Use `cargo flamegraph`, `perf`, `samply`, or `pprof`.
3. Are they fighting a benchmark microoptimization, or a real bottleneck?

Common honest wins (no `unsafe` needed):
- `String::with_capacity` / `Vec::with_capacity` when size is known.
- `&str` keys for `HashMap` lookups (avoid `String` allocation per lookup with `HashMap::raw_entry` or borrowing keys).
- Replace `Vec::contains` with `HashSet` in hot inner loops.
- Use `Box<[T]>` instead of `Vec<T>` for read-only owned arrays (saves capacity field).
- `SmallVec` / `tinyvec` for "usually small" arrays.

## UNSAFE-11 (MAY) — Don't shrink-wrap allocations away from clarity

A `Cow<'a, [u8]>` to avoid one allocation in a non-hot path adds complexity at every use site. Only optimize what shows up in profiles.

## UNSAFE-12 (SHOULD) — Atomic ordering: be precise

`Ordering::Relaxed` doesn't synchronize. `Ordering::SeqCst` is correct but expensive. Most ad-hoc lock-free code wants `Acquire`/`Release` pairs. If the author wrote `SeqCst` everywhere without thought, that's a documentation/correctness flag. If they wrote `Relaxed` for a "happens-before" relationship, that's a bug.

```rust
// Producer-consumer pattern: Release on store, Acquire on load.
flag.store(true, Ordering::Release);
// ... in another thread ...
if flag.load(Ordering::Acquire) { /* sees writes that happened before .store */ }
```

Read [Rust Atomics and Locks](https://marabos.nl/atomics/) before reviewing concurrent unsafe.

## UNSAFE-13 (MUST) — `Send` / `Sync` manual impls require justification

`unsafe impl Send for X` and `unsafe impl Sync for X` are `unsafe` for a reason. Every such impl needs a `// SAFETY:` comment explaining why the type is thread-safe.

```rust
// SAFETY: `Inner` is never aliased except through this struct's API,
// which serializes access via the mutex in `Outer`.
unsafe impl Send for Outer {}
unsafe impl Sync for Outer {}
```

## UNSAFE-14 (SHOULD) — Run `cargo miri test` for unsafe code

`miri` catches UB the compiler can't. Any crate with `unsafe` blocks should have miri in CI.
