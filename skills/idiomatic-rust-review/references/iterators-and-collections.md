# Iterators & Collections

Sources: [Effectively Using Iterators In Rust](https://hermanradtke.com/2015/06/22/effectively-using-iterators-in-rust.html/), [Rust Iterators Beyond the Basics](https://blog.jetbrains.com/rust/2024/03/12/rust-iterators-beyond-the-basics-part-i-building-blocks/), [Iteration patterns for Result & Option](http://xion.io/post/code/rust-iter-patterns.html), [Russian Dolls and clean Rust code](https://web.archive.org/web/20220126183049/https://blog.mgattozzi.dev/russian-dolls/), [Rayon: data parallelism in Rust](https://smallcultfollowing.com/babysteps/blog/2015/12/18/rayon-data-parallelism-in-rust/).

---

## ITER-01 (SHOULD) — Prefer iterator chains over index loops

```rust
// BAD
let mut out = Vec::new();
for i in 0..items.len() {
    if items[i].active { out.push(items[i].name.clone()); }
}

// GOOD
let out: Vec<String> = items.iter()
    .filter(|i| i.active)
    .map(|i| i.name.clone())
    .collect();
```

The iterator form is allocation-aware (`collect` can size-hint), composes with `rayon` for parallelism, and reads top-to-bottom as a pipeline.

Counter-rule: when the body has multiple early returns, side-effects, or interleaved state, a `for` loop is clearer. Don't force a `.fold()` for the sake of it.

## ITER-02 (MUST) — `collect::<Result<Vec<_>, _>>()` to short-circuit on first error

```rust
// BAD — partial results, error swallowed
let parsed: Vec<i32> = strs.iter().map(|s| s.parse().unwrap()).collect();

// GOOD — first error propagates
let parsed: Vec<i32> = strs.iter().map(|s| s.parse()).collect::<Result<_, _>>()?;
```

Same trick works for `Option`: `iter.map(...).collect::<Option<Vec<_>>>()`.

## ITER-03 (SHOULD) — `flat_map` instead of `map(...).flatten()`

```rust
// BAD
words.iter().map(|w| w.chars()).flatten()

// GOOD
words.iter().flat_map(|w| w.chars())
```

Same number of allocations, one fewer combinator, signals intent.

## ITER-04 (SHOULD) — Use `iter()` / `iter_mut()` / `into_iter()` deliberately

These have different semantics; pick the one you need:

- `iter()` → `&T` — caller keeps the collection.
- `iter_mut()` → `&mut T` — caller keeps the collection, mutates in place.
- `into_iter()` → `T` — consumes the collection.

```rust
// BAD — into_iter consumes when we only needed to read
for item in &vec { … }            // borrows
for item in vec.into_iter() { … } // consumes, vec is gone
```

`for x in vec` calls `into_iter()` implicitly. `for x in &vec` calls `iter()`. Know the difference.

## ITER-05 (SHOULD) — Don't `collect()` then iterate again

Every `collect()` allocates. If you only need to iterate, stay lazy.

```rust
// BAD
let v: Vec<_> = items.iter().filter(|i| i.active).collect();
for item in v { use_it(item); }

// GOOD
for item in items.iter().filter(|i| i.active) { use_it(item); }
```

The collection is justified when you'll iterate multiple times, sort, index, or send across a thread boundary.

## ITER-06 (MAY) — Prefer `.sum()` / `.product()` / `.min()` / `.max()` over manual fold

```rust
// BAD
let total = nums.iter().fold(0, |a, b| a + b);

// GOOD
let total: i64 = nums.iter().sum();
```

Reach for `fold` when the accumulator isn't a primitive numeric reduction.

## ITER-07 (SHOULD) — `partition` for splitting into two collections

```rust
// BAD
let mut passed = Vec::new();
let mut failed = Vec::new();
for r in results { if r.ok { passed.push(r); } else { failed.push(r); } }

// GOOD
let (passed, failed): (Vec<_>, Vec<_>) = results.into_iter().partition(|r| r.ok);
```

## ITER-08 (SHOULD) — `try_fold` / `try_for_each` for fallible reductions

```rust
// IDIOMATIC
let total = nums.iter().try_fold(0i64, |acc, &n| acc.checked_add(n).ok_or(Overflow))?;
```

## ITER-09 (MUST) — Don't index into iterators with `.nth()` in loops

`.nth(i)` consumes `i+1` items every call. In a loop you've made the iteration O(n²).

```rust
// BAD
for i in 0..n {
    let x = my_iter.clone().nth(i).unwrap();
    use_it(x);
}

// GOOD
for x in my_iter { use_it(x); }
```

If you genuinely need indexed access, collect to a `Vec` once.

## ITER-10 (SHOULD) — `Vec::with_capacity` when the size is known

```rust
// OK
let mut v = Vec::new();
for x in 0..1000 { v.push(transform(x)); }

// BETTER
let mut v = Vec::with_capacity(1000);
for x in 0..1000 { v.push(transform(x)); }

// BEST
let v: Vec<_> = (0..1000).map(transform).collect();  // collect uses size_hint
```

## ITER-11 (MAY) — `HashMap::entry` for "insert-or-update"

```rust
// BAD
if let Some(v) = map.get_mut(&k) {
    v.push(item);
} else {
    map.insert(k, vec![item]);
}

// GOOD
map.entry(k).or_default().push(item);
```

## ITER-12 (SHOULD) — Match container choice to access pattern

| Need | Container |
| --- | --- |
| Ordered, indexed, push/pop at end | `Vec<T>` |
| Push/pop at both ends | `VecDeque<T>` |
| Stable iteration, dedup | `BTreeSet<T>` |
| Fast lookup, no order | `HashSet<T>` |
| Insertion-ordered map | `IndexMap` (crate) |
| LRU cache | `lru` (crate) |
| Sparse integer keys | `BTreeMap<usize, T>` or `slotmap` |

Flag uses of `Vec::contains` in hot paths — that's an O(n) linear scan; a `HashSet` is O(1).

## ITER-13 (SHOULD) — Use `windows` / `chunks` / `chunks_exact` over manual indexing

```rust
// BAD
for i in 1..v.len() {
    let prev = v[i - 1];
    let cur = v[i];
    process_pair(prev, cur);
}

// GOOD
for pair in v.windows(2) {
    process_pair(pair[0], pair[1]);
}
```

## ITER-14 (MAY) — `zip` + `enumerate` order: `.iter().enumerate()` reads better than `.enumerate().iter()`

`enumerate()` adds `(usize, T)` tuples. Place it where it composes naturally:

```rust
for (i, (a, b)) in xs.iter().zip(ys.iter()).enumerate() { … }
```

## ITER-15 (MAY) — `rayon`'s `par_iter` for embarrassingly parallel work

When the work per item is non-trivial (file parse, CPU-bound transform), changing `.iter()` to `.par_iter()` often gives a free speedup. Don't suggest it for IO-bound or tiny-work loops where the overhead dominates.
