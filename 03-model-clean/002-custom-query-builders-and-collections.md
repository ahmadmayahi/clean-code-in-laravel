In the [previous chapter](/books/clean-code-in-laravel/eloquent-models-done-right), we used scopes to encapsulate common query constraints. A `scopeActive()` here, a `scopeForStatus()` there — clean and simple. But scopes do not scale.

Open a model that has been in production for two years. You will find `scopeActive`, `scopePending`, `scopeShipped`, `scopeForUser`, `scopePlacedBetween`, `scopePlacedToday`, `scopePlacedThisMonth`, `scopeHighValue`, `scopeSearch`, `scopeWithItemCount`, and a dozen more. Each scope is a few lines, but together they take up half the model. The model is supposed to define data — instead, it has become a query catalog.

The problem is not the scopes themselves. Each one is a reasonable encapsulation of a query constraint. The problem is *where* they live. Thirty scopes on a model means the model is doing two jobs: defining data and defining queries. These are different concerns that change for different reasons. A new filter on the orders page adds a scope. A new column on the orders table changes a cast. Both edit the same file, both create merge conflicts, and both make it harder to find what you are looking for.

Custom Query Builders solve this by extracting all query logic into a dedicated class. The model stays focused on data, and the builder owns the queries. Custom Collections do the same for in-memory operations — grouping, aggregation, and domain-specific filtering on data you have already fetched.

## Custom Query Builders

A Custom Query Builder is a class that extends `Illuminate\Database\Eloquent\Builder` and contains all the query methods for a specific model. Instead of `scopeActive()` on the model, you write `active()` on the builder — a regular method with a typed return, full IDE autocompletion, and no `scope` prefix magic.

Laravel 12 provides the `#[UseEloquentBuilder]` attribute to bind a builder to a model:

```php
namespace App\Builders;

use App\Enums\OrderStatus;
use Carbon\Carbon;
use Illuminate\Database\Eloquent\Builder;

class OrderBuilder extends Builder
{
    public function active(): self
    {
        return $this->where('is_active', true);
    }

    public function forStatus(OrderStatus $status): self
    {
        return $this->where('status', $status);
    }

    public function pending(): self
    {
        return $this->forStatus(OrderStatus::Pending);
    }

    public function shipped(): self
    {
        return $this->forStatus(OrderStatus::Shipped);
    }

    public function placedBetween(Carbon $from, Carbon $to): self
    {
        return $this->whereBetween('placed_at', [$from, $to]);
    }

    public function placedToday(): self
    {
        return $this->whereDate('placed_at', today());
    }

    public function placedThisMonth(): self
    {
        return $this->placedBetween(
            now()->startOfMonth(),
            now()->endOfMonth(),
        );
    }

    public function highValue(float $threshold = 100.00): self
    {
        return $this->where('total', '>=', $threshold);
    }

    public function forUser(int $userId): self
    {
        return $this->where('user_id', $userId);
    }

    public function withItemCount(): self
    {
        return $this->withCount('items');
    }

    public function search(string $term): self
    {
        return $this->where(function (Builder $query) use ($term): void {
            $query->where('order_number', 'like', "%{$term}%")
                ->orWhereHas('user', fn (Builder $q) => $q->where('name', 'like', "%{$term}%"));
        });
    }
}
```

Connect it to the model using the attribute:

```php
namespace App\Models;

use App\Builders\OrderBuilder;
use Illuminate\Database\Eloquent\Attributes\UseEloquentBuilder;
use Illuminate\Database\Eloquent\Model;

#[UseEloquentBuilder(OrderBuilder::class)]
class Order extends Model
{
    // The model is now clean — no scopes needed
}
```

Usage is identical to scopes, but with full IDE autocompletion and typed return values:

```php
$orders = Order::query()
    ->forUser($user->id)
    ->shipped()
    ->placedThisMonth()
    ->highValue()
    ->withItemCount()
    ->latest('placed_at')
    ->paginate(15);
```

Every method in `OrderBuilder` is a real method on a real class. Your IDE can autocomplete them, your static analyzer can verify them, and you can navigate to their definitions with a click. With scopes, `Order::active()` works through `__call` magic — your IDE cannot see it, and a typo like `Order::actvie()` fails silently at runtime instead of at analysis time.

### The `newEloquentBuilder` Approach

You can also override the `newEloquentBuilder` method on the model to bind a custom builder. This approach works in all Laravel versions:

```php
class Order extends Model
{
    public function newEloquentBuilder($query): OrderBuilder
    {
        return new OrderBuilder($query);
    }
}
```

Both approaches achieve the same result. The `#[UseEloquentBuilder]` attribute is simply cleaner — it expresses the binding declaratively, without overriding a method.

## Why Custom Query Builders Beat Scopes

| Aspect | Scopes | Custom Query Builder |
|---|---|---|
| Location | Inside the model | Separate class |
| IDE support | Limited (`__call` magic) | Full autocompletion |
| Static analysis | Weak | Strong (typed return) |
| Model cleanliness | Clutters the model | Keeps model focused |
| Reusability | Per-model only | Can share via traits or inheritance |
| Testability | Test through model | Test independently |
| Method prefix | Requires `scope` prefix | No prefix — regular methods |

The difference becomes dramatic as your application grows. A model with thirty scopes is painful to navigate — you scroll past walls of query logic to find a relationship or a cast. A Custom Query Builder with thirty methods is a well-organized query catalog that lives in its own file, with its own tests, and its own reason to change.

### When to Keep Scopes

Not every model needs a Custom Query Builder. If a model has two or three scopes, the overhead of a separate class is not justified — the scopes are fine where they are. Extract a builder when:

- The model has **more than four or five scopes** — the model file is getting long and scopes are drowning out the data definition.
- Multiple scopes **compose together frequently** — a builder can express this composition more fluently.
- You want **full IDE autocompletion** and static analysis on query methods.
- The query logic is **complex enough to test independently** — a builder gives you a clean test boundary.

## Builder Composition and Reuse

Builder methods compose naturally because each one returns `self`. You can chain them freely, and they combine into a single database query:

```php
Order::query()
    ->active()
    ->forUser($userId)
    ->placedThisMonth()
    ->highValue(500)
    ->latest('placed_at')
    ->get();
```

This produces one SQL query with all the constraints combined. The builder handles composition — you do not need to think about how the methods interact.

### Convenience Methods That Compose Existing Methods

Builders excel at creating named shortcuts for common query combinations. Notice how `pending()` and `shipped()` in the example above are just wrappers around `forStatus()`:

```php
public function pending(): self
{
    return $this->forStatus(OrderStatus::Pending);
}

public function shipped(): self
{
    return $this->forStatus(OrderStatus::Shipped);
}
```

These are not "combo scopes" that bundle unrelated conditions — they are named wrappers for a single concept. `Order::query()->pending()` reads better than `Order::query()->forStatus(OrderStatus::Pending)` and is less error-prone. This kind of composition is one of the main reasons to use a builder.

### Sharing Query Logic Between Builders

When multiple models share the same query patterns — `active()`, `search()`, `createdBetween()` — you can extract them into a trait:

```php
namespace App\Builders\Concerns;

trait HasActiveScope
{
    public function active(): self
    {
        return $this->where('is_active', true);
    }

    public function inactive(): self
    {
        return $this->where('is_active', false);
    }
}
```

```php
class OrderBuilder extends Builder
{
    use HasActiveScope;

    // Order-specific query methods...
}

class ProductBuilder extends Builder
{
    use HasActiveScope;

    // Product-specific query methods...
}
```

This is cleaner than duplicating `scopeActive()` on every model, and it makes the shared behavior explicit — when you update `HasActiveScope`, it affects all builders that use it.

## Custom Collections

Just as Custom Query Builders handle how you **query** data, Custom Collections handle how you **work with** the results. A Custom Collection is a class that extends `Illuminate\Database\Eloquent\Collection` and adds domain-specific methods for in-memory operations on data you have already fetched.

Laravel 12 provides the `#[CollectedBy]` attribute:

```php
namespace App\Collections;

use App\Enums\OrderStatus;
use App\Models\Order;
use Illuminate\Database\Eloquent\Collection;

class OrderCollection extends Collection
{
    public function totalRevenue(): float
    {
        return $this->sum('total');
    }

    public function averageOrderValue(): float
    {
        return $this->avg('total') ?? 0;
    }

    public function pending(): self
    {
        return $this->where('status', OrderStatus::Pending);
    }

    public function shipped(): self
    {
        return $this->where('status', OrderStatus::Shipped);
    }

    public function cancelled(): self
    {
        return $this->where('status', OrderStatus::Cancelled);
    }

    public function highValue(float $threshold = 100.00): self
    {
        return $this->where('total', '>=', $threshold);
    }

    public function placedBy(int $userId): self
    {
        return $this->where('user_id', $userId);
    }

    public function groupByStatus(): self
    {
        return $this->groupBy(fn (Order $order): string => $order->status->value);
    }

    public function sortByNewest(): self
    {
        return $this->sortByDesc('placed_at');
    }
}
```

Connect it to the model:

```php
namespace App\Models;

use App\Builders\OrderBuilder;
use App\Collections\OrderCollection;
use Illuminate\Database\Eloquent\Attributes\CollectedBy;
use Illuminate\Database\Eloquent\Attributes\UseEloquentBuilder;
use Illuminate\Database\Eloquent\Model;

#[UseEloquentBuilder(OrderBuilder::class)]
#[CollectedBy(OrderCollection::class)]
class Order extends Model
{
    // Clean model — query logic in OrderBuilder, collection logic in OrderCollection
}
```

Now you can chain domain-specific methods on any collection of orders:

```php
$orders = Order::query()->forUser($user->id)->get();

$revenue = $orders->totalRevenue();
$pendingCount = $orders->pending()->count();
$avgValue = $orders->highValue()->averageOrderValue();
```

Without a custom collection, this code would be scattered across controllers and views — `$orders->sum('total')` here, `$orders->where('status', OrderStatus::Pending)->count()` there. The custom collection gives these operations a name, makes them discoverable, and ensures they are consistent wherever they are used.

### The `newCollection` Approach

Override `newCollection` on the model if you prefer not to use the attribute:

```php
class Order extends Model
{
    public function newCollection(array $models = []): OrderCollection
    {
        return new OrderCollection($models);
    }
}
```

### When to Create a Custom Collection

Not every model needs a custom collection. The base `Collection` class is already powerful, and adding a custom class for a few `sum()` and `filter()` calls creates unnecessary files. Create a custom collection when:

- You have **domain-specific operations** that are used in multiple places — `totalRevenue()`, `averageOrderValue()`, `groupByStatus()`.
- You want to give **names to common filter/aggregation patterns** — `$orders->pending()` is more readable than `$orders->where('status', OrderStatus::Pending)`.
- The collection operations are **complex enough to test independently** — a method that calculates weighted averages, applies business rules, or transforms data in a non-trivial way.

If a model only needs basic collection operations (`sum`, `filter`, `map`, `pluck`), the base collection is fine. Do not create a custom collection just to wrap existing methods with slightly different names.

## Query Builder vs. Collection: Where to Filter

A common mistake is filtering in the Collection when you should filter in the Query Builder:

```php
// Bad: fetches ALL orders, then filters in PHP
$shipped = Order::all()->where('status', OrderStatus::Shipped);

// Good: filters at the database level
$shipped = Order::query()->shipped()->get();
```

The first line fetches every order from the database into memory, hydrates them as Eloquent model instances, and *then* filters. If you have 100,000 orders and 500 of them are shipped, you loaded 99,500 rows for nothing. The second line lets the database do the filtering — it only returns the 500 rows you actually need.

The rule: **filter in the Query Builder** (database level) whenever possible. Use Collection methods for in-memory operations on data you have already fetched:

```php
// Good: fetch once, then work with the collection
$orders = Order::query()
    ->forUser($user->id)
    ->placedThisMonth()
    ->get();

// Now use collection methods for in-memory grouping and calculations
$byStatus = $orders->groupByStatus();
$revenue = $orders->totalRevenue();
$avgValue = $orders->averageOrderValue();
```

The query builder narrows the result set. The collection processes it. Together, they form a clear pipeline: the database does the heavy lifting, and PHP handles the presentation.

### Methods That Exist on Both

Some method names appear on both the builder and the collection — `where()`, `count()`, `first()`. This can cause confusion:

```php
// This calls the BUILDER method — runs a SQL query
$pending = Order::query()->where('status', 'pending')->get();

// This calls the COLLECTION method — filters in PHP
$pending = $orders->where('status', 'pending');
```

The distinction matters for performance. `Order::query()->where(...)` adds a `WHERE` clause to the SQL query — the database filters. `$orders->where(...)` iterates over a PHP array — memory and CPU do the work. For large datasets, the difference is orders of magnitude.

When you have both a builder method and a collection method with the same name (like `pending()`), the context makes it clear which one is called:

```php
// Builder — this is a query, returns a builder
Order::query()->pending()->get();

// Collection — this is in-memory filtering, returns a collection
$orders->pending();
```

## Testing Custom Query Builders

Builder methods produce SQL queries. You can test them by calling the method and asserting on the resulting data — no need to test the SQL itself:

```php
it('filters orders by status', function (): void {
    Order::factory()->create(['status' => OrderStatus::Pending]);
    Order::factory()->create(['status' => OrderStatus::Shipped]);
    Order::factory()->create(['status' => OrderStatus::Pending]);

    $results = Order::query()->pending()->get();

    expect($results)
        ->toHaveCount(2)
        ->each(fn ($order) => $order->status->toBe(OrderStatus::Pending));
});

it('filters orders by date range', function (): void {
    Order::factory()->create(['placed_at' => now()->subDays(5)]);
    Order::factory()->create(['placed_at' => now()->subDays(15)]);
    Order::factory()->create(['placed_at' => now()->subDays(45)]);

    $results = Order::query()
        ->placedBetween(now()->subDays(30), now())
        ->get();

    expect($results)->toHaveCount(2);
});

it('searches orders by order number or user name', function (): void {
    $user = User::factory()->create(['name' => 'Ahmad Mayahi']);
    Order::factory()->for($user)->create(['order_number' => 'ORD-ABC123']);
    Order::factory()->create(['order_number' => 'ORD-XYZ789']);

    expect(Order::query()->search('ABC')->get())->toHaveCount(1)
        ->and(Order::query()->search('Ahmad')->get())->toHaveCount(1)
        ->and(Order::query()->search('nonexistent')->get())->toHaveCount(0);
});

it('composes multiple builder methods into a single query', function (): void {
    Order::factory()->create([
        'is_active' => true,
        'status' => OrderStatus::Shipped,
        'total' => 250,
        'placed_at' => now(),
    ]);

    Order::factory()->create([
        'is_active' => true,
        'status' => OrderStatus::Shipped,
        'total' => 50, // Below threshold
        'placed_at' => now(),
    ]);

    $results = Order::query()
        ->active()
        ->shipped()
        ->highValue(100)
        ->placedThisMonth()
        ->get();

    expect($results)->toHaveCount(1);
});
```

Test builder methods through actual database queries — not by asserting on generated SQL strings. The SQL is an implementation detail; the result set is what matters.

## Testing Custom Collections

Collection tests do not need the database. Create model instances with `factory()->make()` (in-memory, no persistence) and test the collection methods directly:

```php
it('calculates total revenue', function (): void {
    $orders = new OrderCollection([
        Order::factory()->make(['total' => 100.00]),
        Order::factory()->make(['total' => 250.00]),
        Order::factory()->make(['total' => 50.00]),
    ]);

    expect($orders->totalRevenue())->toBe(400.00);
});

it('calculates average order value', function (): void {
    $orders = new OrderCollection([
        Order::factory()->make(['total' => 100.00]),
        Order::factory()->make(['total' => 200.00]),
    ]);

    expect($orders->averageOrderValue())->toBe(150.00);
});

it('returns zero average for empty collection', function (): void {
    $orders = new OrderCollection([]);

    expect($orders->averageOrderValue())->toBe(0.0);
});

it('filters pending orders', function (): void {
    $orders = new OrderCollection([
        Order::factory()->make(['status' => OrderStatus::Pending]),
        Order::factory()->make(['status' => OrderStatus::Shipped]),
        Order::factory()->make(['status' => OrderStatus::Pending]),
    ]);

    expect($orders->pending())->toHaveCount(2);
});

it('groups orders by status', function (): void {
    $orders = new OrderCollection([
        Order::factory()->make(['status' => OrderStatus::Pending]),
        Order::factory()->make(['status' => OrderStatus::Shipped]),
        Order::factory()->make(['status' => OrderStatus::Pending]),
    ]);

    $grouped = $orders->groupByStatus();

    expect($grouped)->toHaveCount(2)
        ->and($grouped['pending'])->toHaveCount(2)
        ->and($grouped['shipped'])->toHaveCount(1);
});
```

Collection tests are fast — no database, no HTTP, no faking. This is one of the main advantages of extracting collection logic into a dedicated class. The methods are independently testable with exact control over the input data.

## A Complete Example

Here is how a model, its Query Builder, and its Collection work together in a controller:

```php
class OrderController extends Controller
{
    public function index(Request $request): View
    {
        $orders = Order::query()
            ->forUser($request->user()->id)
            ->when($request->status, fn ($q, $status) => $q->forStatus(OrderStatus::from($status)))
            ->when($request->search, fn ($q, $term) => $q->search($term))
            ->withItemCount()
            ->latest('placed_at')
            ->paginate(15);

        return view('orders.index', [
            'orders' => $orders,
            'stats' => [
                'total_revenue' => $orders->getCollection()->totalRevenue(),
                'average_value' => $orders->getCollection()->averageOrderValue(),
            ],
        ]);
    }
}
```

The Query Builder handles the database filtering and pagination. The Collection handles the in-memory calculations on the current page of results. The model stays clean — it defines data, relationships, and casts. Each class has one job.

Notice `$orders->getCollection()` — when you paginate, Laravel returns a `LengthAwarePaginator`, not a collection. Call `getCollection()` to access the underlying collection with your custom methods.

## Common Mistakes

### Filtering in the Collection When You Should Use the Builder

```php
// Bad: loads all orders, filters in PHP
$highValue = Order::all()->where('total', '>=', 100);

// Good: filters at the database level
$highValue = Order::query()->highValue()->get();
```

If you have a builder method for it, use the builder. Collection filtering is for operations on data you *already* need in memory.

### Duplicating Logic Between Builder and Collection

If both `OrderBuilder` and `OrderCollection` have a `pending()` method, they serve different purposes — the builder adds a SQL `WHERE` clause, the collection filters an in-memory array. But if you find yourself writing the *same business rule* in both places, you have a consistency problem. When the definition of "pending" changes, you need to update two classes.

The fix: query at the builder level whenever possible. Only add collection-level filtering when you are working with a pre-fetched collection that you need to break down further — like showing "pending" and "shipped" tabs from a single query.

### Making Builder Methods Too Granular

```php
// Too granular — these are just raw where clauses with names
public function whereUserIdIs(int $userId): self
{
    return $this->where('user_id', $userId);
}

public function whereStatusIs(string $status): self
{
    return $this->where('status', $status);
}
```

Builder methods should represent meaningful domain concepts. `forUser()` is borderline acceptable because it expresses intent. `whereUserIdIs()` is just `where()` with a longer name — it adds a method without adding meaning. If a builder method does not express a concept that someone would use in a conversation about the domain, it probably should not exist.

### Returning the Wrong Type

Builder methods should return `self` so they can be chained. A common mistake is returning `void` or `Builder` (the parent class) instead:

```php
// Bad: returns void — cannot chain
public function active(): void
{
    $this->where('is_active', true);
}

// Bad: returns parent type — loses IDE autocompletion for custom methods
public function active(): Builder
{
    return $this->where('is_active', true);
}

// Good: returns self — chainable with full IDE support
public function active(): self
{
    return $this->where('is_active', true);
}
```

Returning `self` ensures that `Order::query()->active()->shipped()` chains correctly *and* your IDE knows that `shipped()` is available after `active()`.

## Organizing Builders and Collections

```
app/
├── Builders/
│   ├── OrderBuilder.php
│   ├── UserBuilder.php
│   └── ProductBuilder.php
├── Collections/
│   ├── OrderCollection.php
│   ├── UserCollection.php
│   └── ProductCollection.php
└── Models/
    ├── Order.php
    ├── User.php
    └── Product.php
```

Keep builders and collections in their own directories at the app level. This mirrors how Laravel organizes its own code — models in one place, query logic in another, collection behavior in a third.

For a domain-oriented perspective on custom query builders and collections — where they live inside domain folders alongside models and states — see [Eloquent Without the Bloat](/books/thinking-in-domains-in-laravel/eloquent-without-the-bloat) in *Thinking in Domains in Laravel*.

## The Checklist

1. **Extract to a Custom Query Builder** when a model has more than four or five scopes — keep the model focused on data
2. **Use `#[UseEloquentBuilder]`** for clean, declarative model-builder binding
3. **Use `#[CollectedBy]`** for custom collections with domain-specific methods
4. **Return `self`** from builder and collection methods for fluent chaining and IDE support
5. **Filter at the database level** — use Query Builder methods, not Collection filtering, for large datasets
6. **Collection methods for in-memory work** — grouping, aggregation, and transformation on already-fetched data
7. **Name methods after domain concepts** — `highValue()` and `pending()`, not `whereStatusIs()` and `whereTotalGreaterThan()`
8. **Share common query patterns** via traits on builders — `HasActiveScope`, `Searchable`
9. **Test builders with real queries** — assert on result sets, not SQL strings
10. **Test collections without the database** — use `factory()->make()` for fast, focused tests

## Summary

- Custom Query Builders extract query logic from models into dedicated classes. The model defines data; the builder owns the queries. This separation keeps both focused and independently testable.
- Use the `#[UseEloquentBuilder]` attribute to bind a builder to a model, or override `newEloquentBuilder()` for pre-Laravel 12 projects. Both achieve the same result.
- Builder methods are real methods on a real class — full IDE autocompletion, typed returns, and static analysis support. Scopes rely on `__call` magic, which limits tooling and silently swallows typos.
- Extract a builder when a model has more than four or five scopes. For two or three scopes, the overhead of a separate class is not justified.
- Builder methods should return `self` for fluent chaining, represent meaningful domain concepts, and compose naturally — `active()->shipped()->highValue()` produces a single SQL query.
- Share common query patterns between builders using traits. `HasActiveScope` on both `OrderBuilder` and `ProductBuilder` is cleaner than duplicating `scopeActive()` on both models.
- Custom Collections add domain-specific methods for in-memory operations — `totalRevenue()`, `averageOrderValue()`, `groupByStatus()`. Use the `#[CollectedBy]` attribute to bind a collection to a model.
- Create a custom collection only when you have domain operations used in multiple places or complex enough to test independently. The base `Collection` is sufficient for simple `sum`, `filter`, and `map` calls.
- Always filter at the database level (Query Builder) when possible. Collection filtering loads all rows into memory, then discards most of them. For large datasets, the performance difference is orders of magnitude.
- Test builder methods through actual database queries — assert on result sets, not generated SQL. Test collection methods without the database using `factory()->make()` for fast, focused assertions.
- Use `$paginator->getCollection()` to access custom collection methods on paginated results.

## References

- [Eloquent: Getting Started](https://laravel.com/docs/eloquent) — Laravel Documentation
- [Eloquent: Collections](https://laravel.com/docs/eloquent-collections) — Laravel Documentation
- [Collections](https://laravel.com/docs/collections) — Laravel Documentation
- [Dedicated Query Builders in Laravel](https://timacdonald.me/dedicated-eloquent-model-query-builders/) — Tim MacDonald
- [Giving Collections a Voice](https://timacdonald.me/giving-collections-a-voice/) — Tim MacDonald
- [Laravel Beyond CRUD: Models](https://stitcher.io/blog/laravel-beyond-crud-04-models) — Brent Roose
- [Custom Eloquent Builders in Laravel](https://laravel-news.com/custom-eloquent-builders) — Laravel News
- [Eloquent Performance Patterns](https://eloquent-course.reinink.ca/) — Jonathan Reinink
