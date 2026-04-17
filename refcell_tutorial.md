# RefCell<T> et le Pattern Interior Mutability

---

## C'est quoi l'Interior Mutability ?

En Rust, la règle normale est :
- `&T` → référence immutable — lecture seule
- `&mut T` → référence mutable — lecture/écriture, mais **une seule à la fois**

L'**interior mutability** permet de **muter des données même derrière une référence immutable `&T`**.

Le borrow checking est déplacé de la **compilation** vers l'**exécution**.

```
Règles normales    → vérifiées à la COMPILATION → erreur de compilation
RefCell<T>        → vérifiées à l'EXÉCUTION    → panic si violées
```

---

## Rappel — règles du borrow checker

```rust
// OK — plusieurs références immutables
let x = 5;
let r1 = &x;
let r2 = &x; // OK

// OK — une seule référence mutable
let mut y = 5;
let r = &mut y; // OK

// ERREUR — impossible d'avoir & et &mut en même temps
let mut z = 5;
let r1 = &z;
let r2 = &mut z; // ERREUR à la compilation
```

`RefCell<T>` applique ces mêmes règles, mais **à l'exécution**.

---

## Syntaxe de base

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);

    // Emprunt immutable — comme &T
    {
        let lecture = data.borrow();
        println!("{:?}", *lecture); // [1, 2, 3]
    } // lecture droppée ici

    // Emprunt mutable — comme &mut T
    {
        let mut ecriture = data.borrow_mut();
        ecriture.push(4);
    } // ecriture droppée ici

    println!("{:?}", data.borrow()); // [1, 2, 3, 4]
}
```

| Méthode | Équivalent | Comportement si règle violée |
|---|---|---|
| `.borrow()` | `&T` | panic |
| `.borrow_mut()` | `&mut T` | panic |
| `.try_borrow()` | `&T` | retourne `Err` |
| `.try_borrow_mut()` | `&mut T` | retourne `Err` |

---

## Panic à l'exécution si règles violées

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);

    let _a = data.borrow();     // OK — emprunt immutable
    let _b = data.borrow();     // OK — plusieurs immutables autorisés
    let _c = data.borrow_mut(); // PANIC — déjà emprunté en immutable !
}
// thread 'main' panicked at 'already borrowed: BorrowMutError'
```

### Version sans panic — `try_borrow_mut`
```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);
    let _a = data.borrow();

    match data.try_borrow_mut() {
        Ok(mut val) => *val += 1,
        Err(e)      => println!("Impossible d'emprunter : {}", e),
    }
}
```

---

## Cas d'utilisation — Mock dans les tests

Le cas classique : implémenter un trait qui requiert `&self` (immutable), mais on a besoin de modifier un état interne.

```rust
pub trait Messager {
    fn envoyer(&self, msg: &str); // &self — pas &mut self
}

// En production
struct EmailClient;
impl Messager for EmailClient {
    fn envoyer(&self, msg: &str) {
        println!("Email envoyé : {}", msg);
    }
}

// Dans les tests — on veut capturer les messages
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessager {
        messages: RefCell<Vec<String>>, // RefCell permet la mutation via &self
    }

    impl MockMessager {
        fn new() -> Self {
            MockMessager { messages: RefCell::new(vec![]) }
        }
    }

    impl Messager for MockMessager {
        fn envoyer(&self, msg: &str) {
            // &self est immutable, mais RefCell permet la mutation intérieure
            self.messages.borrow_mut().push(String::from(msg));
        }
    }

    #[test]
    fn test_envoi() {
        let mock = MockMessager::new();
        mock.envoyer("bonjour");
        mock.envoyer("au revoir");

        assert_eq!(mock.messages.borrow().len(), 2);
    }
}
```

---

## `Rc<RefCell<T>>` — Le combo ownership partagé + mutabilité

`Rc<T>` seul = plusieurs propriétaires, mais immutable.
`Rc<RefCell<T>>` = plusieurs propriétaires **et** mutable.

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

    let a = Rc::clone(&shared);
    let b = Rc::clone(&shared);

    // a et b peuvent tous les deux muter la valeur
    a.borrow_mut().push(4);
    b.borrow_mut().push(5);

    println!("{:?}", shared.borrow()); // [1, 2, 3, 4, 5]
    println!("Compteur Rc : {}", Rc::strong_count(&shared)); // 3
}
```

### Cas d'utilisation — graphe avec liens bidirectionnels
```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Noeud {
    valeur: i32,
    enfants: RefCell<Vec<Rc<Noeud>>>,
}

impl Noeud {
    fn new(val: i32) -> Rc<Self> {
        Rc::new(Noeud {
            valeur: val,
            enfants: RefCell::new(vec![]),
        })
    }

    fn ajouter_enfant(&self, enfant: Rc<Noeud>) {
        self.enfants.borrow_mut().push(enfant);
    }
}

fn main() {
    let racine = Noeud::new(1);
    let enfant1 = Noeud::new(2);
    let enfant2 = Noeud::new(3);

    racine.ajouter_enfant(Rc::clone(&enfant1));
    racine.ajouter_enfant(Rc::clone(&enfant2));

    println!("Racine : {}", racine.valeur);
    for enfant in racine.enfants.borrow().iter() {
        println!("  Enfant : {}", enfant.valeur);
    }
}
```

---

## `Cell<T>` — Alternative légère pour les types `Copy`

`Cell<T>` est similaire à `RefCell<T>` mais pour les types qui implémentent `Copy`. Pas de références — seulement des copies.

```rust
use std::cell::Cell;

fn main() {
    let c = Cell::new(5);

    println!("{}", c.get()); // 5 — copie la valeur

    c.set(10); // pas besoin de borrow_mut
    println!("{}", c.get()); // 10
}
```

| | `Cell<T>` | `RefCell<T>` |
|---|---|---|
| Types supportés | `Copy` uniquement | Tous |
| Accès | `.get()` / `.set()` | `.borrow()` / `.borrow_mut()` |
| Overhead | Minimal | Léger (compteur de borrows) |
| Références | Non (copie) | Oui |

---

## Résumé — quand utiliser quoi ?

| Situation | Solution |
|---|---|
| Mutation normale | `let mut x` |
| Mutation derrière `&self` (single-thread) | `RefCell<T>` |
| Plusieurs propriétaires + mutation (single-thread) | `Rc<RefCell<T>>` |
| Mutation thread-safe | `Mutex<T>` |
| Plusieurs propriétaires + mutation (multi-thread) | `Arc<Mutex<T>>` |
| Type `Copy`, mutation derrière `&self` | `Cell<T>` |

```
Compile time checks (rapide, sûr)
        ↑
    &mut T  ←── préférer quand possible
        ↓
Runtime checks (flexible, risque de panic)
    RefCell<T>  ←── quand le borrow checker est trop restrictif
```

> Utiliser `RefCell<T>` avec parcimonie — si possible, restructurer le code pour utiliser `&mut T`.
