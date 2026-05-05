# Mastering `Option<T>` in Rust

The `Option` type is one of the most fundamental and powerful features of Rust. It represents a value that can either be **something** (`Some`) or **nothing** (`None`), replacing the dangerous concept of `null` found in other languages.

## 1. What is `Option<T>`?

In Rust, `Option` is an enum defined in the standard library:

```rust
enum Option<T> {
    Some(T), // Contains a value of type T
    None,    // Contains no value
}
```

By using `Option`, Rust forces you to handle the case where a value might be missing at compile-time, preventing "Null Pointer Exceptions".

---

## 2. Basic Usage

### Creating Options
```rust
let some_number = Some(5);
let some_string = Some("a string");

let absent_number: Option<i32> = None;
```

### Common Functions returning Option
```rust
fn divide(numerator: f64, denominator: f64) -> Option<f64> {
    if denominator == 0.0 {
        None
    } else {
        Some(numerator / denominator)
    }
}

let result = divide(10.0, 2.0); // Some(5.0)
let error = divide(10.0, 0.0);  // None
```

---

## 3. Extracting the Value

### `match` (The most explicit way)
```rust
match divide(10.0, 2.0) {
    Some(val) => println!("Result: {val}"),
    None => println!("Cannot divide by zero"),
}
```

### `if let` (Syntactic sugar for matching only `Some`)
```rust
if let Some(val) = divide(10.0, 2.0) {
    println!("Result: {val}");
}
```

### `unwrap` and `expect` (Use with caution!)
- `unwrap()`: Returns the value or panics if `None`.
- `expect("message")`: Like unwrap, but with a custom error message.
```rust
let val = divide(10.0, 2.0).unwrap(); 
let val = divide(10.0, 0.0).expect("Division failed!"); // Panics with message
```

### Safe Fallbacks
```rust
// Use a default value
let val = divide(10.0, 0.0).unwrap_or(0.0); // 0.0

// Use a closure to compute a default (lazy evaluation)
let val = divide(10.0, 0.0).unwrap_or_else(|| {
    // heavy computation...
    -1.0
});

// Use the type's default value (e.g., 0 for numbers)
let val: f64 = divide(10.0, 0.0).unwrap_or_default();
```

---

## 4. Transforming and Chaining

### `.map()`
Transforms the value inside `Some`, but leaves `None` as `None`.
```rust
let maybe_number = Some(5);
let maybe_string = maybe_number.map(|n| n.to_string()); // Some("5")
```

### `.and_then()` (FlatMap)
Use this when your transformation function *also* returns an `Option`.
```rust
fn get_sq_root(x: f64) -> Option<f64> {
    if x < 0.0 { None } else { Some(x.sqrt()) }
}

let result = divide(16.0, 2.0)
    .and_then(get_sq_root); // Some(2.828...)
```

### `.filter()`
Returns `None` if the value doesn't match a predicate.
```rust
let x = Some(10);
let filtered = x.filter(|&n| n > 5); // Some(10)
let filtered2 = x.filter(|&n| n > 20); // None
```

---

## 5. Working with References

Often you don't want to move the value out of the `Option`.

- `.as_ref()`: Converts `Option<T>` to `Option<&T>`.
- `.as_mut()`: Converts `Option<T>` to `Option<&mut T>`.

```rust
let text = Some(String::from("Hello"));
let length = text.as_ref().map(|s| s.len()); // text is still owned
```

---

## 6. Converting to `Result`

You can turn an "optional value" into an "error" if it's missing.

```rust
let x = Some(42);
let res: Result<i32, &str> = x.ok_or("Value was missing"); // Ok(42)

let y: Option<i32> = None;
let res2 = y.ok_or("Error code 404"); // Err("Error code 404")
```

---

## 7. The Question Mark Operator `?`

In functions that return `Option`, you can use `?` to quickly return `None` if an operation fails.

```rust
fn get_processed_value(a: f64, b: f64) -> Option<f64> {
    let div = divide(a, b)?;      // returns None immediately if divide is None
    let root = get_sq_root(div)?; // returns None immediately if get_sq_root is None
    Some(root * 2.0)
}
```

---

## 8. Advanced Utilities

### `.take()`
Takes the value out of the `Option`, leaving `None` in its place. Useful for moving values out of structs.
```rust
let mut x = Some(10);
let y = x.take(); // y = Some(10), x = None
```

### `.replace()`
Replaces the value and returns the old one.
```rust
let mut x = Some(10);
let old = x.replace(20); // old = Some(10), x = Some(20)
```

### `.zip()`
Combines two options into one option of a tuple.
```rust
let x = Some(1);
let y = Some("hi");
let zipped = x.zip(y); // Some((1, "hi"))
```

---

## Summary Table

| Method | Behavior |
|--------|----------|
| `unwrap()` | Get value or panic |
| `unwrap_or(d)` | Get value or default `d` |
| `map(f)` | Apply `f` to value if present |
| `and_then(f)` | Chain another `Option`-returning function |
| `ok_or(e)` | Convert to `Result`, using `e` as error |
| `?` | Return `None` early if value is absent |
