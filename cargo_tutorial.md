# Cargo : Workspaces, Install et Commandes Personnalisées

---

## Cargo Workspaces

Un workspace permet de gérer **plusieurs crates liées** dans un seul dépôt, avec un seul `Cargo.lock` partagé.

### Structure typique
```
mon_projet/
├── Cargo.toml          ← manifest du workspace
├── Cargo.lock          ← partagé entre tous les membres
├── app/
│   ├── Cargo.toml
│   └── src/main.rs
├── lib_core/
│   ├── Cargo.toml
│   └── src/lib.rs
└── lib_utils/
    ├── Cargo.toml
    └── src/lib.rs
```

### `Cargo.toml` du workspace (racine)
```toml
[workspace]
members = [
    "app",
    "lib_core",
    "lib_utils",
]
```

### `Cargo.toml` d'un membre
```toml
[package]
name = "app"
version = "0.1.0"
edition = "2021"

[dependencies]
lib_core = { path = "../lib_core" }
lib_utils = { path = "../lib_utils" }
```

### Commandes workspace

```bash
# Compiler tous les membres
cargo build

# Compiler un seul membre
cargo build -p lib_core

# Tester tous les membres
cargo test

# Tester un seul membre
cargo test -p app

# Lancer un binaire spécifique
cargo run -p app
```

### Avantages
- Un seul `Cargo.lock` → versions cohérentes entre crates
- Compilation partagée → pas de recompilation des dépendances communes
- `cargo test` / `cargo build` opèrent sur tout le workspace
- Idéal pour monorepos (lib + CLI + serveur dans le même repo)

---

## Cargo Install

`cargo install` compile et installe un binaire Rust dans `~/.cargo/bin/`.

### Installer depuis crates.io
```bash
cargo install ripgrep      # installe rg
cargo install bat          # installe bat (cat amélioré)
cargo install cargo-watch  # installe cargo-watch
```

### Installer une version spécifique
```bash
cargo install ripgrep --version 13.0.0
```

### Installer depuis un dépôt Git
```bash
cargo install --git https://github.com/user/repo
cargo install --git https://github.com/user/repo --branch dev
cargo install --git https://github.com/user/repo --tag v1.2.0
```

### Installer depuis un chemin local
```bash
cargo install --path ./mon_outil
```

### Lister les binaires installés
```bash
cargo install --list
```

### Désinstaller
```bash
cargo uninstall ripgrep
```

### Forcer la réinstallation
```bash
cargo install ripgrep --force
```

### Où sont installés les binaires ?
```
~/.cargo/bin/   ← ajouté au PATH par rustup automatiquement
```

---

## Cargo Custom Commands (sous-commandes)

Cargo supporte des **sous-commandes personnalisées** : tout binaire nommé `cargo-xxx` dans le PATH devient accessible via `cargo xxx`.

### Utiliser une sous-commande existante

```bash
# Installer
cargo install cargo-watch
cargo install cargo-expand
cargo install cargo-audit
cargo install cargo-outdated
cargo install cargo-edit

# Utiliser
cargo watch -x run           # relance à chaque modification
cargo expand                 # affiche le code après expansion des macros
cargo audit                  # vérifie les vulnérabilités des dépendances
cargo outdated               # liste les dépendances obsolètes
cargo add serde              # ajoute une dépendance au Cargo.toml
cargo rm serde               # supprime une dépendance
cargo upgrade                # met à jour les dépendances
```

### Créer sa propre sous-commande

Toute application nommée `cargo-mon_outil` devient `cargo mon_outil`.

**1. Créer le projet**
```bash
cargo new cargo-bonjour
cd cargo-bonjour
```

**2. `src/main.rs`**
```rust
fn main() {
    // Cargo passe le nom de la sous-commande comme premier argument
    // args() → ["cargo-bonjour", "bonjour", ...args_utilisateur]
    let args: Vec<String> = std::env::args().skip(2).collect();

    if args.is_empty() {
        println!("Bonjour, monde !");
    } else {
        println!("Bonjour, {} !", args.join(" "));
    }
}
```

**3. Installer**
```bash
cargo install --path .
```

**4. Utiliser**
```bash
cargo bonjour
# Bonjour, monde !

cargo bonjour Alice Bob
# Bonjour, Alice Bob !
```

### Sous-commandes populaires à connaître

| Commande | Description |
|---|---|
| `cargo watch` | Relance automatiquement à chaque changement |
| `cargo expand` | Affiche le code après expansion des macros |
| `cargo audit` | Détecte les vulnérabilités dans les dépendances |
| `cargo outdated` | Liste les dépendances avec des mises à jour disponibles |
| `cargo add` | Ajoute une dépendance (inclus dans Cargo 1.62+) |
| `cargo tree` | Affiche l'arbre des dépendances (inclus dans Cargo) |
| `cargo clippy` | Linter avancé (inclus dans rustup) |
| `cargo fmt` | Formateur de code (inclus dans rustup) |
| `cargo doc` | Génère la documentation HTML |
| `cargo bench` | Lance les benchmarks |

---

## Résumé

| Fonctionnalité | Commande clé | Usage |
|---|---|---|
| Workspace | `cargo build -p nom` | Monorepo multi-crates |
| Install | `cargo install nom` | Installer des outils Rust |
| Uninstall | `cargo uninstall nom` | Supprimer un binaire installé |
| Custom command | `cargo-xxx` dans PATH | Étendre Cargo avec ses propres outils |
