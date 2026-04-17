# Les Itérateurs en Rust

## Avantages

**Performance**
- Les itérateurs sont des *zero-cost abstractions* — le compilateur les optimise aussi bien (souvent mieux) qu'une boucle `for` manuelle
- Évitent les copies inutiles en travaillant avec des références

**Lisibilité et expressivité**
```rust
// Sans itérateur
let mut total = 0;
for i in 0..nums.len() {
    if nums[i] % 2 == 0 {
        total += nums[i] * 2;
    }
}

// Avec itérateur
let total: i32 = nums.iter()
    .filter(|&&x| x % 2 == 0)
    .map(|&x| x * 2)
    .sum();
```

**Sécurité**
- Pas d'accès par index → pas de risque d'out-of-bounds
- Le compilateur garantit que l'itérateur reste valide pendant son utilisation

---

## Method Chaining

Chaque méthode d'itérateur retourne un nouvel itérateur, ce qui permet d'enchaîner :

```rust
let nums = vec![1, 2, 3, 4, 5, 6, 7, 8];

let result: Vec<String> = nums.iter()
    .filter(|&&x| x % 2 == 0)   // garde les pairs → [2, 4, 6, 8]
    .map(|&x| x * 10)            // multiplie par 10 → [20, 40, 60, 80]
    .take(3)                      // garde les 3 premiers → [20, 40, 60]
    .map(|x| format!("{}!", x))  // transforme en String → ["20!", "40!", "60!"]
    .collect();                   // consomme l'itérateur → Vec
```

---

## Lazy Evaluation

Les itérateurs sont **lazy** — aucun calcul n'a lieu avant une méthode consommatrice :

```rust
// Seuls les 3 premiers éléments sont traités, malgré le range de 1 million
let result: Vec<_> = (0..1_000_000)
    .filter(|x| x % 2 == 0)
    .take(3)
    .collect();
```

---

## Méthodes consommatrices

Ces méthodes terminent la chaîne et déclenchent l'exécution :

| Méthode       | Retourne        |
|---------------|-----------------|
| `.collect()`  | `Vec`, `HashMap`, `String`... |
| `.sum()`      | total           |
| `.count()`    | nombre d'éléments |
| `.any()`      | `bool`          |
| `.find()`     | `Option<T>`     |
| `.for_each()` | `()`            |

Sans méthode consommatrice, **rien ne s'exécute** — l'itérateur reste en attente.
