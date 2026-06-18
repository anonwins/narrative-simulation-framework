# 21. Sim: Economy — NSF Specification

> **Framework:** Currency, debt pressure, vendors, pawn, financial survival mechanics.
> **Content pack:** Prices, vendor inventories, economic narrative pressure content.
> Terminology: [Glossary](../terminology-glossary.md)

This system models **financial survival**.

In most RPGs:

```text
Economy = Gold
```

In a NSF-style game:

```text
Economy = Pressure
```

Money is not primarily used to buy power.

Money is used to:

* avoid disaster
* maintain dignity
* solve problems
* access opportunities
* survive another day

The player should frequently feel:

```text
I need money.
```

not:

```text
I need a better sword.
```

---

# Core Principle

The economy should create tension.

The system should constantly ask:

```text
Can the player afford to continue living?
```

rather than:

```text
Can the player buy stronger gear?
```

---

# Core Architecture

```text
EconomyService
 ├─ Currency
 ├─ DebtLedger
 ├─ RentLedger
 ├─ Vendors
 ├─ PawnShop
 ├─ FinancialEvents
 ├─ SurvivalExpenses
 ├─ Rewards
 └─ EconomicHistory
```

---

# Core Model

```csharp
class EconomyState
{
    int Money;

    int Debt;

    List<Expense> PendingExpenses;

    List<FinancialObligation> Obligations;
}
```

The economy should track more than cash.

---

# Currency

The player possesses liquid money.

Money is used for:

* food
* lodging
* transportation
* services
* information
* tools
* books
* clothing
* bribes (if supported)

Money should feel scarce.

---

# Financial Pressure

The economy should generate ongoing pressure.

Sources:

```text
Rent
Debt
Interest
Obligations
Story beat costs
Optional Purchases
```

Without pressure, money becomes meaningless.

---

# Rent System

One of the most important economic mechanics.

The player must periodically pay for lodging.

Example:

```text
Day End
↓
Rent Due
↓
Player Pays
OR
Consequences Trigger
```

This creates urgency.

---

# Rent Object

```csharp
class RentRequirement
{
    int Amount;
    TimeSpan DueTime;
    bool Paid;
}
```

---

# Rent Outcomes

Possible outcomes:

## Paid

```text
Safe lodging
Story continues
```

---

## Late

```text
Warning
Actor reaction
Reduced trust
```

---

## Unpaid

```text
Alternative sleeping arrangements
Story beat consequences
Narrative fallout
```

Failure should create story, not game over.

---

# Debt System

Debt represents future obligations.

```csharp
class Debt
{
    int Amount;
    string CreditorId;
    DateTime DueDate;
}
```

Debt can be:

* explicit
* social
* institutional

---

# Debt Pressure

Debt should affect:

* dialogue
* stress
* available choices
* reputation
* story beat branch options

The player should occasionally be forced to think economically.

---

# Debt Collection Events

Examples:

```text
Reminder
Threat
Negotiation
Extension
Consequence
```

Debt should generate narrative.

---

# Financial Survival Loop

Recommended loop:

```text
Explore
↓
Find opportunities
↓
Earn money
↓
Pay obligations
↓
Avoid collapse
↓
Repeat
```

This loop should sit beneath the investigation.

---

# Vendors

Vendors are not loot sinks.

They are:

```text
Economic actors
```

Each vendor should have:

```csharp
class Vendor
{
    Inventory Inventory;
    PricingModel Prices;
    Schedule Schedule;
}
```

---

# Vendor Categories

Examples:

```text
Bookstore
Pawn Shop
Food Vendor
Clothing Vendor
Tool Vendor
Black Market Vendor
```

Different categories create different economic decisions.

---

# Vendor Inventory

Inventory should support:

```text
Static Stock
Story beat stock
Conditional Stock
Time-Based Stock
```

This makes the economy feel alive.

---

# Price System

Items should have:

```csharp
BasePrice
BuyPrice
SellPrice
```

Usually:

```text
Buy Price > Sell Price
```

The player should lose value when liquidating possessions.

---

# Pawn Shop System

A key NSF-style mechanic.

The pawn shop converts possessions into survival money.

---

# Pawn Shop Loop

```text
Find Item
↓
Sell Item
↓
Gain Cash
↓
Pay Expense
```

The player often sacrifices future utility for present survival.

---

# Pawnable Objects

Support categories:

```text
Valuables
Books
Clothing
Tools
Story-beat-neutral items
Collectibles
```

Some items should be unsellable.

---

# Economic Tradeoffs

The economy should frequently create decisions:

```text
Buy useful item
OR
Save rent money
```

```text
Keep collectible
OR
Pawn collectible
```

```text
Purchase information
OR
Purchase food
```

These are meaningful choices.

---

# Information Market

Important for inquiry-based games.

Money may purchase:

* tips
* documents
* records
* access
* introductions

Facts become an economic resource.

---

# Service Economy

Actors may charge for:

```text
Transportation
Expertise
Access
Repair
Medical Help
```

This broadens economic gameplay.

---

# Story beat rewards

Story beat rewards should often be money.

However:

```text
Money should never become infinite.
```

Otherwise pressure collapses.

---

# Economic Scaling

Avoid exponential growth.

NSF-style economies work best when:

```text
Income grows slowly
Expenses remain relevant
```

The player should never become overwhelmingly rich.

---

# Windfall Events

Rare large payouts can exist.

Examples:

```text
Section bonus
Rare Item Sale
Unexpected Discovery
```

These should feel significant.

---

# Poverty State

The engine should support being broke.

Example:

```text
Money = 0
```

Consequences:

* limited options
* altered dialogue
* new opportunities
* desperation choices

Being poor should remain playable.

---

# Financial Reputation

Optional subsystem.

Tracks how others view the player's finances.

Examples:

```text
Reliable
Broke
In Debt
Generous
Cheap
```

Can influence dialogue.

---

# Economic Opportunities

Money sources should include:

```text
Story beat rewards
Scavenging
Item Sales
Services
Tips
Side Jobs
Negotiation Success
```

Multiple paths reduce frustration.

---

# Bottle / Recycling Loop

A NSF-style economy benefits from low-value scavenging.

Example:

```text
Find Bottle
↓
Recycle
↓
Small Income
```

This provides:

* emergency income
* world interaction
* survival flavor

---

# Economy and Time

Many economic systems should interact with time.

Examples:

```text
Rent due at night
Vendor closes at 20:00
Debt matures in 3 days
```

Time pressure creates economic pressure.

---

# Economy and Relationships

Relationships can affect finances.

Examples:

```text
Discount
Loan
Gift
Refusal of Service
```

Social systems and economy should interact.

---

# Economy and Politics

Ideological identity may influence economics.

Examples:

```text
faction_guild contact helps financially
Business owner offers opportunity
```

These should be narrative consequences rather than raw bonuses.

---

# Economy and Belief

Beliefs may modify:

```text
Prices
Income Sources
Financial Opportunities
Debt Outcomes
```

This creates systemic interaction.

---

# Event Integration

Economy should emit:

```text
MoneyChanged
DebtAdded
DebtPaid
RentPaid
RentMissed
ItemSold
ItemPurchased
VendorUnlocked
```

Other systems subscribe to these events.

---

# Save Data Requirements

Serialize:

```text
Current Money
Debt
Obligations
Vendor State
Shop Inventories
Purchase History
Economic Flags
```

---

# Minimal Engine Interfaces

```csharp
interface IEconomyService
{
    int Money { get; }

    bool CanAfford(int amount);

    void AddMoney(int amount);

    void RemoveMoney(int amount);
}

interface IVendor
{
    int GetBuyPrice(Item item);

    int GetSellPrice(Item item);
}
```

---

# Recommended Internal Loop

```text
Interaction
      ↓
Acquire Resources
      ↓
Economic Decision
      ↓
Financial Pressure
      ↓
Narrative Consequence
      ↓
New Opportunity
```

---

# Most Important Insight

The Economy is not about wealth accumulation.

It is about maintaining enough resources to continue pursuing the investigation.

The engine should constantly create situations where the player asks:

```text
Can I afford this?
What am I willing to sacrifice?
What happens if I don't pay?
```

If implemented correctly, money becomes a narrative force.

If implemented incorrectly, it becomes a meaningless number that only exists to buy items.
