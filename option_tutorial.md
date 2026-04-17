# Option en Rust

## C'est quoi ?

`Option<T>` représente une valeur qui peut être **présente ou absente**, sans utiliser `null`.

```rust
enum Option<T> {
    Some(T),  // il y a une valeur
    None,     // pas de valeur
}
```

---

## Cas d'utilisation typiques

```rust
fn diviser(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 {
        None
    } else {
        Some(a / b)
    }
}

let result = diviser(10.0, 2.0); // Some(5.0)
let erreur = diviser(10.0, 0.0); // None
```

---

## Extraire la valeur

### `match` (façon explicite)
```rust
match diviser(10.0, 2.0) {
    Some(val) => println!("Résultat : {}", val),
    None      => println!("Division impossible"),
}
```

### `if let` (quand on ignore le cas None)
```rust
if let Some(val) = diviser(10.0, 2.0) {
    println!("Résultat : {}", val);
}
```

### `unwrap()` (dangereux — panic si None)
```rust
let val = diviser(10.0, 2.0).unwrap(); // 5.0
let val = diviser(10.0, 0.0).unwrap(); // PANIC!
```

### `unwrap_or()` (valeur par défaut)
```rust
let val = diviser(10.0, 0.0).unwrap_or(0.0); // 0.0
```

### `unwrap_or_else()` (valeur par défaut calculée)
```rust
let val = diviser(10.0, 0.0).unwrap_or_else(|| {
    println!("Calcul de la valeur par défaut...");
    -1.0
}); // -1.0
```

---

## Méthodes utiles

### `.map()` — transformer la valeur si présente
```rust
let result = diviser(10.0, 2.0)
    .map(|x| x * 100.0); // Some(500.0)

let result = diviser(10.0, 0.0)
    .map(|x| x * 100.0); // None — pas de panic
```

### `.and_then()` — chaîner des fonctions qui retournent Option
```rust
fn racine(x: f64) -> Option<f64> {
    if x < 0.0 { None } else { Some(x.sqrt()) }
}

let result = diviser(16.0, 4.0)
    .and_then(|x| racine(x)); // Some(2.0)

let result = diviser(16.0, 0.0)
    .and_then(|x| racine(x)); // None
```

### `.is_some()` / `.is_none()` — vérifier sans extraire
```rust
let opt: Option<i32> = Some(42);
opt.is_some(); // true
opt.is_none(); // false
```

### `.filter()` — annuler la valeur si condition non respectée
```rust
let result = Some(42)
    .filter(|&x| x > 50); // None

let result = Some(42)
    .filter(|&x| x > 10); // Some(42)
```

---

## Option dans les collections

```rust
let v = vec![1, 2, 3];

let premier = v.first();       // Option<&i32> → Some(&1)
let get = v.get(10);           // Option<&i32> → None (index hors limites)
let trouve = v.iter().find(|&&x| x == 2); // Option<&i32> → Some(&2)
```

---

## L'opérateur `?` (propagation)

Dans une fonction qui retourne `Option`, `?` retourne `None` automatiquement si la valeur est absente :

```rust
fn traitement(a: f64, b: f64) -> Option<f64> {
    let d = diviser(a, b)?;   // retourne None si division impossible
    let r = racine(d)?;       // retourne None si valeur négative
    Some(r * 2.0)
}
```

---

## Option vs null

| Langage      | Valeur absente | Sécurité         |
|--------------|----------------|------------------|
| Java/JS/C    | `null`         | Crash à l'exécution |
| Rust         | `Option<T>`    | Vérification forcée à la compilation |

Rust **force** à gérer le cas `None` — impossible d'oublier de vérifier.
