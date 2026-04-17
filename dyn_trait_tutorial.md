# dyn Trait en Rust — sans et avec Box

---

## `dyn Trait` seul — impossible

`dyn Trait` est un type de taille inconnue à la compilation (DST — Dynamically Sized Type).
Il doit toujours vivre derrière une référence ou un pointeur.

```rust
// ERREUR — taille inconnue à la compilation
let animal: dyn Animal = Chien;

// OK — derrière une référence ou pointeur
let animal: &dyn Animal = &Chien;
let animal: Box<dyn Animal> = Box::new(Chien);
```

---

## `&dyn Trait` — la plus courante

Pas d'allocation heap, pas d'ownership transféré — simple emprunt.

```rust
fn faire_son(animal: &dyn Animal) {
    println!("{}", animal.son());
}

let chien = Chien;
faire_son(&chien); // pas de Box, pas d'allocation heap
```

---

## `&mut dyn Trait`

```rust
fn renommer(animal: &mut dyn Animal) {
    // modification possible
}
```

---

## `Rc<dyn Trait>` — ownership partagé (single-thread)

```rust
use std::rc::Rc;

let animal: Rc<dyn Animal> = Rc::new(Chien);
let copie = Rc::clone(&animal); // deux propriétaires
```

---

## `Arc<dyn Trait>` — ownership partagé (multi-thread)

```rust
use std::sync::Arc;

let animal: Arc<dyn Animal> = Arc::new(Chien);
// peut être partagé entre threads
```

---

## Quand choisir quoi ?

| | Allocation | Ownership | Thread-safe |
|---|---|---|---|
| `&dyn Trait` | Aucune | Emprunt | Selon le type |
| `Box<dyn Trait>` | Heap | Unique | Non |
| `Rc<dyn Trait>` | Heap | Partagé | Non |
| `Arc<dyn Trait>` | Heap | Partagé | Oui |

---

## Règle générale

- **`&dyn Trait`** → paramètres de fonctions — plus simple, pas d'allocation
- **`Box<dyn Trait>`** → posséder la valeur (struct, retour de fonction, collection)
- **`Rc<dyn Trait>`** → plusieurs propriétaires dans un même thread
- **`Arc<dyn Trait>`** → plusieurs propriétaires entre threads
