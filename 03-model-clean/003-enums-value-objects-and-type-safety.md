Strings are the most dangerous data type in programming. A status stored as `"pending"` can be misspelled as `"pendign"`, compared incorrectly, or silently accept invalid values. A price stored as a float can produce `0.1 + 0.2 = 0.30000000000000004`. A state transition buried in an `if/else` chain can allow an order to go from "delivered" back to "pending" without anyone noticing.

These are not hypothetical problems. They are the bugs that ship on Friday at 5pm, the ones that pass every test because the test uses the same misspelled string as the code. PHP 8.1 introduced backed enums, and combined with Value Objects and the State Pattern, they give you a toolkit that makes entire categories of bugs impossible.

This chapter covers three tools, each solving a different problem:

- **Enums** replace magic strings and integers with type-safe constants
- **Value Objects** replace primitive values with immutable, self-validating domain concepts
- **The State Pattern** replaces conditional transition logic with dedicated state classes

Each is a step up in complexity. Use the simplest tool that fits your problem.

## Why Type Safety Matters

Consider this code that uses strings throughout:

```php
// Strings everywhere — anything goes
$order->update(['status' => 'pending']);
$order->update(['status' => 'pneding']);  // Typo — no error, no warning
$order->update(['status' => 'banana']);   // Nonsense — still no error

if ($order->status === 'shiped') {       // Typo in comparison — always false
    // This code never runs, and you'll spend an hour debugging
}
```

Every string comparison is a chance for a typo. Every assignment is a chance for an invalid value. The database happily stores `"banana"` as a status, and your application happily reads it back. You only discover the problem when a customer calls.

Type-safe code eliminates this:

```php
$order->update(['status' => OrderStatus::Pending]);

// OrderStatus::Pneding — your IDE catches it immediately
// OrderStatus::Banana — does not exist, cannot be used

if ($order->status === OrderStatus::Shipped) {
    // Guaranteed correct — the enum case exists or the code doesn't run
}
```

The shift is from "hope the string is right" to "your IDE and static analysis guarantee it is right." Bugs move from runtime (production, customers, 3am alerts) to development time (red squiggly lines in your editor).

## Backed Enums

A backed enum is a type-safe replacement for string or integer constants:

```php
namespace App\Enums;

enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Pending',
            self::Processing => 'Processing',
            self::Shipped => 'Shipped',
            self::Delivered => 'Delivered',
            self::Cancelled => 'Cancelled',
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::Pending => 'yellow',
            self::Processing => 'blue',
            self::Shipped => 'indigo',
            self::Delivered => 'green',
            self::Cancelled => 'red',
        };
    }

    public function icon(): string
    {
        return match ($this) {
            self::Pending => 'clock',
            self::Processing => 'refresh',
            self::Shipped => 'truck',
            self::Delivered => 'check-circle',
            self::Cancelled => 'x-circle',
        };
    }
}
```

Notice how `match` expressions are exhaustive — if you add a new case to the enum and forget to handle it in `label()`, `color()`, or `icon()`, PHP throws an error. With strings and `switch` statements, the missing case silently falls through.

### Enums in Eloquent

Eloquent casts enums automatically when you use the `casts()` method:

```php
protected function casts(): array
{
    return [
        'status' => OrderStatus::class,
    ];
}

// Now $order->status is an OrderStatus enum, not a string
$order->status; // OrderStatus::Pending
$order->status->label(); // "Pending"
$order->status->color(); // "yellow"
```

Once cast, you never deal with the raw string. The enum goes in, the enum comes out. If the database contains a value that does not match any case, `from()` throws a `ValueError` — you find the problem immediately instead of propagating bad data through your application.

### Enums in Validation

Use enums in validation rules to ensure incoming data matches a valid case:

```php
use Illuminate\Validation\Rule;

public function rules(): array
{
    return [
        'status' => ['required', Rule::enum(OrderStatus::class)],
    ];
}
```

This rejects any value that is not a valid backing value of the enum. No more maintaining a separate list of allowed values that can drift out of sync.

### Enums in Blade

```blade
<span class="badge badge-{{ $order->status->color() }}">
    <x-icon :name="$order->status->icon()" />
    {{ $order->status->label() }}
</span>
```

The template has no conditional logic — no `@if ($order->status === 'pending')`. The enum carries its own display logic, so adding a new status means updating one file (the enum), not hunting through every Blade template.

### Implementing Interfaces on Enums

Enums can implement interfaces, which is useful when you need polymorphic behavior:

```php
interface HasColor
{
    public function color(): string;
}

enum Priority: string implements HasColor
{
    case Low = 'low';
    case Medium = 'medium';
    case High = 'high';
    case Critical = 'critical';

    public function color(): string
    {
        return match ($this) {
            self::Low => 'gray',
            self::Medium => 'yellow',
            self::High => 'orange',
            self::Critical => 'red',
        };
    }
}

enum OrderStatus: string implements HasColor
{
    // ...
}
```

Now any component that accepts a `HasColor` works with both `Priority` and `OrderStatus`. The interface enforces that every enum provides the method — if a future developer adds a new `HasColor` enum and forgets `color()`, PHP throws a fatal error.

## Value Objects

Enums solve the problem of "which values are valid." Value Objects solve a different problem: "what does this value *mean*, and what can I do with it?"

### The Problem: Primitives That Lie

Consider a function that calculates a discount:

```php
function applyDiscount(float $price, float $discount): float
{
    return $price - $discount;
}

// These all compile and run — but which is correct?
applyDiscount(99.99, 10.00);      // $10 off? Or 10%?
applyDiscount(10.00, 99.99);      // Oops — arguments reversed, negative price
applyDiscount(99.99, -5.00);      // Negative discount — price goes up?
```

The types are technically correct — both arguments are floats — but the types tell you nothing about what the values represent. A price in dollars, a price in cents, a discount percentage, and a discount amount are all `float`. PHP cannot tell them apart, so you cannot tell them apart at 3am when you are debugging a payment issue.

### The Solution: Value Objects

A Value Object is an immutable object that represents a concept from your domain. Unlike an entity (which has an identity), a Value Object is defined entirely by its attributes. Two Money objects with the same amount and currency are equal, regardless of when or where they were created:

```php
namespace App\Support\ValueObjects;

use App\Enums\Currency;
use InvalidArgumentException;

readonly class Money
{
    public function __construct(
        public int $amount,       // Amount in cents
        public Currency $currency,
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('Money amount cannot be negative.');
        }
    }

    public function add(Money $other): self
    {
        $this->ensureSameCurrency($other);

        return new self($this->amount + $other->amount, $this->currency);
    }

    public function subtract(Money $other): self
    {
        $this->ensureSameCurrency($other);

        return new self($this->amount - $other->amount, $this->currency);
    }

    public function multiply(float $factor): self
    {
        return new self((int) round($this->amount * $factor), $this->currency);
    }

    public function formatted(): string
    {
        return $this->currency->symbol() . number_format($this->amount / 100, 2);
    }

    public function equals(Money $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    private function ensureSameCurrency(Money $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException(
                "Cannot operate on different currencies: {$this->currency->value} and {$other->currency->value}",
            );
        }
    }
}
```

Now the discount function becomes impossible to misuse:

```php
function applyDiscount(Money $price, Money $discount): Money
{
    return $price->subtract($discount);
}

// Clear, self-documenting, and type-safe
$price = new Money(9999, Currency::USD);
$discount = new Money(1000, Currency::USD);
$final = applyDiscount($price, $discount); // Money(8999, USD)

// Cannot mix currencies — throws at runtime
$euros = new Money(1000, Currency::EUR);
$final = applyDiscount($price, $euros); // InvalidArgumentException
```

The `readonly` keyword ensures immutability — once created, a Money object cannot be changed. Every operation returns a new instance. This eliminates an entire class of bugs where shared mutable state leads to unexpected changes.

### Using Value Objects with Eloquent

Create a custom cast to use Value Objects with Eloquent:

```php
namespace App\Support\Casts;

use App\Enums\Currency;
use App\Support\ValueObjects\Money;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

/** @implements CastsAttributes<Money|null, Money|int|null> */
class MoneyCast implements CastsAttributes
{
    public function __construct(
        private readonly string $currencyField = 'currency',
    ) {}

    /** @param array<string, mixed> $attributes */
    public function get(Model $model, string $key, mixed $value, array $attributes): ?Money
    {
        if ($value === null) {
            return null;
        }

        return new Money(
            amount: (int) $value,
            currency: Currency::from($attributes[$this->currencyField] ?? 'USD'),
        );
    }

    /**
     * @param array<string, mixed> $attributes
     * @return array<string, int|string|null>
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): array
    {
        if ($value === null) {
            return [$key => null];
        }

        if ($value instanceof Money) {
            return [
                $key => $value->amount,
                $this->currencyField => $value->currency->value,
            ];
        }

        return [$key => (int) $value];
    }
}
```

Now use it in your model:

```php
protected function casts(): array
{
    return [
        'total' => MoneyCast::class,
    ];
}

// Usage
$order->total; // Money(amount: 9999, currency: Currency::USD)
$order->total->formatted(); // "$99.99"
$order->total->add(new Money(500, Currency::USD)); // Money(amount: 10499, ...)
```

The database stores an integer (cents) and a string (currency code). Your application works with a `Money` object that enforces its own rules. The cast handles the translation transparently.

### Other Common Value Objects

Money is the classic example, but Value Objects are useful for any domain concept with its own rules:

```php
readonly class Address
{
    public function __construct(
        public string $street,
        public string $city,
        public string $state,
        public string $zip,
        public string $country,
    ) {}

    public function fullAddress(): string
    {
        return "{$this->street}, {$this->city}, {$this->state} {$this->zip}, {$this->country}";
    }

    public function equals(Address $other): bool
    {
        return $this->street === $other->street
            && $this->city === $other->city
            && $this->state === $other->state
            && $this->zip === $other->zip
            && $this->country === $other->country;
    }
}
```

```php
use Carbon\CarbonImmutable;
use InvalidArgumentException;

readonly class DateRange
{
    public function __construct(
        public CarbonImmutable $start,
        public CarbonImmutable $end,
    ) {
        if ($start->isAfter($end)) {
            throw new InvalidArgumentException('Start date must be before end date.');
        }
    }

    public function days(): int
    {
        return $this->start->diffInDays($this->end);
    }

    public function contains(CarbonImmutable $date): bool
    {
        return $date->between($this->start, $this->end);
    }

    public function overlaps(DateRange $other): bool
    {
        return $this->start->isBefore($other->end) && $this->end->isAfter($other->start);
    }
}
```

The pattern is always the same: constructor validates, properties are readonly, methods return new instances. The object protects its own invariants — an `Address` always has all fields, a `DateRange` always has start before end.

## The State Pattern

The State Pattern is the most powerful pattern in this chapter. It replaces complex conditional logic with dedicated state classes, making your code extensible and maintainable.

### The Problem: Conditional Hell

Every application has entities that change state. An order goes from pending to processing to shipped. A subscription goes from trialing to active to cancelled. Without the State Pattern, this logic becomes a tangle of if/else statements:

```php
// Before: conditional logic scattered everywhere
public function ship(Order $order): void
{
    if ($order->status === 'pending') {
        throw new Exception('Cannot ship a pending order. Process it first.');
    }

    if ($order->status === 'cancelled') {
        throw new Exception('Cannot ship a cancelled order.');
    }

    if ($order->status === 'shipped') {
        throw new Exception('Order is already shipped.');
    }

    if ($order->status === 'delivered') {
        throw new Exception('Order is already delivered.');
    }

    $order->update(['status' => 'shipped', 'shipped_at' => now()]);
}
```

This code has several problems: the transition rules are scattered, adding a new status requires modifying every method, and there is no way to see all valid transitions at a glance.

### Step 1: Enums (Better, But Not Enough)

Enums improve type safety but do not solve the transition problem:

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';

    public function canTransitionTo(self $new): bool
    {
        return match ($this) {
            self::Pending => in_array($new, [self::Processing, self::Cancelled], true),
            self::Processing => in_array($new, [self::Shipped, self::Cancelled], true),
            self::Shipped => in_array($new, [self::Delivered], true),
            self::Delivered => false,
            self::Cancelled => false,
        };
    }
}
```

This centralizes the rules, but the enum grows with every new behavior. What if shipping an order also needs to send a notification? What if cancelling needs to refund the payment? The enum becomes a God object.

### Step 2: The State Pattern with spatie/model-states

The `spatie/model-states` package provides a clean implementation of the State Pattern for Eloquent models:

```bash
composer require spatie/laravel-model-states
```

Define an abstract base state:

```php
namespace App\States\Order;

use Spatie\ModelStates\State;
use Spatie\ModelStates\StateConfig;

abstract class OrderState extends State
{
    abstract public function label(): string;
    abstract public function color(): string;

    public static function config(): StateConfig
    {
        return parent::config()
            ->default(PendingState::class)
            ->allowTransition(PendingState::class, ProcessingState::class)
            ->allowTransition(PendingState::class, CancelledState::class)
            ->allowTransition(ProcessingState::class, ShippedState::class)
            ->allowTransition(ProcessingState::class, CancelledState::class)
            ->allowTransition(ShippedState::class, DeliveredState::class);
    }
}
```

Define each concrete state:

```php
class PendingState extends OrderState
{
    public function label(): string
    {
        return 'Pending';
    }

    public function color(): string
    {
        return 'yellow';
    }
}

class ProcessingState extends OrderState
{
    public function label(): string
    {
        return 'Processing';
    }

    public function color(): string
    {
        return 'blue';
    }
}

class ShippedState extends OrderState
{
    public function label(): string
    {
        return 'Shipped';
    }

    public function color(): string
    {
        return 'indigo';
    }
}

class DeliveredState extends OrderState
{
    public function label(): string
    {
        return 'Delivered';
    }

    public function color(): string
    {
        return 'green';
    }
}

class CancelledState extends OrderState
{
    public function label(): string
    {
        return 'Cancelled';
    }

    public function color(): string
    {
        return 'red';
    }
}
```

Use it in the model:

```php
use Spatie\ModelStates\HasStates;

class Order extends Model
{
    use HasStates;

    protected function casts(): array
    {
        return [
            'status' => OrderState::class,
        ];
    }
}
```

### Custom Transitions

For transitions that need side effects, create transition classes:

```php
namespace App\States\Order\Transitions;

use App\Models\Order;
use App\States\Order\ShippedState;
use Spatie\ModelStates\Transition;

class ShipOrderTransition extends Transition
{
    public function __construct(
        private readonly Order $order,
        private readonly string $trackingNumber,
    ) {}

    public function handle(): Order
    {
        $this->order->update([
            'status' => ShippedState::class,
            'tracking_number' => $this->trackingNumber,
            'shipped_at' => now(),
        ]);

        event(new OrderShipped($this->order));

        return $this->order->refresh();
    }
}
```

Register the transition:

```php
public static function config(): StateConfig
{
    return parent::config()
        ->allowTransition(ProcessingState::class, ShippedState::class, ShipOrderTransition::class);
}
```

Use it:

```php
$order->status->transitionTo(ShippedState::class, trackingNumber: 'TRACK-123');
```

Each transition is a standalone class with a single responsibility. Adding a new side effect means editing one class, not hunting through controllers and services for every place that changes a status.

### Querying by State

```php
// Find all pending orders
$pending = Order::whereState('status', PendingState::class)->get();

// Find orders in multiple states
$active = Order::whereState('status', [ProcessingState::class, ShippedState::class])->get();
```

## Organizing Type-Safe Code

```
app/
├── Enums/
│   ├── Currency.php
│   ├── PaymentMethod.php
│   └── UserRole.php
├── States/
│   └── Order/
│       ├── OrderState.php
│       ├── PendingState.php
│       ├── ProcessingState.php
│       ├── ShippedState.php
│       ├── DeliveredState.php
│       ├── CancelledState.php
│       └── Transitions/
│           ├── ShipOrderTransition.php
│           └── CancelOrderTransition.php
└── Support/
    ├── ValueObjects/
    │   ├── Money.php
    │   └── Address.php
    └── Casts/
        ├── MoneyCast.php
        └── AddressCast.php
```

For a deeper look at when enums are enough and when to upgrade to the state pattern — including transitions, side effects, and states that never change — see [States, Transitions, and Enums](/books/thinking-in-domains-in-laravel/states-transitions-and-enums) in *Thinking in Domains in Laravel*.

## When to Use Each

| Tool | Use When | Example |
|---|---|---|
| Enum | Fixed set of values with no transition rules | `Currency`, `PaymentMethod`, `UserRole` |
| Value Object | Immutable concept defined by its attributes | `Money`, `Address`, `DateRange` |
| State Pattern | Entity with lifecycle transitions and side effects | `OrderStatus`, `SubscriptionStatus`, `InvoiceStatus` |

## The Type Safety Checklist

1. **Replace magic strings with enums** — if a column has a fixed set of values, it should be an enum
2. **Add methods to enums** — `label()`, `color()`, `icon()` keep display logic with the data it describes
3. **Use `Rule::enum()` in validation** — never maintain a separate list of allowed values
4. **Cast enums in Eloquent** — work with enum instances, not raw strings
5. **Use Value Objects for domain concepts** — Money, Address, DateRange, anything with its own rules
6. **Make Value Objects readonly and immutable** — every operation returns a new instance
7. **Create custom Eloquent casts** for Value Objects — transparent conversion between database and domain
8. **Use the State Pattern for lifecycle transitions** — when entities move through states with side effects
9. **Register transitions as classes** — side effects belong in transition classes, not controllers
10. **Match exhaustiveness catches missing cases** — add a new enum case and PHP throws an error everywhere you forgot to handle it

## Summary

- Strings are the most dangerous type in your application. Every string comparison is a potential typo, every assignment a potential invalid value. Enums eliminate these bugs before they reach production.
- Backed enums replace magic strings with type-safe constants. They carry their own display logic (`label()`, `color()`, `icon()`), work natively with Eloquent casts, and validate input through `Rule::enum()`.
- `match` expressions on enums are exhaustive — add a new case and PHP tells you every place you forgot to handle it. `switch` statements on strings fail silently.
- Value Objects replace primitives with self-validating, immutable domain concepts. A `Money` object enforces non-negative amounts, prevents mixed-currency arithmetic, and formats itself for display.
- The `readonly` keyword on Value Objects guarantees immutability. Every operation returns a new instance, eliminating shared mutable state bugs.
- Custom Eloquent casts bridge Value Objects and the database — the database stores primitives, your application works with rich domain objects.
- The State Pattern replaces scattered conditional logic with dedicated state classes. All valid transitions are defined in one place, and each transition can carry its own side effects.
- Use `spatie/model-states` to implement the State Pattern in Laravel. Define an abstract base state with `config()`, concrete states for each status, and transition classes for side effects.
- Enums are for fixed sets of values. Value Objects are for immutable domain concepts. The State Pattern is for entities with lifecycle transitions. Use the simplest tool that fits your problem.

## References

- [Enumerations](https://www.php.net/manual/en/language.enumerations.php) — PHP Documentation
- [Backed Enumerations](https://www.php.net/manual/en/language.enumerations.backed.php) — PHP Documentation
- [Eloquent: Mutators & Casting — Enum Casting](https://laravel.com/docs/eloquent-mutators#enum-casting) — Laravel Documentation
- [Validation — Enum Rule](https://laravel.com/docs/validation#rule-enum) — Laravel Documentation
- [Eloquent: Mutators & Casting — Custom Casts](https://laravel.com/docs/eloquent-mutators#custom-casts) — Laravel Documentation
- [spatie/laravel-model-states](https://github.com/spatie/laravel-model-states) — Spatie, GitHub
- [Model States Documentation](https://spatie.be/docs/laravel-model-states) — Spatie
- [Using PHP Enums in Laravel](https://laravel-news.com/php-enums-in-laravel) — Laravel News
- [Value Objects in PHP](https://martinjoo.dev/value-objects-in-php) — Martin Joo
- [States, Transitions, and Enums](/books/thinking-in-domains-in-laravel/states-transitions-and-enums) — Thinking in Domains in Laravel
