# ☕ Coffee Pricing in Rust --- Two Design Approaches

## 1️⃣ Decorator Pattern (OOP Style)

``` rust
trait Beverage {
    fn description(&self) -> String;
    fn cost(&self) -> f64;
}

struct BlackCoffee;

impl Beverage for BlackCoffee {
    fn description(&self) -> String {
        "Black coffee".to_string()
    }

    fn cost(&self) -> f64 {
        2.0
    }
}

struct Milk {
    beverage: Box<dyn Beverage>,
}

impl Milk {
    fn new(beverage: Box<dyn Beverage>) -> Self {
        Self { beverage }
    }
}

impl Beverage for Milk {
    fn description(&self) -> String {
        format!("{}, milk", self.beverage.description())
    }

    fn cost(&self) -> f64 {
        self.beverage.cost() + 0.4
    }
}

struct Sugar {
    beverage: Box<dyn Beverage>,
}

impl Sugar {
    fn new(beverage: Box<dyn Beverage>) -> Self {
        Self { beverage }
    }
}

impl Beverage for Sugar {
    fn description(&self) -> String {
        format!("{}, sugar", self.beverage.description())
    }

    fn cost(&self) -> f64 {
        self.beverage.cost() + 0.2
    }
}
```

## 2️⃣ Rust-Idiomatic Design (Recommended)

``` rust
#[derive(Debug, Clone, Copy)]
enum BaseDrink {
    BlackCoffee,
    Espresso,
}

impl BaseDrink {
    fn price(self) -> u32 {
        match self {
            BaseDrink::BlackCoffee => 200,
            BaseDrink::Espresso => 250,
        }
    }

    fn label(self) -> &'static str {
        match self {
            BaseDrink::BlackCoffee => "Black coffee",
            BaseDrink::Espresso => "Espresso",
        }
    }
}

#[derive(Debug, Clone, Copy)]
enum AddOn {
    Milk,
    Cream,
    Sugar,
    EspressoShot,
}

impl AddOn {
    fn price(self) -> u32 {
        match self {
            AddOn::Milk => 40,
            AddOn::Cream => 50,
            AddOn::Sugar => 20,
            AddOn::EspressoShot => 120,
        }
    }

    fn label(self) -> &'static str {
        match self {
            AddOn::Milk => "milk",
            AddOn::Cream => "cream",
            AddOn::Sugar => "sugar",
            AddOn::EspressoShot => "extra espresso shot",
        }
    }
}

struct Drink {
    base: BaseDrink,
    addons: Vec<AddOn>,
}

impl Drink {
    fn new(base: BaseDrink) -> Self {
        Self {
            base,
            addons: Vec::new(),
        }
    }

    fn black_coffee() -> Self {
        Self::new(BaseDrink::BlackCoffee)
    }

    fn espresso() -> Self {
        Self::new(BaseDrink::Espresso)
    }

    fn with(mut self, addon: AddOn) -> Self {
        self.addons.push(addon);
        self
    }

    fn milk(self) -> Self {
        self.with(AddOn::Milk)
    }

    fn sugar(self) -> Self {
        self.with(AddOn::Sugar)
    }

    fn cream(self) -> Self {
        self.with(AddOn::Cream)
    }

    fn espresso_shot(self) -> Self {
        self.with(AddOn::EspressoShot)
    }

    fn price(&self) -> u32 {
        self.base.price() + self.addons.iter().map(|a| a.price()).sum::<u32>()
    }

    fn description(&self) -> String {
        if self.addons.is_empty() {
            self.base.label().to_string()
        } else {
            let addons = self
                .addons
                .iter()
                .map(|a| a.label())
                .collect::<Vec<_>>()
                .join(", ");
            format!("{}, {}", self.base.label(), addons)
        }
    }

    fn latte() -> Self {
        Drink::espresso().milk()
    }
}
```

## 🔥 Key Takeaway

-   Decorator = good for learning patterns
-   Data model = best for real Rust systems
