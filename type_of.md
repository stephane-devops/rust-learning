# `type_of` Function in Rust

## Code

```rust
fn type_of<T>(_: &T) -> &'static str {
    type_name::<T>()
}
```

## Explanation

This function returns the type name of any value passed to it as a string.

### How it works

- `T` is a generic type parameter — the function works with any type
- `_: &T` takes a reference to a value of type `T`, but ignores it (the `_` pattern discards the binding). The reference is only needed so the compiler can infer `T` from the argument
- `type_name::<T>()` (from `std::any::type_name`) returns a `&'static str` containing the human-readable name of type `T` — e.g. `"i32"`, `"alloc::string::String"`, `"my_crate::MyStruct"`

### Example usage

```rust
use std::any::type_name;

fn type_of<T>(_: &T) -> &'static str {
    type_name::<T>()
}

fn main() {
    let x = 42u32;
    let s = String::from("hello");
    println!("{}", type_of(&x)); // "u32"
    println!("{}", type_of(&s)); // "alloc::string::String"
}
```

### Key note

`type_name` is primarily a debugging aid — the returned string format is not guaranteed to be stable across Rust versions.

---

## Other uses of `type_name`

### 1. Directly with turbofish (no value needed)

```rust
use std::any::type_name;

fn main() {
    println!("{}", type_name::<Vec<i32>>());   // "alloc::vec::Vec<i32>"
    println!("{}", type_name::<Option<f64>>()); // "core::option::Option<f64>"
    println!("{}", type_name::<(u8, bool)>());  // "(u8, bool)"
}
```

Useful when you know the type at compile time but don't have a value.

### 2. Generic function logging / tracing

```rust
use std::any::type_name;

fn process<T>(value: T) {
    println!("Processing value of type: {}", type_name::<T>());
    // ... actual processing
}

fn main() {
    process(42i32);        // "Processing value of type: i32"
    process("hello");      // "Processing value of type: &str"
    process(vec![1, 2, 3]); // "Processing value of type: alloc::vec::Vec<i32>"
}
```

### 3. Asserting types in tests

```rust
use std::any::type_name;

fn assert_type<Expected, Actual>(_: &Actual) {
    assert_eq!(
        type_name::<Expected>(),
        type_name::<Actual>(),
        "Type mismatch"
    );
}

#[test]
fn test_return_type() {
    let result = some_function();
    assert_type::<u32, _>(&result);
}
```

### 4. Dynamic dispatch debugging

```rust
use std::any::type_name;

trait Animal {
    fn speak(&self);
}

fn debug_animal<T: Animal>(a: &T) {
    println!("Animal type: {}", type_name::<T>());
    a.speak();
}
```

### 6. Generic function with trait bound (`Display`)

```rust
fn process<T: std::fmt::Display>(value: T) {
    println!("Processing value {} of type: {}", value, type_name::<T>());
    // ... actual processing
}
```

**Breakdown:**

- `<T: std::fmt::Display>` — generic type `T` constrained to types that implement `Display`
- `value: T` — takes ownership of the value (not just a reference)
- `{}` in the format string works because `Display` is guaranteed by the trait bound
- `type_name::<T>()` reveals the concrete type at runtime

**Example output:**

```rust
process(42);        // Processing value 42 of type: i32
process("hello");   // Processing value hello of type: &str
process(3.14f64);   // Processing value 3.14 of type: f64
```

Rust uses **monomorphization** — a separate specialized version of this function is compiled for each concrete type used.

---

### 5. Short name helper (strip module path)

```rust
use std::any::type_name;

fn short_type_name<T>() -> &'static str {
    type_name::<T>().split("::").last().unwrap_or("unknown")
}

fn main() {
    println!("{}", short_type_name::<String>()); // "String"
    println!("{}", short_type_name::<Vec<i32>>()); // "Vec<i32>"
}
```

---

## Deep Dive: The Mechanism Behind `<T: std::fmt::Display>`

### What is a trait bound?

A **trait bound** constrains which types are valid for a generic parameter. Without a bound, `T` can be literally any type — the compiler knows nothing about it and won't let you call any methods on it. A bound narrows that universe.

```rust
fn print_it<T>(value: T) {
    println!("{}", value); // ERROR: `T` doesn't implement `Display`
}

fn print_it<T: std::fmt::Display>(value: T) {
    println!("{}", value); // OK: `Display` is guaranteed
}
```

---

### What does `std::fmt::Display` mean?

`Display` is a trait defined in the standard library:

```rust
pub trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

Any type that implements `Display` can be formatted with `{}`. The trait bound `T: Display` is a compile-time promise: *"whoever calls this function must supply a type that has implemented `fmt`"*.

---

### Compile-time enforcement (not runtime)

The check happens entirely at **compile time**. When you write:

```rust
fn process<T: Display>(value: T) { ... }
```

And call `process(42)`, the compiler:

1. Infers `T = i32`
2. Checks: does `i32` implement `Display`? → Yes
3. Proceeds to compile

If you call `process(vec![1, 2, 3])` (Vec does not implement `Display`):

```
error[E0277]: `Vec<{integer}>` doesn't implement `std::fmt::Display`
```

No runtime panic, no overhead — it simply won't compile.

---

### Monomorphization: one generic, many concrete functions

Rust does **not** use virtual dispatch for generics (unlike Java/C# generics at runtime). Instead it uses **monomorphization**: the compiler generates a separate, fully specialized copy of the function for each concrete type used.

```rust
fn process<T: Display>(value: T) { ... }

process(42i32);    // compiler emits: process_i32(value: i32) { ... }
process("hello");  // compiler emits: process_str(value: &str) { ... }
```

**Trade-off:**

| | Monomorphization (Rust generics) | Dynamic dispatch (`dyn Trait`) |
|---|---|---|
| Performance | Zero overhead, inlined | Vtable lookup at runtime |
| Binary size | Grows per concrete type | One copy of the code |
| Flexibility | Type known at compile time | Type resolved at runtime |

---

### Multiple bounds

You can require more than one trait using `+`:

```rust
fn process<T: Display + Clone + PartialOrd>(value: T) { ... }
```

Or with a `where` clause (cleaner for complex signatures):

```rust
fn process<T>(value: T)
where
    T: Display + Clone + PartialOrd,
{
    // ...
}
```

Both are identical to the compiler — `where` is purely a readability choice.

---

### Lifetime bounds follow the same syntax

```rust
fn longest<'a, T: Display>(x: &'a str, label: T) -> &'a str {
    println!("Label: {}", label);
    x
}
```

`'a` is a lifetime parameter; `T: Display` is a type bound. They coexist in the same `<>` list.

---

### Summary

| Concept | What it means |
|---|---|
| `<T>` | Introduce a generic type parameter |
| `T: Trait` | Constrain `T` — must implement `Trait` |
| Checked at | Compile time, at each call site |
| Implemented via | Monomorphization (no vtable, zero overhead) |
| Multiple bounds | `T: TraitA + TraitB` or `where T: TraitA + TraitB` |
