# Déréférencement en Rust

---

## C'est quoi ?

Déréférencer (`*`) signifie **accéder à la valeur pointée** par une référence ou un pointeur.

```rust
let x = 5;
let r = &x;      // r est une référence vers x

println!("{}", r);  // 5 — déréférencement automatique
println!("{}", *r); // 5 — déréférencement explicite

*r == x // true
```

---

## Avec les références classiques

```rust
let mut x = 10;
let r = &mut x;

*r += 5; // modifie la valeur via la référence

println!("{}", x); // 15
```

### Comparaison avec déréférencement
```rust
let x = 5;
let y = 5;
let r = &x;

// ERREUR — compare une référence avec un entier
// r == y

// OK — compare les valeurs
*r == y  // true
```

---

## Avec `Box<T>`

`Box<T>` implémente `Deref`, donc `*` fonctionne comme avec une référence.

```rust
let x = 5;
let b = Box::new(x);

println!("{}", *b); // 5

// Équivalent interne :
// *(b.deref())
```

---

## Le trait `Deref`

Le déréférencement avec `*` appelle en réalité la méthode `deref()` du trait `Deref`.

```rust
pub trait Deref {
    type Target;
    fn deref(&self) -> &Self::Target;
}
```

### Implémenter `Deref` sur un type custom

```rust
struct MonBox<T>(T);

impl<T> MonBox<T> {
    fn new(x: T) -> MonBox<T> {
        MonBox(x)
    }
}

impl<T> std::ops::Deref for MonBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}

fn main() {
    let x = 5;
    let b = MonBox::new(x);

    println!("{}", *b); // 5 — appelle b.deref() puis *
}
```

### Ce que fait `*b` en interne
```rust
*b
// est traduit par le compilateur en :
*(b.deref())
```

---

## Deref Coercion — déréférencement automatique en chaîne

Quand on passe une valeur à une fonction, Rust applique automatiquement `deref()` autant de fois que nécessaire pour correspondre au type attendu.

```rust
fn afficher(s: &str) {
    println!("{}", s);
}

fn main() {
    let s = String::from("bonjour");
    let b = Box::new(s);

    afficher(&b); // Box<String> → &String → &str — automatique !
}
```

### Chaîne de coercions
```
&Box<String>
    → deref() → &String
        → deref() → &str      ✓ correspond au paramètre
```

### Sans deref coercion — ce qu'on devrait écrire
```rust
afficher(&(*(*b))); // explicite et verbeux — Rust évite ça
afficher(&b);       // ce qu'on écrit vraiment
```

---

## `DerefMut` — déréférencement mutable

Comme `Deref` mais pour la mutabilité :

```rust
use std::ops::DerefMut;

let mut b = Box::new(5);
*b += 10;             // appelle DerefMut::deref_mut()
println!("{}", *b);   // 15
```

### Règles de coercion mutable
Rust applique la deref coercion mutable seulement si :
- `T: DerefMut<Target=U>` → `&mut T` devient `&mut U`
- `T: Deref<Target=U>` → `&mut T` peut devenir `&U` (mut → immuable autorisé)
- `&T` ne devient jamais `&mut U` (immuable → mutable interdit)

---

## Exemples de deref coercion dans la pratique

```rust
// String → &str
fn longueur(s: &str) -> usize { s.len() }
let s = String::from("hello");
longueur(&s); // &String coercé en &str automatiquement

// Vec<T> → &[T]
fn somme(s: &[i32]) -> i32 { s.iter().sum() }
let v = vec![1, 2, 3];
somme(&v); // &Vec<i32> coercé en &[i32] automatiquement

// Box<String> → &str
let b = Box::new(String::from("hello"));
longueur(&b); // &Box<String> → &String → &str automatiquement
```

---

## Résumé

| Opération | Ce qui se passe |
|---|---|
| `*r` sur `&T` | Accède directement à la valeur `T` |
| `*b` sur `Box<T>` | Appelle `b.deref()` puis `*` |
| `&b` passé à `fn(&str)` | Deref coercion automatique en chaîne |
| `*b += 1` sur `Box<i32>` | Appelle `DerefMut::deref_mut()` |

```rust
// Les trois formes équivalentes
let b = Box::new(String::from("hello"));

b.len()          // déréférencement automatique via Deref
(*b).len()       // déréférencement explicite
b.deref().len()  // appel direct au trait
```
