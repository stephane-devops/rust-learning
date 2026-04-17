# Box<dyn Trait> en Rust

---

## C'est quoi `Box<dyn Trait>` ?

`Box<dyn Trait>` combine deux concepts :

- **`Box<T>`** — alloue une valeur sur le **heap** et en possède la propriété
- **`dyn Trait`** — représente *n'importe quel type* qui implémente le trait (dispatch dynamique)

Ensemble, ils permettent de stocker des types **différents** dans une même collection ou fonction, tant qu'ils partagent le même trait.

---

## Pourquoi en a-t-on besoin ?

En Rust, la taille de chaque type doit être connue à la compilation. Or, si on veut un `Vec` de formes différentes (`Cercle`, `Rectangle`...), leurs tailles sont différentes — impossible sans indirection.

```rust
// ERREUR — taille inconnue à la compilation
let formes: Vec<dyn Aire> = vec![...];

// OK — Box a une taille fixe (pointeur)
let formes: Vec<Box<dyn Aire>> = vec![...];
```

`Box` résout le problème : c'est un pointeur de taille fixe qui pointe vers la vraie valeur sur le heap.

---

## Exemple de base

```rust
trait Animal {
    fn son(&self) -> &str;
    fn nom(&self) -> &str;
}

struct Chien;
struct Chat;

impl Animal for Chien {
    fn son(&self) -> &str { "Woof" }
    fn nom(&self) -> &str { "Chien" }
}

impl Animal for Chat {
    fn son(&self) -> &str { "Miaou" }
    fn nom(&self) -> &str { "Chat" }
}

fn main() {
    let animaux: Vec<Box<dyn Animal>> = vec![
        Box::new(Chien),
        Box::new(Chat),
        Box::new(Chien),
    ];

    for animal in &animaux {
        println!("{} dit : {}", animal.nom(), animal.son());
    }
}
// Chien dit : Woof
// Chat dit : Miaou
// Chien dit : Woof
```

---

## `Box<dyn Trait>` vs `impl Trait`

| | `impl Trait` | `Box<dyn Trait>` |
|---|---|---|
| Résolution | Compilation (statique) | Exécution (dynamique) |
| Performance | Plus rapide (inlining possible) | Légère overhead (vtable) |
| Types hétérogènes | Non — un seul type concret | Oui — types différents |
| Allocation heap | Non | Oui |
| Usage typique | Paramètres de fonction | Collections, retours polymorphiques |

```rust
// impl Trait — un seul type concret (le compilateur choisit)
fn faire_son(animal: &impl Animal) {
    println!("{}", animal.son());
}

// Box<dyn Trait> — type inconnu jusqu'à l'exécution
fn faire_son(animal: &Box<dyn Animal>) {
    println!("{}", animal.son());
}
```

---

## Retourner un `Box<dyn Trait>`

Utile quand la fonction doit retourner des types différents selon une condition :

```rust
trait Forme {
    fn aire(&self) -> f64;
}

struct Cercle { rayon: f64 }
struct Rectangle { largeur: f64, hauteur: f64 }

impl Forme for Cercle {
    fn aire(&self) -> f64 { std::f64::consts::PI * self.rayon * self.rayon }
}

impl Forme for Rectangle {
    fn aire(&self) -> f64 { self.largeur * self.hauteur }
}

// Impossible avec impl Trait — type de retour variable
fn creer_forme(est_cercle: bool) -> Box<dyn Forme> {
    if est_cercle {
        Box::new(Cercle { rayon: 3.0 })
    } else {
        Box::new(Rectangle { largeur: 4.0, hauteur: 5.0 })
    }
}

let f = creer_forme(true);
println!("Aire : {:.2}", f.aire()); // Aire : 28.27
```

---

## Comment ça fonctionne — la vtable

Quand on utilise `dyn Trait`, Rust crée une **vtable** (table de pointeurs de fonctions) à la compilation pour chaque type concret.

```
Box<dyn Animal>
├── pointeur → valeur sur le heap (Chien ou Chat)
└── pointeur → vtable
              ├── son()  → adresse de Chien::son ou Chat::son
              └── nom()  → adresse de Chien::nom ou Chat::nom
```

À l'exécution, chaque appel de méthode passe par la vtable — c'est le coût du dispatch dynamique.

---

## `Box` pour les types récursifs

`Box` est aussi nécessaire pour les structures récursives (liste chaînée, arbre...) :

```rust
// ERREUR — taille infinie
enum Liste {
    Noeud(i32, Liste),
    Vide,
}

// OK — Box brise la récursion avec un pointeur
enum Liste {
    Noeud(i32, Box<Liste>),
    Vide,
}

let liste = Liste::Noeud(1, Box::new(Liste::Noeud(2, Box::new(Liste::Vide))));
```

---

## Résumé

| Concept | Description |
|---|---|
| `Box<T>` | Pointeur owned vers une valeur sur le heap, taille fixe |
| `dyn Trait` | Type dynamique — n'importe quel type qui implémente le trait |
| `Box<dyn Trait>` | Permet le polymorphisme avec types hétérogènes |
| vtable | Mécanisme interne pour résoudre les méthodes à l'exécution |
| Quand utiliser | Collections mixtes, retours polymorphiques, types récursifs |
