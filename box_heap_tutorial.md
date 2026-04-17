# Box<T> — Pointer vers des données sur le Heap

---

## Stack vs Heap

Avant de comprendre `Box`, il faut comprendre la différence entre stack et heap.

```
Stack                          Heap
┌──────────────────┐           ┌──────────────────────────┐
│ Taille fixe      │           │ Taille dynamique          │
│ Allocation rapide│           │ Allocation plus lente     │
│ LIFO             │           │ Accès via pointeur        │
│ Libéré auto      │           │ Libéré via Drop/ownership │
│                  │           │                           │
│ let x: i32 = 5;  │           │ String, Vec, Box...       │
└──────────────────┘           └──────────────────────────┘
```

Par défaut, Rust alloue tout sur le **stack**. `Box<T>` force l'allocation sur le **heap**.

---

## Syntaxe de base

```rust
fn main() {
    // Sans Box — valeur sur le stack
    let x = 5;

    // Avec Box — valeur sur le heap, pointeur sur le stack
    let b = Box::new(5);

    println!("x = {}", x);
    println!("b = {}", b); // déréférencement automatique via Deref
    println!("*b = {}", *b); // déréférencement explicite
}
```

### Ce qui se passe en mémoire
```
Stack              Heap
┌────────────┐     ┌────────────┐
│ b (ptr)  ──┼────►│     5      │
└────────────┘     └────────────┘

Quand b sort du scope :
- le pointeur sur le stack est libéré
- la valeur sur le heap est libérée (via Drop)
```

---

## Cas d'utilisation 1 — Types récursifs

Sans `Box`, un type récursif a une taille infinie — Rust ne peut pas le compiler.

```rust
// ERREUR — taille infinie, Rust ne peut pas allouer
enum Liste {
    Noeud(i32, Liste),
    Vide,
}

// OK — Box a une taille fixe (pointeur)
enum Liste {
    Noeud(i32, Box<Liste>),
    Vide,
}
```

### Pourquoi ça fonctionne
```
Sans Box :
Noeud(i32, Noeud(i32, Noeud(i32, ...))) → taille infinie

Avec Box :
Noeud
├── i32       (4 bytes)
└── Box<Liste> (8 bytes — juste un pointeur)
               └── (sur le heap) Noeud suivant...
```

### Construire une liste chaînée
```rust
enum Liste {
    Noeud(i32, Box<Liste>),
    Vide,
}

use Liste::{Noeud, Vide};

fn main() {
    let liste = Noeud(1,
        Box::new(Noeud(2,
            Box::new(Noeud(3,
                Box::new(Vide))))));

    // Parcourir la liste
    let mut courant = &liste;
    loop {
        match courant {
            Noeud(val, suivant) => {
                println!("{}", val);
                courant = suivant;
            }
            Vide => break,
        }
    }
}
// 1
// 2
// 3
```

---

## Cas d'utilisation 2 — Déplacer l'ownership sans copier

Pour les grandes structures, `Box` permet de transférer l'ownership via un pointeur (8 bytes) plutôt que de copier toute la donnée.

```rust
struct GrandTableau {
    data: [f64; 10_000], // 80 000 bytes sur le stack
}

// Sans Box — copie 80 000 bytes à chaque transfert
fn traiter(t: GrandTableau) { /* ... */ }

// Avec Box — déplace seulement 8 bytes (le pointeur)
fn traiter(t: Box<GrandTableau>) { /* ... */ }

let grand = Box::new(GrandTableau { data: [0.0; 10_000] });
traiter(grand); // déplace le pointeur, pas les données
```

---

## Cas d'utilisation 3 — `Box<dyn Trait>` (polymorphisme)

`Box` permet de posséder un trait object — un type dont on ne connaît pas la taille exacte.

```rust
trait Dessinable {
    fn dessiner(&self);
}

struct Cercle { rayon: f64 }
struct Rectangle { largeur: f64, hauteur: f64 }

impl Dessinable for Cercle {
    fn dessiner(&self) { println!("Cercle r={}", self.rayon); }
}

impl Dessinable for Rectangle {
    fn dessiner(&self) { println!("Rect {}x{}", self.largeur, self.hauteur); }
}

fn main() {
    // Vec de types différents — impossible sans Box
    let formes: Vec<Box<dyn Dessinable>> = vec![
        Box::new(Cercle { rayon: 3.0 }),
        Box::new(Rectangle { largeur: 4.0, hauteur: 5.0 }),
        Box::new(Cercle { rayon: 1.5 }),
    ];

    for forme in &formes {
        forme.dessiner();
    }
}
```

---

## Le trait `Deref` — Box se comporte comme un `&`

`Box<T>` implémente `Deref<Target = T>`, ce qui permet d'utiliser `*` pour accéder à la valeur, et la déréférencement automatique dans les appels de méthode.

```rust
fn main() {
    let b = Box::new(String::from("bonjour"));

    // Déréférencement explicite
    println!("{}", *b);

    // Déréférencement automatique — appelle String::len()
    println!("{}", b.len()); // pas besoin de (*b).len()

    // Coercion automatique Box<String> → &str
    afficher(&b);
}

fn afficher(s: &str) {
    println!("{}", s);
}
```

### Chaîne de déréférencement
```
Box<String> → *Box<String> = String → *String = str
&Box<String> est automatiquement coercé en &str
```

---

## Le trait `Drop` — libération automatique

Quand un `Box` sort du scope, `Drop` est appelé automatiquement — libère la mémoire heap.

```rust
struct Valeur(i32);

impl Drop for Valeur {
    fn drop(&mut self) {
        println!("Libération de {}", self.0);
    }
}

fn main() {
    let a = Box::new(Valeur(1));
    let b = Box::new(Valeur(2));
    println!("Avant la fin du scope");
} // b droppé, puis a droppé (ordre inverse de déclaration)

// Avant la fin du scope
// Libération de 2
// Libération de 1
```

### Forcer un drop anticipé
```rust
let b = Box::new(Valeur(42));
drop(b); // libère immédiatement
// b ne peut plus être utilisé ici
```

---

## Box n'a pas d'overhead de performance

`Box<T>` est une **zero-cost abstraction** :
- Pas de comptage de références (contrairement à `Rc`)
- Pas de locks (contrairement à `Mutex`)
- Une seule allocation heap + un pointeur sur le stack
- Le compilateur peut même optimiser certains `Box` pour éviter l'allocation

---

## Résumé

| Situation | Sans Box | Avec Box |
|---|---|---|
| Type récursif | Erreur de compilation | Taille fixe via pointeur |
| Grande structure | Copie coûteuse | Déplace 8 bytes |
| `dyn Trait` | Impossible à posséder | `Box<dyn Trait>` |
| Mémoire | Stack | Heap |
| Ownership | Unique | Unique |
| Thread-safe | N/A | Non (utiliser `Arc`) |

```rust
// Les trois usages en résumé
let b1: Box<i32>            = Box::new(42);              // heap basique
let b2: Box<dyn Trait>      = Box::new(MaStruct);        // polymorphisme
let b3: Box<ListeRecursive> = Box::new(ListeRecursive {}); // récursion
```
