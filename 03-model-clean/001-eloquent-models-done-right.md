Open any Laravel project that has been in production for more than a year, and find the `User` model. Chances are it is the longest file in the application. It sends emails, calculates subscription status, formats names for display, generates avatar URLs, checks permissions, syncs data to third-party services, and contains a dozen scopes that are only used in one place. The model started clean — a few relationships, a cast or two — but every feature added "just one more method," and now it is 800 lines of tangled responsibilities.

This happens because models are the most convenient place to put logic. They have access to the data, they are available everywhere, and adding a method to a model is the path of least resistance. But convenience is not architecture. A model that does everything is a model that is hard to test, hard to refactor, and hard for new developers to understand.

A clean model focuses on three things: **data definition**, **relationships**, and **data access**. Everything else belongs somewhere else — in [Actions](/books/clean-code-in-laravel/actions), [Services](/books/clean-code-in-laravel/organizing-your-application), [Jobs](/books/clean-code-in-laravel/jobs), Policies, or [API Resources](https://laravel.com/docs/eloquent-resources). This chapter shows you what a model should contain, what it should not, and how to draw the line.

## The Anatomy of a Clean Model

Here is a well-structured Eloquent model:

```php
namespace App\Models;

use App\Enums\OrderStatus;
use App\Support\Casts\MoneyCast;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Order extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'user_id',
        'shipping_address_id',
        'subtotal',
        'tax',
        'discount',
        'total',
        'status',
        'notes',
        'placed_at',
    ];

    protected function casts(): array
    {
        return [
            'status' => OrderStatus::class,
            'subtotal' => MoneyCast::class,
            'tax' => MoneyCast::class,
            'discount' => MoneyCast::class,
            'total' => MoneyCast::class,
            'placed_at' => 'datetime',
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function shippingAddress(): BelongsTo
    {
        return $this->belongsTo(Address::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public function payments(): HasMany
    {
        return $this->hasMany(Payment::class);
    }

    protected function isPaid(): Attribute
    {
        return Attribute::make(
            get: fn (): bool => $this->status === OrderStatus::Paid,
        );
    }

    protected function itemCount(): Attribute
    {
        return Attribute::make(
            get: fn (): int => $this->items->count(),
        );
    }
}
```

Notice what this model does **not** contain: no email sending, no PDF generation, no complex business logic. It defines the data shape, the relationships, and a few computed properties. That is all.

The structure follows a consistent order: traits, properties, casts, relationships, accessors, scopes. This ordering is a convention — not a rule — but consistency within a project makes models predictable. When a developer opens any model, they know where to find what they are looking for.

## The `casts()` Method

Laravel 12 uses the `casts()` method instead of the `$casts` property. The method approach is more flexible because it can use constructor arguments and conditional logic:

```php
protected function casts(): array
{
    return [
        'status' => OrderStatus::class,
        'metadata' => 'array',
        'total' => MoneyCast::class,
        'placed_at' => 'datetime',
        'is_gift' => 'boolean',
        'email_verified_at' => 'datetime',
    ];
}
```

Casts are one of the most underused features in Laravel. Every column that represents something more specific than a raw string or integer should have a cast. Dates should be `datetime`, not raw strings. Booleans should be `boolean`, not integers you compare with `=== 1`. JSON columns should be `array` or `collection`, not strings you `json_decode()` manually. Statuses should be [enums](/books/clean-code-in-laravel/enums-value-objects-and-type-safety), not strings you compare with `===`.

The cost of a cast is near zero. The benefit is type safety at every point where you read or write the attribute. When `status` is cast to `OrderStatus`, you cannot accidentally assign `'pendign'` — PHP's type system catches it before it reaches the database.

### Custom Casts

For values that are more than a primitive type but less than a relationship, custom casts let you work with rich objects instead of raw database values:

```php
namespace App\Support\Casts;

use App\Support\ValueObjects\Money;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

/** @implements CastsAttributes<Money|null, int|null> */
class MoneyCast implements CastsAttributes
{
    public function get(Model $model, string $key, mixed $value, array $attributes): ?Money
    {
        if ($value === null) {
            return null;
        }

        return new Money(
            amount: (int) $value,
            currency: $attributes['currency'] ?? 'USD',
        );
    }

    public function set(Model $model, string $key, mixed $value, array $attributes): ?int
    {
        if ($value === null) {
            return null;
        }

        return $value instanceof Money ? $value->amount : (int) $value;
    }
}
```

Now `$order->total` returns a `Money` object with formatting methods, arithmetic, and currency awareness — not a raw integer that you have to remember is stored in cents. We cover value objects and custom casts in detail in [Enums, Value Objects, and Type Safety](/books/clean-code-in-laravel/enums-value-objects-and-type-safety).

## Relationships: The Right Way

Always define return types on relationships. This helps your IDE, static analysis tools, and other developers understand your model:

```php
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function items(): HasMany
{
    return $this->hasMany(OrderItem::class);
}

public function tags(): BelongsToMany
{
    return $this->belongsToMany(Tag::class)
        ->withTimestamps()
        ->withPivot('sort_order');
}

public function latestPayment(): HasOne
{
    return $this->hasOne(Payment::class)->latestOfMany();
}
```

### Constraining Relationships

Use constrained relationships for common queries that represent a meaningful subset of the related data:

```php
public function activeItems(): HasMany
{
    return $this->hasMany(OrderItem::class)
        ->where('is_cancelled', false);
}

public function highValuePayments(): HasMany
{
    return $this->hasMany(Payment::class)
        ->where('amount', '>', 1000);
}
```

Constrained relationships are useful when the constraint represents a *concept* — "active items" or "high-value payments" are things your domain talks about. If the constraint is ad-hoc and only used in one place, use a scope or a query builder method instead of polluting the model with a relationship that is really just a filtered query.

### Handling Nullable Relationships

When a relationship might be `null`, you need to handle it safely. Laravel provides `withDefault()` to return a fallback model when the relationship is empty, but this is a pattern you should avoid. Fabricating a fake model with made-up data (`'name' => 'Unassigned'`) introduces phantom objects that look real but are not persisted, have no ID, and can cause subtle bugs downstream — especially if that object gets passed to code that expects a real model.

Instead, use the nullsafe operator:

```php
// The relationship is nullable — handle it explicitly
$managerName = $employee->manager?->name ?? 'Unassigned';
```

This is clear about what is happening: the relationship might be null, and you are providing a display fallback. The nullsafe operator is a language-level feature that every PHP developer recognizes. It does not fabricate objects.

For nullable relationships, document the nullability in a PHPDoc block at the top of the model. Eloquent models are full of magic properties that are invisible without documentation — columns, accessors, and relationships are all accessed through `__get()`. A PHPDoc block makes them visible to your IDE and to other developers:

```php
/**
 * @property int $id
 * @property string $name
 * @property int|null $manager_id
 * @property-read User|null $manager
 * @property-read Collection<int, Project> $projects
 */
class Employee extends Model
{
    public function manager(): BelongsTo
    {
        return $this->belongsTo(User::class, 'manager_id');
    }
}
```

The `@property-read User|null $manager` annotation tells your IDE and every developer reading the code that this relationship is nullable. When someone types `$employee->manager->`, their IDE will warn them that this might be null. This is better protection than `withDefault()` because it catches the problem at development time, not by hiding it behind a fake object at runtime.

You do not need to document every column and relationship on every model from day one. But for models that are central to your domain — the ones other developers interact with regularly — a PHPDoc block pays for itself quickly.

### Relationship Methods Are Not Query Methods

A common mistake is adding query logic to relationship methods — conditions that change based on runtime state:

```php
// Before: the relationship changes based on who is looking
public function posts(): HasMany
{
    if (auth()->user()?->isAdmin()) {
        return $this->hasMany(Post::class);
    }

    return $this->hasMany(Post::class)->where('is_published', true);
}
```

This breaks eager loading, caching, and makes the relationship unpredictable. A relationship should always return the same result for the same data. If you need filtered results, use a scope, a constrained relationship with a clear name like `publishedPosts()`, or filter in the [controller](/books/clean-code-in-laravel/controllers).

## Accessors and Mutators

Laravel 12 uses the `Attribute` class for accessors and mutators:

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

// Accessor only
protected function fullName(): Attribute
{
    return Attribute::make(
        get: fn (): string => "{$this->first_name} {$this->last_name}",
    );
}

// Accessor and Mutator
protected function email(): Attribute
{
    return Attribute::make(
        get: fn (string $value): string => strtolower($value),
        set: fn (string $value): string => strtolower(trim($value)),
    );
}
```

### When to Use Accessors vs. When Not To

Accessors are for derived data that is a direct function of the model's own attributes. `$user->fullName` combines `first_name` and `last_name` — it is a property of the user, derived from the user's data. That belongs on the model.

But accessors are not a place for business logic. Consider this:

```php
// Before: business logic disguised as an accessor
protected function discountPercentage(): Attribute
{
    return Attribute::make(
        get: function (): float {
            if ($this->isVip() && $this->orders()->count() > 50) {
                return 0.20;
            }

            if ($this->created_at->diffInYears(now()) > 2) {
                return 0.10;
            }

            return 0.0;
        },
    );
}
```

This accessor queries the database, applies business rules, and will behave differently depending on when you call it. It looks like a simple property — `$user->discountPercentage` — but it hides significant complexity. This belongs in a Service like `DiscountCalculator` or an [Action](/books/clean-code-in-laravel/actions), not in an accessor.

The rule: if an accessor needs to query related models, call external services, or apply business rules that could change independently of the model, it does not belong on the model.

### Generated Columns: When the Database Should Do the Work

Not every derived value needs an accessor. When you need to search, sort, or index a computed value — like a `full_name` built from `first_name` and `last_name` — an accessor is invisible to the database. You end up writing `whereRaw("CONCAT(first_name, ' ', last_name) LIKE ?", [...])`, which cannot use an index and scans every row.

MySQL [generated columns](https://dev.mysql.com/doc/refman/8.4/en/create-table-generated-columns.html) solve this by computing the value at the database level. Laravel supports them natively with `virtualAs()` and `storedAs()` in migrations:

```php
Schema::table('users', function (Blueprint $table): void {
    $table->string('full_name')
        ->virtualAs("CONCAT(first_name, ' ', last_name)")
        ->nullable();
});
```

Now `full_name` is a real column — Eloquent reads it like any other attribute, and you can query, sort, and (with `STORED`) index it. No accessor needed.

The short version: use generated columns when you need the database to *know about* the derived value. Use accessors when the value is only for display or involves PHP-specific logic. They are complementary — a model might have a generated `full_name` column for querying *and* a `formattedName` accessor for locale-specific presentation.

For a deep dive into virtual vs. stored columns, indexing strategies, and more practical examples, see [MySQL Generated Columns in Laravel](/blog/mysql-generated-columns-in-laravel).

## Scopes

Scopes encapsulate common query constraints. Use them for queries that are reused across your application:

```php
use Illuminate\Database\Eloquent\Builder;

public function scopeActive(Builder $query): void
{
    $query->where('is_active', true);
}

public function scopePlacedBetween(Builder $query, Carbon $from, Carbon $to): void
{
    $query->whereBetween('placed_at', [$from, $to]);
}

public function scopeForStatus(Builder $query, OrderStatus $status): void
{
    $query->where('status', $status);
}

// Usage
Order::active()->forStatus(OrderStatus::Pending)->get();
Order::placedBetween(now()->subMonth(), now())->get();
```

### Keep Scopes Composable

Each scope should apply one constraint. When you need a combination, compose them in the query — do not create "combo scopes" that bundle unrelated conditions:

```php
// Before: this scope does three things
public function scopeReadyToShip(Builder $query): void
{
    $query->where('status', OrderStatus::Paid)
        ->where('is_active', true)
        ->whereHas('items', fn (Builder $q) => $q->where('in_stock', true));
}

// After: compose simple scopes
Order::query()
    ->forStatus(OrderStatus::Paid)
    ->active()
    ->whereHas('items', fn (Builder $q) => $q->where('in_stock', true))
    ->get();
```

The exception is when the combination represents a domain concept that has a single, stable definition. If "ready to ship" is a term your business uses consistently, a named scope is fine. But if you are just bundling conditions for convenience, compose them instead.

However, when your model accumulates many scopes, it is a sign that you need a [Custom Query Builder](/books/clean-code-in-laravel/custom-query-builders-and-collections) — which we cover in the next chapter.

## Model Events and Observers

Model events fire at specific points in a model's lifecycle — `creating`, `created`, `updating`, `updated`, `deleting`, `deleted`, and more. They are useful for data-integrity logic that must always happen when the model changes, regardless of where in the application the change originates.

### Inline Events for Data Integrity

Use the `booted()` method for simple, data-focused logic:

```php
protected static function booted(): void
{
    static::creating(function (Order $order): void {
        $order->order_number ??= 'ORD-' . strtoupper(Str::random(8));
    });

    static::deleting(function (Order $order): void {
        $order->items()->delete();
    });
}
```

These are data-integrity operations — generating a default value and cascading a delete. They are simple, they do not call external services, and they must always happen regardless of how the model is created or deleted. This is the correct use of model events.

### Observers for Side Effects

For complex side effects — sending notifications, syncing to external services, dispatching [Jobs](/books/clean-code-in-laravel/jobs) — use an Observer. Observers separate the side effect from the model, making both easier to test and maintain:

```php
namespace App\Observers;

use App\Models\Order;

class OrderObserver
{
    public function created(Order $order): void
    {
        event(new OrderPlaced($order));
    }

    public function updating(Order $order): void
    {
        if ($order->isDirty('status')) {
            event(new OrderStatusChanged($order));
        }
    }
}
```

Register the Observer in your `AppServiceProvider`:

```php
use App\Models\Order;
use App\Observers\OrderObserver;

public function boot(): void
{
    Order::observe(OrderObserver::class);
}
```

### The Hidden Cost of Model Events

Model events have a subtle problem: they create invisible coupling. When a developer writes `Order::create([...])`, there is no indication in that line of code that an email will be sent, a webhook will fire, or an external API will be called. The side effects are hidden in an Observer or a `booted()` method that the developer may not know about.

This becomes painful in tests. A test that creates an order for a simple assertion suddenly triggers a cascade of events, notifications, and Job dispatches. You end up wrapping tests in `Event::fake()` and `Notification::fake()` — not because the test cares about events, but because the model's hidden side effects interfere with the assertion.

The guidelines are:

- **Model events** are for data integrity — generating defaults, cascading deletes, maintaining denormalized columns. These are things that *must* happen every time, including in tests.
- **Observers** are for side effects — notifications, syncs, analytics. These are things you want to turn off in tests.
- **Neither** should contain business logic. If the side effect involves decisions, conditional branching, or orchestration, it belongs in an [Action](/books/clean-code-in-laravel/actions) or a [Job](/books/clean-code-in-laravel/jobs).

When model events become complex enough that you are spending more time managing their side effects than benefiting from the automation, consider dispatching domain events explicitly from [Actions](/books/clean-code-in-laravel/actions) instead. Explicit is better than magic.

## `$fillable` vs. `$guarded`

Laravel protects against mass assignment — accidentally setting columns that should not be user-controlled — using either `$fillable` (a whitelist) or `$guarded` (a blacklist). Both approaches work, and both are valid. What matters is that your team picks one and uses it consistently.

### The `$fillable` Approach

List every column that can be mass-assigned:

```php
protected $fillable = [
    'user_id',
    'shipping_address_id',
    'subtotal',
    'tax',
    'discount',
    'total',
    'status',
    'notes',
    'placed_at',
];
```

The advantage is explicitness. A new column is not mass-assignable until you add it to the list. The downside is maintenance — every migration that adds a column requires updating `$fillable` too, and forgetting to do so produces a silent bug where the column is never set.

### The `$guarded` Approach

List the columns that should *never* be mass-assigned:

```php
protected $guarded = ['id'];
```

This is the approach Taylor Otwell uses in most of his projects. Every column is mass-assignable by default, and you only guard the ones that must be protected — typically just the primary key. The advantage is less boilerplate and no "forgot to add to fillable" bugs. The trade-off is that a new column is automatically mass-assignable, which requires that your [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation) properly validate every incoming field.

The important thing is not which approach you choose — it is that you use [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation) to validate input before it reaches the model. Mass assignment protection is a safety net, not a substitute for validation.

## Keeping Models Lean

Models grow because they are the most accessible place to put logic. Every time someone asks "where should this go?", the model is the easiest answer. Keeping models lean requires active discipline and clear conventions.

### Extract Behavior Into Traits (Concerns)

When a model needs behavior that is conceptually separate from its core data, extract it into a [Concern](/books/clean-code-in-laravel/organizing-your-application):

```php
// app/Models/Concerns/HasSubscription.php
namespace App\Models\Concerns;

use App\Models\Subscription;
use Illuminate\Database\Eloquent\Relations\HasOne;

trait HasSubscription
{
    public function subscription(): HasOne
    {
        return $this->hasOne(Subscription::class);
    }

    public function isSubscribed(): bool
    {
        return $this->subscription !== null
            && $this->subscription->isActive();
    }

    public function onTrial(): bool
    {
        return $this->subscription?->onTrial() ?? false;
    }
}
```

```php
class User extends Authenticatable
{
    use HasSubscription;
}
```

Now `$user->isSubscribed()` works the same way, but the subscription logic is isolated in its own file. The User model stays focused on user data, and the subscription behavior is independently readable and testable.

### Promote Scopes to Query Builders

When a model has more than five or six scopes, it is time to extract them into a [Custom Query Builder](/books/clean-code-in-laravel/custom-query-builders-and-collections). This is covered in the next chapter.

### Move Business Logic to Actions

If a model method orchestrates multiple steps, calls external services, or has side effects beyond data persistence, it belongs in an [Action](/books/clean-code-in-laravel/actions):

```php
// Before: business logic in the model
class Order extends Model
{
    public function cancel(): void
    {
        $this->update(['status' => OrderStatus::Cancelled]);

        foreach ($this->items as $item) {
            $item->product->increment('stock', $item->quantity);
        }

        $this->user->notify(new OrderCancelledNotification($this));

        if ($this->isPaid) {
            app(StripeGateway::class)->refund($this->payment_intent_id);
        }
    }
}

// After: an Action handles the orchestration
class CancelOrderAction
{
    public function __construct(
        private readonly StripeGateway $stripe,
    ) {}

    public function execute(Order $order): void
    {
        $order->update(['status' => OrderStatus::Cancelled]);

        $order->items->each(function (OrderItem $item): void {
            $item->product->increment('stock', $item->quantity);
        });

        $order->user->notify(new OrderCancelledNotification($order));

        if ($order->isPaid) {
            $this->stripe->refund($order->payment_intent_id);
        }
    }
}
```

The model's `cancel()` method did four things: update status, restore stock, send a notification, and process a refund. That is not data access — it is business orchestration. Moving it to an Action makes the dependencies explicit (the Stripe gateway is injected, not resolved from the container inside the model), makes it testable without a full model instance, and keeps the model focused on what it knows about: its own data.

## What Does NOT Belong in a Model

| Does Not Belong | Where It Goes | Why |
|---|---|---|
| Sending emails or notifications | [Jobs](/books/clean-code-in-laravel/jobs) or Listeners | Side effect, not data |
| Generating PDFs or exports | [Actions](/books/clean-code-in-laravel/actions) or [Services](/books/clean-code-in-laravel/organizing-your-application) | Business operation |
| Complex calculations | [Services](/books/clean-code-in-laravel/organizing-your-application) | Reusable logic with dependencies |
| Authorization checks | [Policies](https://laravel.com/docs/authorization#creating-policies) | Separate concern |
| API response formatting | [API Resources](https://laravel.com/docs/eloquent-resources) | Presentation layer |
| Multi-step business logic | [Actions](/books/clean-code-in-laravel/actions) | Orchestration |
| HTTP or API calls | [Services](/books/clean-code-in-laravel/organizing-your-application) or [Jobs](/books/clean-code-in-laravel/jobs) | External dependency |
| View formatting (`toHtml`, `toLabel`) | [View Models](/books/clean-code-in-laravel/view-models) or Blade components | Presentation layer |

For a domain-oriented approach to keeping models lean — including custom query builders, custom collections, and event-driven models organized by domain — see [Eloquent Without the Bloat](/books/thinking-in-domains-in-laravel/eloquent-without-the-bloat) in *Thinking in Domains in Laravel*.

## Common Model Anti-Patterns

### The God Model

A model with 50+ methods, 20 scopes, 15 relationships, and business logic mixed in with data access. The `User` model is the usual victim. The fix is extraction: move scopes to a [Custom Query Builder](/books/clean-code-in-laravel/custom-query-builders-and-collections), behavior to Concerns, business logic to [Actions](/books/clean-code-in-laravel/actions), and query logic to dedicated classes.

### The `toArray()` Override

Overriding `toArray()` to control API output ties your API's data contract to the model's internal representation. When the model changes, the API changes — often unintentionally. Use [API Resources](https://laravel.com/docs/eloquent-resources) instead. They give you explicit control over the API response shape without coupling it to the model.

### Abusing Accessors for Performance-Sensitive Logic

Accessors look like properties, which makes developers treat them as cheap. But an accessor that queries a relationship or performs a calculation runs every time it is accessed — including inside loops:

```php
// This triggers a query on every iteration
foreach ($users as $user) {
    echo $user->orderCount; // Accessor that calls $this->orders()->count()
}
```

If the accessor hits the database, use eager loading or a subquery select instead:

```php
$users = User::withCount('orders')->get();

foreach ($users as $user) {
    echo $user->orders_count; // Pre-loaded, no extra query
}
```

### Putting Validation in the Model

Some patterns from other frameworks encourage model-level validation — a `$rules` property or a `validate()` method on the model. In Laravel, validation belongs in [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation). Form Requests run before the controller action, provide built-in authorization, return meaningful error responses, and keep validation rules close to the HTTP layer where they belong.

## Testing Models

Model tests verify relationships, casts, accessors, and scopes — the data-layer logic that the model is responsible for. Business logic tests belong with the [Actions](/books/clean-code-in-laravel/actions) and [Services](/books/clean-code-in-laravel/organizing-your-application) that contain that logic.

### Testing Relationships

Verify that relationships are defined correctly by checking that they return the expected related models:

```php
it('belongs to a user', function (): void {
    $order = Order::factory()->for(User::factory())->create();

    expect($order->user)
        ->toBeInstanceOf(User::class);
});

it('has many items', function (): void {
    $order = Order::factory()
        ->has(OrderItem::factory()->count(3))
        ->create();

    expect($order->items)
        ->toHaveCount(3)
        ->each->toBeInstanceOf(OrderItem::class);
});
```

### Testing Casts

Verify that attributes are cast to the correct types:

```php
it('casts status to OrderStatus enum', function (): void {
    $order = Order::factory()->create(['status' => 'pending']);

    expect($order->status)->toBe(OrderStatus::Pending);
});

it('casts placed_at to a datetime', function (): void {
    $order = Order::factory()->create(['placed_at' => '2025-03-15 10:30:00']);

    expect($order->placed_at)->toBeInstanceOf(Carbon::class);
});
```

### Testing Accessors

```php
it('computes full name from first and last name', function (): void {
    $user = User::factory()->make([
        'first_name' => 'Ahmad',
        'last_name' => 'Mayahi',
    ]);

    expect($user->fullName)->toBe('Ahmad Mayahi');
});
```

### Testing Scopes

```php
it('filters active orders', function (): void {
    Order::factory()->create(['is_active' => true]);
    Order::factory()->create(['is_active' => false]);

    $results = Order::active()->get();

    expect($results)->toHaveCount(1)
        ->first()->is_active->toBeTrue();
});

it('filters orders by status', function (): void {
    Order::factory()->create(['status' => OrderStatus::Pending]);
    Order::factory()->create(['status' => OrderStatus::Shipped]);

    $results = Order::forStatus(OrderStatus::Pending)->get();

    expect($results)->toHaveCount(1)
        ->first()->status->toBe(OrderStatus::Pending);
});
```

### Using Factories Effectively

Always use [factories](https://laravel.com/docs/eloquent-factories) when creating models in tests. Factories produce valid model instances with sensible defaults, and their states let you express test scenarios clearly:

```php
// Define states in the factory
class OrderFactory extends Factory
{
    public function paid(): static
    {
        return $this->state(['status' => OrderStatus::Paid, 'paid_at' => now()]);
    }

    public function cancelled(): static
    {
        return $this->state(['status' => OrderStatus::Cancelled]);
    }

    public function withItems(int $count = 3): static
    {
        return $this->has(OrderItem::factory()->count($count));
    }
}

// Use states in tests — the intent is immediately clear
$order = Order::factory()->paid()->withItems(5)->create();
```

Factory states are reusable across your test suite. If the definition of "paid" changes — maybe it now requires a `payment_intent_id` — you update it in one place instead of hunting through dozens of tests.

## The Model Ordering Convention

A consistent internal structure makes models predictable. Here is the ordering convention used in this book:

```php
class Order extends Model
{
    // 1. Traits
    use HasFactory, SoftDeletes;

    // 2. Constants
    public const MAX_ITEMS = 50;

    // 3. Properties ($fillable, $guarded, $table, $with, etc.)
    protected $fillable = [/* ... */];

    // 4. Casts
    protected function casts(): array { /* ... */ }

    // 5. Relationships
    public function user(): BelongsTo { /* ... */ }
    public function items(): HasMany { /* ... */ }

    // 6. Accessors and Mutators
    protected function fullName(): Attribute { /* ... */ }

    // 7. Scopes
    public function scopeActive(Builder $query): void { /* ... */ }

    // 8. Model events
    protected static function booted(): void { /* ... */ }
}
```

This is not a Laravel requirement — it is a readability convention. When every model follows the same order, developers can navigate unfamiliar models quickly.

## The Model Checklist

1. **Use `casts()` method** — not the `$casts` property. Cast every column that has a type more specific than a raw string or integer
2. **Type all relationships** — `BelongsTo`, `HasMany`, `HasOne`, etc. with explicit return types
3. **Keep scopes focused** — one constraint per scope, compose them in queries. Promote to a [Custom Query Builder](/books/clean-code-in-laravel/custom-query-builders-and-collections) when the model has too many
4. **Use accessors for derived data** — `$order->isPaid`, `$user->fullName`. Not for business logic that queries other models or calls external services
5. **Model events for data integrity only** — generating IDs, cascading deletes, maintaining denormalized columns
6. **No business logic** — delegate to [Actions](/books/clean-code-in-laravel/actions) and [Services](/books/clean-code-in-laravel/organizing-your-application)
7. **No presentation logic** — delegate to [API Resources](https://laravel.com/docs/eloquent-resources) and [View Models](/books/clean-code-in-laravel/view-models)
8. **Use Concerns for cohesive behavior** — extract related methods into traits when they form a logical group
9. **Validate in Form Requests** — not in the model
10. **Use factories with expressive states** — make test setup readable and reusable

## Summary

- Eloquent models should focus on three things: data definition, relationships, and data access. Everything else — email sending, PDF generation, complex business logic — belongs in [Actions](/books/clean-code-in-laravel/actions), [Services](/books/clean-code-in-laravel/organizing-your-application), [Jobs](/books/clean-code-in-laravel/jobs), or dedicated classes.
- Cast every column to its correct type using the `casts()` method. Enums for statuses, `datetime` for dates, `boolean` for flags, custom casts for value objects. The cost is near zero, the benefit is type safety everywhere.
- Always type relationship return values. Use constrained relationships for domain concepts, the nullsafe operator for nullable relationships, and keep relationship methods pure — never add conditional logic based on runtime state.
- Accessors are for derived data that is a direct function of the model's own attributes. If an accessor queries the database, applies business rules, or calls external services, move that logic to a Service or Action.
- When you need to query, sort, or index a derived value, use a database generated column instead of an accessor. Generated columns are computed by the database and are visible to queries and indexes. Use `VIRTUAL` for computed-on-read values and `STORED` for indexable ones.
- Scopes should apply one constraint each and compose cleanly. When a model accumulates more than five or six scopes, extract them into a [Custom Query Builder](/books/clean-code-in-laravel/custom-query-builders-and-collections).
- Model events are for data integrity — generating defaults and cascading deletes. Observers are for side effects — notifications and syncs. Neither should contain business logic or orchestration.
- Both `$fillable` and `$guarded` are valid approaches to mass assignment protection. Choose one and be consistent. Either way, [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation) are your primary line of defense for input validation.
- Keep models lean by extracting behavior into Concerns, promoting scopes to query builders, and moving business logic to Actions. The model is not the right home for orchestration, external API calls, or presentation formatting.
- Test models at the data layer: relationships, casts, accessors, and scopes. Use factories with expressive states to make test setup readable and reusable.
- Follow a consistent internal ordering convention — traits, constants, properties, casts, relationships, accessors, scopes, events — so every model in the project is predictable.

## References

- [Eloquent: Getting Started](https://laravel.com/docs/eloquent) — Laravel Documentation
- [Eloquent: Relationships](https://laravel.com/docs/eloquent-relationships) — Laravel Documentation
- [Eloquent: Mutators & Casting](https://laravel.com/docs/eloquent-mutators) — Laravel Documentation
- [Eloquent: Factories](https://laravel.com/docs/eloquent-factories) — Laravel Documentation
- [20 Laravel Eloquent Tips and Tricks](https://laravel-news.com/eloquent-tips-tricks) — Laravel News
- [Effective Eloquent](https://laravel-news.com/effective-eloquent) — Laravel News
- [Dedicated Query Builders in Laravel](https://timacdonald.me/dedicated-eloquent-model-query-builders/) — Tim MacDonald
- [Giving Collections a Voice](https://timacdonald.me/giving-collections-a-voice/) — Tim MacDonald
- [Laravel Beyond CRUD](https://spatie.be/products/laravel-beyond-crud) — Brent Roose
- [Eloquent Performance Patterns](https://eloquent-course.reinink.ca/) — Jonathan Reinink
- [20 Tips to Optimize Your Eloquent Queries](https://martinjoo.dev/35-eloquent-recipes) — Martin Joo
- [MySQL Generated Columns in Laravel](/blog/mysql-generated-columns-in-laravel) — Ahmad Mayahi
- [MySQL Generated Columns](https://dev.mysql.com/doc/refman/8.4/en/create-table-generated-columns.html) — MySQL Documentation
