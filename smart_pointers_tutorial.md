# Smart Pointers en Rust

Les smart pointers sont des structures qui se comportent comme des pointeurs, mais avec des capacités supplémentaires : gestion de l'ownership, comptage de références, accès thread-safe, etc.

---

## `Box<T>` — Pointeur heap, ownership unique

Alloue une valeur sur le **heap**. Le plus simple des smart pointers.

### Quand utiliser
- Type de taille inconnue à la compilation (DST, récursif)
- Transférer l'ownership d'une grande donnée sans la copier
- `Box<dyn Trait>` pour le polymorphisme dynamique

```rust
// Valeur sur le heap
let b = Box::new(5);
println!("{}", b); // 5 — déréférencement automatique

// Type récursif
enum Liste {
    Noeud(i32, Box<Liste>),
    Vide,
}

let liste = Liste::Noeud(1,
    Box::new(Liste::Noeud(2,
        Box::new(Liste::Vide))));
```

### Mémoire
```
Stack        Heap
┌───────┐    ┌───────┐
│  ptr ─┼───►│   5   │
└───────┘    └───────┘
```

---

## `Rc<T>` — Reference Counted (single-thread)

Permet **plusieurs propriétaires** d'une même valeur. La valeur est libérée quand le dernier `Rc` est droppé.

```rust
use std::rc::Rc;

let a = Rc::new(String::from("bonjour"));
let b = Rc::clone(&a); // incrémente le compteur, pas de copie
let c = Rc::clone(&a);

println!("Compteur : {}", Rc::strong_count(&a)); // 3
println!("{}", a); // "bonjour"

drop(b);
println!("Compteur : {}", Rc::strong_count(&a)); // 2
// valeur libérée quand compteur atteint 0
```

### Limitation
- **Immutable seulement** — pas de `&mut` avec `Rc`
- **Single-thread** — non `Send`, ne peut pas traverser les threads

```
Stack         Heap
┌───┐         ┌──────────────────┐
│ a ┼────────►│ count: 3         │
└───┘         │ valeur: "bonjour"│
┌───┐         └──────────────────┘
│ b ┼────────►       ▲
└───┘                │
┌───┐                │
│ c ┼────────────────┘
└───┘
```

---

## `RefCell<T>` — Borrow checking à l'exécution

Permet la **mutabilité intérieure** : muter une valeur même derrière une référence immutable.
Le borrow checking se fait à l'exécution (panic si règles violées) plutôt qu'à la compilation.

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// Lecture
let lecture = data.borrow();
println!("{:?}", *lecture); // [1, 2, 3]
drop(lecture); // libérer avant d'emprunter en mutable

// Écriture
data.borrow_mut().push(4);
println!("{:?}", data.borrow()); // [1, 2, 3, 4]
```

### Panic à l'exécution si règles violées
```rust
let data = RefCell::new(5);
let _a = data.borrow();
let _b = data.borrow_mut(); // PANIC — déjà emprunté en immutable
```

---

## `Rc<RefCell<T>>` — Le combo courant

`Rc` pour plusieurs propriétaires + `RefCell` pour la mutabilité :

```rust
use std::rc::Rc;
use std::cell::RefCell;

let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared);
let b = Rc::clone(&shared);

a.borrow_mut().push(4); // mutation via a
b.borrow_mut().push(5); // mutation via b

println!("{:?}", shared.borrow()); // [1, 2, 3, 4, 5]
```

---

## `Arc<T>` — Atomic Reference Counted (multi-thread)

Comme `Rc<T>` mais **thread-safe** grâce aux opérations atomiques.

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(vec![1, 2, 3]);

let data_clone = Arc::clone(&data);
let handle = thread::spawn(move || {
    println!("Thread : {:?}", data_clone);
});

handle.join().unwrap();
println!("Main : {:?}", data);
```

---

## `Arc<Mutex<T>>` — Le combo multi-thread

`Arc` pour partager + `Mutex` pour la mutabilité thread-safe :

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let compteur = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..5 {
    let c = Arc::clone(&compteur);
    handles.push(thread::spawn(move || {
        let mut val = c.lock().unwrap();
        *val += 1;
    }));
}

for h in handles { h.join().unwrap(); }
println!("Compteur : {}", *compteur.lock().unwrap()); // 5
```

---

## `Cow<T>` — Clone on Write

Évite de cloner une valeur si elle n'est pas modifiée.

```rust
use std::borrow::Cow;

fn majuscules(s: &str) -> Cow<str> {
    if s.chars().all(|c| c.is_uppercase()) {
        Cow::Borrowed(s)    // pas de copie — déjà en majuscules
    } else {
        Cow::Owned(s.to_uppercase()) // copie seulement si nécessaire
    }
}

let a = majuscules("BONJOUR"); // Borrowed — pas d'allocation
let b = majuscules("bonjour"); // Owned — allocation
```

---

## Le trait `Deref` — pourquoi ils "agissent comme des pointeurs"

Les smart pointers implémentent `Deref`, ce qui permet la **déréférencement automatique** :

```rust
let b = Box::new(String::from("hello"));

// Ces trois lignes sont équivalentes grâce à Deref
println!("{}", *b);        // déréférencement explicite
println!("{}", b.len());   // déréférencement automatique
println!("{}", b);         // coercion automatique
```

### Chaîne de déréférencement automatique
```rust
let b: Box<String> = Box::new(String::from("hello"));
// Box<String> → String → str
let _s: &str = &b; // fonctionne automatiquement
```

---

## Le trait `Drop` — nettoyage automatique

Tous les smart pointers implémentent `Drop` pour libérer la mémoire :

```rust
struct MonPointeur {
    data: String,
}

impl Drop for MonPointeur {
    fn drop(&mut self) {
        println!("Libération de : {}", self.data);
    }
}

{
    let p = MonPointeur { data: String::from("test") };
    // ... utilisation
} // drop() appelé automatiquement ici
```

---

## Résumé comparatif

| Smart Pointer | Ownership | Mutabilité | Thread-safe | Usage typique |
|---|---|---|---|---|
| `Box<T>` | Unique | Normale | Non | Heap, récursion, `dyn Trait` |
| `Rc<T>` | Partagé | Immutable | Non | Graphes, arbres partagés |
| `RefCell<T>` | Unique | Intérieure | Non | Mutabilité derrière `&` |
| `Rc<RefCell<T>>` | Partagé | Intérieure | Non | Combo courant single-thread |
| `Arc<T>` | Partagé | Immutable | Oui | Partage entre threads |
| `Arc<Mutex<T>>` | Partagé | Gardée | Oui | Mutation entre threads |
| `Cow<T>` | Selon cas | Selon cas | Non | Éviter les copies inutiles |
