
# Coffee Pricing — Same Two Designs in Go and Python

This mirrors the Rust discussion with the same two approaches:

1. **Classic Decorator Pattern**  
2. **Data-oriented / practical model**

---

# Go

## 1) Decorator Pattern in Go

```go
package main

import "fmt"

type Beverage interface {
	Description() string
	Cost() float64
}

type BlackCoffee struct{}

func (b BlackCoffee) Description() string { return "Black coffee" }
func (b BlackCoffee) Cost() float64       { return 2.00 }

type Milk struct {
	beverage Beverage
}

func (m Milk) Description() string { return m.beverage.Description() + ", milk" }
func (m Milk) Cost() float64       { return m.beverage.Cost() + 0.40 }

type Sugar struct {
	beverage Beverage
}

func (s Sugar) Description() string { return s.beverage.Description() + ", sugar" }
func (s Sugar) Cost() float64       { return s.beverage.Cost() + 0.20 }

type Cream struct {
	beverage Beverage
}

func (c Cream) Description() string { return c.beverage.Description() + ", cream" }
func (c Cream) Cost() float64       { return c.beverage.Cost() + 0.50 }

type EspressoShot struct {
	beverage Beverage
}

func (e EspressoShot) Description() string {
	return e.beverage.Description() + ", extra espresso shot"
}
func (e EspressoShot) Cost() float64 { return e.beverage.Cost() + 1.20 }

func main() {
	var drink Beverage = BlackCoffee{}
	drink = Milk{beverage: drink}
	drink = Sugar{beverage: drink}
	drink = Cream{beverage: drink}

	fmt.Printf("%s -> $%.2f\n", drink.Description(), drink.Cost())
}
```

### Notes
- This is a good textbook example.
- Go handles this pattern naturally with interfaces.
- Still, for a pricing domain, this is usually not the simplest design.

---

## 2) Practical Data Model in Go

```go
package main

import (
	"fmt"
	"strings"
)

type BaseDrink int

const (
	BlackCoffee BaseDrink = iota
	Espresso
)

func (b BaseDrink) Label() string {
	switch b {
	case BlackCoffee:
		return "Black coffee"
	case Espresso:
		return "Espresso"
	default:
		return "Unknown"
	}
}

func (b BaseDrink) PriceCents() int {
	switch b {
	case BlackCoffee:
		return 200
	case Espresso:
		return 250
	default:
		return 0
	}
}

type AddOn int

const (
	Milk AddOn = iota
	Cream
	Sugar
	EspressoShot
)

func (a AddOn) Label() string {
	switch a {
	case Milk:
		return "milk"
	case Cream:
		return "cream"
	case Sugar:
		return "sugar"
	case EspressoShot:
		return "extra espresso shot"
	default:
		return "unknown"
	}
}

func (a AddOn) PriceCents() int {
	switch a {
	case Milk:
		return 40
	case Cream:
		return 50
	case Sugar:
		return 20
	case EspressoShot:
		return 120
	default:
		return 0
	}
}

type Drink struct {
	Base   BaseDrink
	AddOns []AddOn
}

func NewDrink(base BaseDrink) Drink {
	return Drink{Base: base, AddOns: []AddOn{}}
}

func (d Drink) With(addon AddOn) Drink {
	d.AddOns = append(d.AddOns, addon)
	return d
}

func (d Drink) PriceCents() int {
	total := d.Base.PriceCents()
	for _, addon := range d.AddOns {
		total += addon.PriceCents()
	}
	return total
}

func (d Drink) Description() string {
	if len(d.AddOns) == 0 {
		return d.Base.Label()
	}

	parts := make([]string, 0, len(d.AddOns))
	for _, addon := range d.AddOns {
		parts = append(parts, addon.Label())
	}
	return d.Base.Label() + ", " + strings.Join(parts, ", ")
}

func Latte() Drink {
	return NewDrink(Espresso).With(Milk)
}

func main() {
	drink := NewDrink(BlackCoffee).With(Milk).With(Sugar)
	fmt.Printf("%s -> $%.2f\n", drink.Description(), float64(drink.PriceCents())/100.0)

	latte := Latte()
	fmt.Printf("%s -> $%.2f\n", latte.Description(), float64(latte.PriceCents())/100.0)
}
```

### Notes
- This is what I would actually prefer in Go too.
- Easier to serialize.
- Easier to validate.
- Better fit for APIs and storage.

---

# Python

## 1) Decorator Pattern in Python

```python
from __future__ import annotations
from abc import ABC, abstractmethod


class Beverage(ABC):
    @abstractmethod
    def description(self) -> str:
        pass

    @abstractmethod
    def cost(self) -> float:
        pass


class BlackCoffee(Beverage):
    def description(self) -> str:
        return "Black coffee"

    def cost(self) -> float:
        return 2.00


class BeverageDecorator(Beverage):
    def __init__(self, beverage: Beverage) -> None:
        self.beverage = beverage


class Milk(BeverageDecorator):
    def description(self) -> str:
        return f"{self.beverage.description()}, milk"

    def cost(self) -> float:
        return self.beverage.cost() + 0.40


class Sugar(BeverageDecorator):
    def description(self) -> str:
        return f"{self.beverage.description()}, sugar"

    def cost(self) -> float:
        return self.beverage.cost() + 0.20


class Cream(BeverageDecorator):
    def description(self) -> str:
        return f"{self.beverage.description()}, cream"

    def cost(self) -> float:
        return self.beverage.cost() + 0.50


class EspressoShot(BeverageDecorator):
    def description(self) -> str:
        return f"{self.beverage.description()}, extra espresso shot"

    def cost(self) -> float:
        return self.beverage.cost() + 1.20


drink = Cream(Sugar(Milk(BlackCoffee())))
print(f"{drink.description()} -> ${drink.cost():.2f}")
```

### Notes
- Python makes the classic pattern very easy to express.
- Good for teaching OOP composition.
- But still not the best real-world model for a pricing domain.

---

## 2) Practical Data Model in Python

```python
from dataclasses import dataclass, field
from enum import Enum


class BaseDrink(Enum):
    BLACK_COFFEE = ("Black coffee", 200)
    ESPRESSO = ("Espresso", 250)

    @property
    def label(self) -> str:
        return self.value[0]

    @property
    def price_cents(self) -> int:
        return self.value[1]


class AddOn(Enum):
    MILK = ("milk", 40)
    CREAM = ("cream", 50)
    SUGAR = ("sugar", 20)
    ESPRESSO_SHOT = ("extra espresso shot", 120)

    @property
    def label(self) -> str:
        return self.value[0]

    @property
    def price_cents(self) -> int:
        return self.value[1]


@dataclass(frozen=True)
class Drink:
    base: BaseDrink
    addons: list[AddOn] = field(default_factory=list)

    def with_addon(self, addon: AddOn) -> "Drink":
        return Drink(base=self.base, addons=[*self.addons, addon])

    def price_cents(self) -> int:
        return self.base.price_cents + sum(addon.price_cents for addon in self.addons)

    def description(self) -> str:
        if not self.addons:
            return self.base.label
        return f"{self.base.label}, " + ", ".join(addon.label for addon in self.addons)

    @staticmethod
    def black_coffee() -> "Drink":
        return Drink(BaseDrink.BLACK_COFFEE)

    @staticmethod
    def espresso() -> "Drink":
        return Drink(BaseDrink.ESPRESSO)

    @staticmethod
    def latte() -> "Drink":
        return Drink.espresso().with_addon(AddOn.MILK)


drink = Drink.black_coffee().with_addon(AddOn.MILK).with_addon(AddOn.SUGAR)
print(f"{drink.description()} -> ${drink.price_cents() / 100:.2f}")

latte = Drink.latte()
print(f"{latte.description()} -> ${latte.price_cents() / 100:.2f}")
```

### Notes
- This is the cleaner design for APIs, persistence, and validation.
- In Python, this is often the version that scales better in business code.

---

# Bottom line

## Decorator pattern
Use it to learn:
- composition
- wrapping behavior
- classic OOP patterns

## Data-oriented model
Use it in real systems when:
- the problem is mostly state + pricing rules
- you need JSON / DB persistence
- you want simpler validation and testing

---

# Honest comparison

- **Rust**: data model is clearly the better fit
- **Go**: data model is usually better too
- **Python**: both are easy, but data model often wins in business apps

The decorator version is still worth learning because it teaches an important design idea:
**add behavior by composition instead of inheritance**
