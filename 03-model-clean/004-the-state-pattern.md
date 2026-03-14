Every non-trivial application has models that change state over time. An order starts as "pending," becomes "paid," then "shipped," and finally "delivered." A blog post begins as a "draft," moves to "in review," and eventually becomes "published." A support ticket is "open," then "in progress," then "resolved."

Managing these states seems simple at first — just add a `status` column and use string comparisons. But as your application grows, the logic around state transitions becomes increasingly complex. Different states have different rules, different allowed transitions, and different side effects. Before long, your codebase is littered with `if/else` chains that are difficult to maintain and easy to break.

Consider what happens when a customer asks: "Why was my order marked as delivered when it was never shipped?" You search the codebase and find twelve places that set the status column — controllers, jobs, commands, webhook handlers. One of them skips the "shipped" step under a specific edge case. This bug would have been impossible if the system enforced that "delivered" can only follow "shipped."

The State Pattern is a design pattern that solves this problem by encapsulating each state's behavior in a dedicated class. Instead of checking the status string everywhere, you ask the state object what it can do. This chapter walks you through the progression from simple strings to enums to the full State Pattern, so you understand not just how to use it, but when and why.

## Stage 1: The String Column

The simplest approach to managing state is a string column in the database:

```php
// Migration
$table->string('status')->default('pending');

// Usage
if ($order->status === 'pending') {
    // ...
}

if ($order->status === 'paid' || $order->status === 'shipped') {
    // ...
}
```

This works for the first few weeks. But it has serious problems:

**No type safety.** There is nothing preventing you from setting `$order->status = 'pneding'` (a typo) or `$order->status = 'banana'` (nonsense). The compiler will not catch it, and the bug will only surface at runtime.

**No discoverability.** When you encounter `$order->status`, you have no idea what the valid values are without searching the codebase or reading the migration.

**No behavior.** The string `'pending'` is just data. It cannot tell you what color to display, what label to show, or what transitions are allowed. That logic must live somewhere else, scattered across your controllers and views.

**No transition rules.** Nothing prevents `$order->status = 'delivered'` from being set directly on a pending order. Every place that changes the status is a potential source of invalid state transitions.

## Stage 2: PHP Enums

PHP 8.1 introduced backed enums, which solve the type safety and discoverability problems:

```php
// app/Enums/OrderStatus.php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Pending Payment',
            self::Paid => 'Paid',
            self::Shipped => 'Shipped',
            self::Delivered => 'Delivered',
            self::Cancelled => 'Cancelled',
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::Pending => 'yellow',
            self::Paid => 'blue',
            self::Shipped => 'indigo',
            self::Delivered => 'green',
            self::Cancelled => 'red',
        };
    }
}
```

You cast the enum in your model using the `casts()` method:

```php
class Order extends Model
{
    protected function casts(): array
    {
        return [
            'status' => OrderStatus::class,
        ];
    }
}
```

Now you can use the enum with full type safety:

```php
$order->status = OrderStatus::Paid;

if ($order->status === OrderStatus::Pending) {
    // ...
}

echo $order->status->label();  // "Pending Payment"
echo $order->status->color();  // "yellow"
```

Enums are a massive improvement over strings. They provide type safety, discoverability, and basic behavior (labels, colors). For many applications, enums are all you need. See [Enums, Value Objects, and Type Safety](/books/clean-code-in-laravel/enums-value-objects-and-type-safety) for a deep dive.

But enums have a limitation: they cannot enforce transition rules or handle side effects. There is nothing preventing you from setting an order's status directly from "pending" to "delivered," skipping the "paid" and "shipped" steps entirely. And if transitioning from "pending" to "paid" should trigger a payment confirmation email, that logic must live somewhere outside the enum.

You could add a `canTransitionTo()` method to the enum, but now the enum is responsible for transition rules, display logic, and — eventually — side effects. It becomes a God object. The enum knows too much.

## Stage 3: The State Pattern

The State Pattern solves these problems by replacing the enum with a set of state classes, each representing one possible state. Each state class defines its own behavior, its own allowed transitions, and its own side effects.

The `spatie/laravel-model-states` package provides an elegant implementation of this pattern for Laravel:

```bash
composer require spatie/laravel-model-states
```

### Defining the Base State

First, create an abstract base state class that defines the configuration — which states exist and which transitions are allowed:

```php
// src/Domain/Order/States/OrderState.php
namespace Domain\Order\States;

use Spatie\ModelStates\State;
use Spatie\ModelStates\StateConfig;

abstract class OrderState extends State
{
    abstract public function label(): string;

    abstract public function color(): string;

    public static function config(): StateConfig
    {
        return parent::config()
            ->default(PendingOrderState::class)
            ->allowTransition(PendingOrderState::class, PaidOrderState::class)
            ->allowTransition(PaidOrderState::class, ShippedOrderState::class)
            ->allowTransition(ShippedOrderState::class, DeliveredOrderState::class)
            ->allowTransition(
                [PendingOrderState::class, PaidOrderState::class],
                CancelledOrderState::class,
            );
    }
}
```

This configuration is a declaration of your business rules. It says: a pending order can become paid. A paid order can become shipped. A shipped order can become delivered. And both pending and paid orders can be cancelled. No other transitions are allowed. Reading the `config()` method gives you the complete state machine at a glance — something that was impossible when transition rules were scattered across twelve files.

### Defining Concrete States

Each concrete state class implements the abstract methods:

```php
// src/Domain/Order/States/PendingOrderState.php
namespace Domain\Order\States;

class PendingOrderState extends OrderState
{
    public function label(): string
    {
        return 'Pending Payment';
    }

    public function color(): string
    {
        return 'yellow';
    }
}
```

```php
// src/Domain/Order/States/PaidOrderState.php
namespace Domain\Order\States;

class PaidOrderState extends OrderState
{
    public function label(): string
    {
        return 'Paid';
    }

    public function color(): string
    {
        return 'blue';
    }
}
```

```php
// src/Domain/Order/States/ShippedOrderState.php
namespace Domain\Order\States;

class ShippedOrderState extends OrderState
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
```

```php
// src/Domain/Order/States/DeliveredOrderState.php
namespace Domain\Order\States;

class DeliveredOrderState extends OrderState
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
```

```php
// src/Domain/Order/States/CancelledOrderState.php
namespace Domain\Order\States;

class CancelledOrderState extends OrderState
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

### Using States on the Model

Configure the model to use the state:

```php
// src/Domain/Order/Models/Order.php
namespace Domain\Order\Models;

use Domain\Order\States\OrderState;
use Illuminate\Database\Eloquent\Model;
use Spatie\ModelStates\HasStates;

class Order extends Model
{
    use HasStates;

    protected function casts(): array
    {
        return [
            'state' => OrderState::class,
        ];
    }
}
```

Now you can interact with the state:

```php
$order = Order::find(1);

// Access state behavior
echo $order->state->label();  // "Pending Payment"
echo $order->state->color();  // "yellow"

// Check if a transition is allowed
$order->state->canTransitionTo(PaidOrderState::class);  // true
$order->state->canTransitionTo(DeliveredOrderState::class);  // false

// Perform a transition
$order->state->transitionTo(PaidOrderState::class);

// Query by state
$pendingOrders = Order::whereState('state', PendingOrderState::class)->get();
$activeOrders = Order::whereNotState('state', CancelledOrderState::class)->get();
```

If you try an invalid transition, the package throws an exception:

```php
// This will throw a CouldNotPerformTransition exception
$order->state->transitionTo(DeliveredOrderState::class);
// Because: Pending → Delivered is not an allowed transition
```

This is the key difference from enums. With an enum, invalid transitions fail silently or require manual enforcement. With the State Pattern, they are structurally impossible.

### Transitions with Side Effects

The real power of the State Pattern emerges when transitions need to perform side effects. Instead of scattering this logic across your controllers, you encapsulate it in dedicated Transition classes:

```php
// src/Domain/Order/States/Transitions/PendingToPaidTransition.php
namespace Domain\Order\States\Transitions;

use Domain\Order\Models\Order;
use Domain\Order\States\PaidOrderState;
use Domain\Order\Events\OrderPaid;
use Spatie\ModelStates\Transition;

class PendingToPaidTransition extends Transition
{
    public function __construct(
        private readonly Order $order,
        private readonly string $paymentReference,
    ) {}

    public function handle(): Order
    {
        $this->order->update([
            'state' => PaidOrderState::class,
            'payment_reference' => $this->paymentReference,
            'paid_at' => now(),
        ]);

        event(new OrderPaid($this->order));

        return $this->order;
    }
}
```

Register the transition in your base state:

```php
public static function config(): StateConfig
{
    return parent::config()
        ->default(PendingOrderState::class)
        ->allowTransition(
            PendingOrderState::class,
            PaidOrderState::class,
            PendingToPaidTransition::class,
        )
        ->allowTransition(PaidOrderState::class, ShippedOrderState::class)
        ->allowTransition(ShippedOrderState::class, DeliveredOrderState::class)
        ->allowTransition(
            [PendingOrderState::class, PaidOrderState::class],
            CancelledOrderState::class,
        );
}
```

Now when you transition from pending to paid, the transition class handles everything:

```php
$order->state->transitionTo(PaidOrderState::class, paymentReference: 'pi_abc123');
```

The transition updates the database, sets the payment reference, records the timestamp, and dispatches the event. All in one place, all testable, all explicit.

Why does this matter? Because the same transition can be triggered from a controller, an Artisan command, a webhook handler, or a queued job. If the "send payment confirmation" logic lives in a controller, every other entry point must duplicate it. In a Transition class, the side effect runs wherever the transition happens.

## Adding State-Specific Behavior

One of the most powerful aspects of the State Pattern is that each state can define its own behavior. Instead of scattering `if ($order->status === 'pending')` checks across your application, you ask the state object what it allows:

```php
abstract class OrderState extends State
{
    abstract public function label(): string;
    abstract public function color(): string;

    public function canBeEdited(): bool
    {
        return false;
    }

    public function canBeCancelled(): bool
    {
        return false;
    }

    public function canBeRefunded(): bool
    {
        return false;
    }
}

class PendingOrderState extends OrderState
{
    public function label(): string { return 'Pending Payment'; }
    public function color(): string { return 'yellow'; }

    public function canBeEdited(): bool
    {
        return true;  // Only pending orders can be edited
    }

    public function canBeCancelled(): bool
    {
        return true;
    }
}

class PaidOrderState extends OrderState
{
    public function label(): string { return 'Paid'; }
    public function color(): string { return 'blue'; }

    public function canBeCancelled(): bool
    {
        return true;  // Paid orders can still be cancelled (refund)
    }

    public function canBeRefunded(): bool
    {
        return true;
    }
}

class ShippedOrderState extends OrderState
{
    public function label(): string { return 'Shipped'; }
    public function color(): string { return 'indigo'; }

    // Shipped orders cannot be edited, cancelled, or refunded
    // All methods return false (the default)
}
```

In your Blade template or Inertia component, you can use these methods directly:

```blade
@if($order->state->canBeEdited())
    <a href="{{ route('orders.edit', $order) }}">Edit Order</a>
@endif

@if($order->state->canBeCancelled())
    <form action="{{ route('orders.cancel', $order) }}" method="POST">
        @csrf
        <button type="submit">Cancel Order</button>
    </form>
@endif
```

No more `if ($order->status === 'pending' || $order->status === 'paid')` scattered across your views. The state object knows what it can do. Adding a new state means creating one class that answers these questions — not hunting through every template and controller.

## Testing State Transitions

State classes and transitions are easy to test in isolation. You do not need HTTP requests or full application bootstrapping:

```php
it('allows pending orders to transition to paid', function (): void {
    $order = Order::factory()->create(['state' => PendingOrderState::class]);

    $order->state->transitionTo(PaidOrderState::class, paymentReference: 'pi_test123');

    expect($order->fresh()->state)
        ->toBeInstanceOf(PaidOrderState::class);
});

it('prevents pending orders from transitioning directly to delivered', function (): void {
    $order = Order::factory()->create(['state' => PendingOrderState::class]);

    $order->state->transitionTo(DeliveredOrderState::class);
})->throws(CouldNotPerformTransition::class);

it('fires OrderPaid event on payment transition', function (): void {
    Event::fake();

    $order = Order::factory()->create(['state' => PendingOrderState::class]);

    $order->state->transitionTo(PaidOrderState::class, paymentReference: 'pi_test123');

    Event::assertDispatched(OrderPaid::class);
});

it('records payment reference and timestamp on payment', function (): void {
    $order = Order::factory()->create(['state' => PendingOrderState::class]);

    $order->state->transitionTo(PaidOrderState::class, paymentReference: 'pi_test123');

    expect($order->fresh())
        ->payment_reference->toBe('pi_test123')
        ->paid_at->not->toBeNull();
});
```

Test the state-specific behavior methods too:

```php
it('only allows editing on pending orders', function (string $stateClass, bool $expected): void {
    $order = Order::factory()->create(['state' => $stateClass]);

    expect($order->state->canBeEdited())->toBe($expected);
})->with([
    'pending can edit' => [PendingOrderState::class, true],
    'paid cannot edit' => [PaidOrderState::class, false],
    'shipped cannot edit' => [ShippedOrderState::class, false],
    'delivered cannot edit' => [DeliveredOrderState::class, false],
    'cancelled cannot edit' => [CancelledOrderState::class, false],
]);
```

## When to Use What

Choosing between strings, enums, and the State Pattern depends on the complexity of your status management:

| Scenario                       | Recommended Approach                  |
| ------------------------------ | ------------------------------------- |
| Simple flag (active/inactive)  | Boolean column                        |
| Status with no behavior        | Enum                                  |
| Status with labels and colors  | Enum with methods                     |
| Status with transition rules   | State Pattern                         |
| Transitions with side effects  | State Pattern with Transition classes |
| Status-specific business logic | State Pattern                         |

The general rule: start with an enum. If you find yourself adding complex transition logic, validation rules for which transitions are allowed, or state-specific behavior beyond simple labels, upgrade to the State Pattern. The migration is straightforward — you are replacing one cast with another.

Do not jump to the State Pattern for a status field that has three values and no transition rules. An enum is simpler and sufficient. The State Pattern earns its keep when the answer to "can this status change to that status?" depends on business rules, and when transitions carry side effects that must run regardless of where the transition is triggered.

For a domain-oriented take on states — including transition classes, states without transitions, and the interplay between states and enums — see [States, Transitions, and Enums](/books/thinking-in-domains-in-laravel/states-transitions-and-enums) in *Thinking in Domains in Laravel*.

## The State Pattern Checklist

1. **Start with an enum** — upgrade to the State Pattern only when transition rules or side effects justify it
2. **Define all transitions in `config()`** — the state machine should be readable in one place
3. **Use Transition classes for side effects** — sending emails, updating timestamps, dispatching events
4. **Define state-specific behavior on state classes** — `canBeEdited()`, `canBeCancelled()` instead of status checks in views
5. **Default abstract methods to `false`** — override only in states where the behavior applies
6. **Test transitions explicitly** — verify both valid transitions and that invalid ones throw exceptions
7. **Use `whereState()` for queries** — type-safe querying by state class
8. **Keep state classes small** — each state should only define its own behavior, not the behavior of other states

## Summary

- State management starts simple with string columns but quickly becomes unmanageable as business rules grow. Transition logic scatters across controllers, jobs, and commands, making bugs like invalid state jumps invisible until production.
- PHP enums solve type safety and discoverability but cannot enforce transition rules or carry side effects. They are the right choice for statuses with no lifecycle — `Currency`, `PaymentMethod`, `UserRole`.
- The State Pattern encapsulates each state in a dedicated class. All valid transitions are declared in a single `config()` method, making the entire state machine readable at a glance.
- Invalid transitions throw a `CouldNotPerformTransition` exception. With enums, invalid transitions fail silently. With the State Pattern, they are structurally impossible.
- Transition classes encapsulate side effects — sending emails, updating timestamps, dispatching events. They run wherever the transition is triggered, not just from controllers.
- State-specific behavior methods (`canBeEdited()`, `canBeCancelled()`) replace scattered conditional checks in views and controllers. Adding a new state means creating one class, not modifying every template.
- The State Pattern is testable in isolation. You can verify valid transitions, assert that invalid ones throw, check side effects, and test behavior methods with Pest datasets.
- Start with an enum. Upgrade to the State Pattern when you need enforced transitions, side effects, or state-specific business logic. The migration is straightforward — you replace one Eloquent cast with another.

## References

- [spatie/laravel-model-states](https://github.com/spatie/laravel-model-states) — Spatie, GitHub
- [Model States Documentation](https://spatie.be/docs/laravel-model-states) — Spatie
- [State Pattern](https://refactoring.guru/design-patterns/state) — Refactoring Guru
- [State Pattern in PHP](https://refactoring.guru/design-patterns/state/php/example) — Refactoring Guru
- [Modeling States and Transitions in Laravel](https://freek.dev/1686-a-package-to-work-with-model-states-in-laravel) — Freek Van der Herten
- [States, Transitions, and Enums](/books/thinking-in-domains-in-laravel/states-transitions-and-enums) — Thinking in Domains in Laravel
- [Enums, Value Objects, and Type Safety](/books/clean-code-in-laravel/enums-value-objects-and-type-safety) — Clean Code in Laravel