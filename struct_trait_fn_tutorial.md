# Structs, Traits et Fonctions en Rust

---

## Fonctions

### Déclaration de base
```rust
fn saluer(nom: &str) -> String {
    format!("Bonjour, {}!", nom)  // pas de ; → valeur de retour implicite
}

fn additionner(a: i32, b: i32) -> i32 {
    a + b
}
```

### Retour multiple avec tuple
```rust
fn min_max(v: &[i32]) -> (i32, i32) {
    let min = *v.iter().min().unwrap();
    let max = *v.iter().max().unwrap();
    (min, max)
}

let (min, max) = min_max(&[3, 1, 4, 1, 5]);
```

### Paramètres génériques
```rust
fn premier<T>(v: &[T]) -> Option<&T> {
    v.first()
}
```

---

## Structs

### Struct de base
```rust
struct Utilisateur {
    nom: String,
    age: u32,
    actif: bool,
}

let user = Utilisateur {
    nom: String::from("Alice"),
    age: 30,
    actif: true,
};

println!("{} a {} ans", user.nom, user.age);
```

### Struct avec `impl` — méthodes associées
```rust
impl Utilisateur {
    // Constructeur (méthode associée — pas de self)
    fn new(nom: &str, age: u32) -> Self {
        Utilisateur {
            nom: String::from(nom),
            age,
            actif: true,
        }
    }

    // Méthode d'instance — prend &self
    fn se_presenter(&self) -> String {
        format!("Je m'appelle {} et j'ai {} ans.", self.nom, self.age)
    }

    // Méthode qui modifie — prend &mut self
    fn desactiver(&mut self) {
        self.actif = false;
    }
}

let mut user = Utilisateur::new("Bob", 25);
println!("{}", user.se_presenter());
user.desactiver();
```

### Tuple Struct
```rust
struct Point(f64, f64);
struct Couleur(u8, u8, u8);

let p = Point(1.0, 2.5);
let rouge = Couleur(255, 0, 0);

println!("x={}, y={}", p.0, p.1);
```

### Dériver des traits automatiquement
```rust
#[derive(Debug, Clone, PartialEq)]
struct Rectangle {
    largeur: f64,
    hauteur: f64,
}

let r = Rectangle { largeur: 10.0, hauteur: 5.0 };
println!("{:?}", r);       // Debug
let r2 = r.clone();        // Clone
println!("{}", r == r2);   // PartialEq → true
```

---

## Traits

Un trait définit un **comportement partagé** entre plusieurs types — similaire aux interfaces dans d'autres langages.

### Définir un trait
```rust
trait Aire {
    fn aire(&self) -> f64;

    // Méthode par défaut (optionnelle à redéfinir)
    fn description(&self) -> String {
        format!("Aire : {:.2}", self.aire())
    }
}
```

### Implémenter un trait
```rust
struct Rectangle {
    largeur: f64,
    hauteur: f64,
}

struct Cercle {
    rayon: f64,
}

impl Aire for Rectangle {
    fn aire(&self) -> f64 {
        self.largeur * self.hauteur
    }
}

impl Aire for Cercle {
    fn aire(&self) -> f64 {
        std::f64::consts::PI * self.rayon * self.rayon
    }
}

let r = Rectangle { largeur: 4.0, hauteur: 3.0 };
let c = Cercle { rayon: 2.0 };

println!("{}", r.description()); // "Aire : 12.00"
println!("{}", c.description()); // "Aire : 12.57"
```

### Trait comme paramètre de fonction
```rust
// Syntaxe impl Trait (simple)
fn afficher_aire(forme: &impl Aire) {
    println!("Aire = {}", forme.aire());
}

// Syntaxe générique (plus flexible)
fn afficher_aire<T: Aire>(forme: &T) {
    println!("Aire = {}", forme.aire());
}

afficher_aire(&r);
afficher_aire(&c);
```

### Plusieurs traits requis
```rust
use std::fmt::Display;

fn afficher<T: Aire + Display>(forme: &T) {
    println!("{} a une aire de {}", forme, forme.aire());
}
```

### Trait objects — polymorphisme dynamique
```rust
fn plus_grande_aire(formes: &[Box<dyn Aire>]) -> f64 {
    formes.iter()
        .map(|f| f.aire())
        .fold(f64::NEG_INFINITY, f64::max)
}

let formes: Vec<Box<dyn Aire>> = vec![
    Box::new(Rectangle { largeur: 4.0, hauteur: 3.0 }),
    Box::new(Cercle { rayon: 2.0 }),
];

println!("Plus grande aire : {}", plus_grande_aire(&formes));
```

---

## Tout ensemble

```rust
trait Affichable {
    fn afficher(&self);
}

#[derive(Debug)]
struct Produit {
    nom: String,
    prix: f64,
}

impl Produit {
    fn new(nom: &str, prix: f64) -> Self {
        Produit { nom: String::from(nom), prix }
    }

    fn avec_remise(&self, pct: f64) -> f64 {
        self.prix * (1.0 - pct / 100.0)
    }
}

impl Affichable for Produit {
    fn afficher(&self) {
        println!("{} → {:.2}€", self.nom, self.prix);
    }
}

fn afficher_tous(items: &[impl Affichable]) {
    for item in items {
        item.afficher();
    }
}

fn main() {
    let produits = vec![
        Produit::new("Clavier", 89.99),
        Produit::new("Souris", 45.00),
    ];

    afficher_tous(&produits);
    println!("Clavier avec 10% de remise : {:.2}€", produits[0].avec_remise(10.0));
}
```

---

## Résumé

| Concept       | Rôle                                              |
|---------------|---------------------------------------------------|
| `fn`          | Encapsuler de la logique réutilisable             |
| `struct`      | Regrouper des données liées                       |
| `impl`        | Ajouter des méthodes à une struct                 |
| `trait`       | Définir un comportement partagé entre types       |
| `impl Trait`  | Exiger un comportement dans une fonction          |
| `dyn Trait`   | Polymorphisme dynamique (types hétérogènes)       |
