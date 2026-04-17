# Concurrence et Parallélisme en Rust

Rust garantit la **thread safety** à la compilation grâce aux traits `Send` et `Sync` — les data races sont impossibles.

> "Fearless Concurrency" — le compilateur détecte les problèmes avant l'exécution.

---

## Threads

### Créer un thread
```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        println!("Thread secondaire !");
    });

    println!("Thread principal");
    handle.join().unwrap(); // attendre la fin du thread
}
```

### Passer des données avec `move`

Le mot-clé `move` transfère l'ownership des variables capturées dans le thread.

```rust
use std::thread;

fn main() {
    let message = String::from("bonjour");

    let handle = thread::spawn(move || {
        // move transfère l'ownership de `message`
        println!("{}", message);
    });

    // ERREUR ici — message a été moved dans le thread
    // println!("{}", message);

    handle.join().unwrap();
}
```

### Plusieurs threads
```rust
use std::thread;

fn main() {
    let handles: Vec<_> = (0..5).map(|i| {
        thread::spawn(move || {
            println!("Thread {}", i);
        })
    }).collect();

    for h in handles {
        h.join().unwrap();
    }
}
```

---

## Message Passing — `mpsc` Channel

**mpsc** = Multiple Producer, Single Consumer.

Les channels permettent aux threads de **communiquer en s'envoyant des valeurs** plutôt qu'en partageant de la mémoire.

> "Do not communicate by sharing memory; share memory by communicating."

### Channel de base
```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel(); // tx = transmitter, rx = receiver

    thread::spawn(move || {
        tx.send(String::from("bonjour")).unwrap();
    });

    let message = rx.recv().unwrap(); // bloquant — attend le message
    println!("{}", message); // "bonjour"
}
```

### Envoyer plusieurs messages
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let messages = vec!["un", "deux", "trois"];
        for msg in messages {
            tx.send(msg).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    // rx en tant qu'itérateur — se termine quand tx est droppé
    for recu in rx {
        println!("{}", recu);
    }
}
// un
// deux
// trois
```

### Plusieurs producteurs (Multiple Producer)
```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let tx2 = tx.clone(); // cloner le transmetteur

    thread::spawn(move || {
        tx.send("depuis thread 1").unwrap();
    });

    thread::spawn(move || {
        tx2.send("depuis thread 2").unwrap();
    });

    for _ in 0..2 {
        println!("{}", rx.recv().unwrap());
    }
}
```

### `recv` vs `try_recv`
```rust
// recv() — bloquant, attend un message
let msg = rx.recv().unwrap();

// try_recv() — non bloquant, retourne immédiatement
match rx.try_recv() {
    Ok(msg)  => println!("Reçu : {}", msg),
    Err(_)   => println!("Pas encore de message"),
}
```

---

## Shared State — `Mutex<T>`

`Mutex` (Mutual Exclusion) garantit qu'un seul thread à la fois peut accéder aux données.

### Mutex de base
```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    {
        let mut val = m.lock().unwrap(); // acquiert le lock
        *val = 10;
    } // lock libéré automatiquement ici (Drop)

    println!("{:?}", m); // Mutex { data: 10 }
}
```

### `Arc<Mutex<T>>` — partage entre threads

`Mutex` seul ne peut pas être partagé entre threads. Il faut `Arc` pour l'ownership partagé thread-safe.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let compteur = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let c = Arc::clone(&compteur);
        handles.push(thread::spawn(move || {
            let mut val = c.lock().unwrap();
            *val += 1;
        }));
    }

    for h in handles { h.join().unwrap(); }

    println!("Résultat : {}", *compteur.lock().unwrap()); // 10
}
```

### Deadlock — le piège à éviter

Un deadlock survient quand deux threads s'attendent mutuellement.

```rust
// DEADLOCK — même thread verrouille deux fois
let m = Mutex::new(5);
let _a = m.lock().unwrap();
let _b = m.lock().unwrap(); // attend _a qui ne sera jamais libéré → deadlock
```

**Bonnes pratiques :**
- Garder les locks aussi courts que possible
- Toujours acquérir les locks dans le même ordre
- Préférer les channels quand c'est possible

---

## `RwLock<T>` — Lectures multiples, écriture exclusive

Plus flexible que `Mutex` : plusieurs lecteurs simultanés, mais un seul écrivain.

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));

    // Plusieurs lecteurs en parallèle
    let mut handles = vec![];
    for i in 0..3 {
        let d = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let lecture = d.read().unwrap();
            println!("Thread {} lit : {:?}", i, *lecture);
        }));
    }

    for h in handles { h.join().unwrap(); }

    // Un seul écrivain
    data.write().unwrap().push(4);
    println!("Final : {:?}", *data.read().unwrap());
}
```

| | `Mutex` | `RwLock` |
|---|---|---|
| Lecteurs simultanés | Non (1 à la fois) | Oui |
| Écrivain simultané | Non | Non |
| Overhead | Faible | Plus élevé |

---

## `Send` et `Sync` — les traits de thread safety

Ces deux traits marqueurs sont au cœur de la sécurité concurrente de Rust.

```rust
// Send — peut être transféré entre threads (moved)
// Sync — peut être accédé depuis plusieurs threads (&T est Send)
```

| Type | Send | Sync |
|---|---|---|
| `i32`, `bool`, `String` | Oui | Oui |
| `Rc<T>` | Non | Non |
| `Arc<T>` | Oui (si T: Send+Sync) | Oui |
| `Mutex<T>` | Oui (si T: Send) | Oui |
| `RefCell<T>` | Oui (si T: Send) | Non |
| `*mut T` (raw ptr) | Non | Non |

```rust
// Le compilateur refuse au moment de la compilation
use std::rc::Rc;

let rc = Rc::new(5);
thread::spawn(move || {
    println!("{}", rc); // ERREUR — Rc n'est pas Send
});
```

---

## Channels vs Shared State — quand choisir ?

| | Channels (`mpsc`) | Shared State (`Arc<Mutex>`) |
|---|---|---|
| Modèle | Producteur / Consommateur | Accès concurrent à une ressource |
| Couplage | Faible | Plus fort |
| Deadlock | Impossible | Possible |
| Ordre des messages | Garanti | Non garanti |
| Usage typique | Pipeline de tâches | Cache partagé, compteur |

---

## Exemple complet — pipeline de traitement

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx_brut, rx_brut) = mpsc::channel::<i32>();
    let (tx_traite, rx_traite) = mpsc::channel::<i32>();

    // Thread 1 — producteur
    thread::spawn(move || {
        for i in 1..=5 {
            tx_brut.send(i).unwrap();
        }
    });

    // Thread 2 — transformation
    thread::spawn(move || {
        for val in rx_brut {
            tx_traite.send(val * val).unwrap(); // carré
        }
    });

    // Thread principal — consommateur
    for resultat in rx_traite {
        println!("{}", resultat);
    }
}
// 1
// 4
// 9
// 16
// 25
```

---

## Résumé

| Concept | Type | Usage |
|---|---|---|
| Thread | `thread::spawn` | Exécution parallèle |
| Message passing | `mpsc::channel` | Communication entre threads |
| Exclusion mutuelle | `Mutex<T>` | Accès unique à une ressource |
| Partage thread-safe | `Arc<T>` | Ownership partagé entre threads |
| Lectures multiples | `RwLock<T>` | Lire souvent, écrire rarement |
| Thread safety | `Send` + `Sync` | Garantis par le compilateur |
