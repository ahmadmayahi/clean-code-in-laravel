An API is a contract between your application and its consumers. Whether those consumers are a mobile app, a JavaScript frontend, or a third-party integration, they depend on the API being predictable and consistent. Break that contract — change a field name, alter an error format, return unexpected data — and every consumer breaks with it.

Clean API design in Laravel means never exposing your database structure directly, giving consumers flexibility within strict boundaries, and ensuring that every response — success or error — follows the same predictable shape.

## API Resources

Returning Eloquent models directly from API endpoints is tempting but dangerous. Models expose your database structure — column names, hidden fields, timestamps, relationship loading — to the outside world. Rename a database column and every consumer breaks. Add an internal field and it leaks into every response. Your database schema becomes your API contract, and database schemas change far more often than API contracts should.

An [API Resource](https://laravel.com/docs/eloquent-resources) solves this by acting as a transformation layer between your [Eloquent models](/books/clean-code-in-laravel/eloquent-models-done-right) and the JSON your API returns. It defines exactly which fields to expose, how to format them, and which relationships to include — so your API contract stays stable even when your database schema changes.

Generate one with Artisan:

```bash
php artisan make:resource OrderResource
```

This creates a class in `app/Http/Resources/` with a single `toArray()` method where you define the response shape:

```php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'status' => $this->status->value,
            'subtotal' => $this->subtotal,
            'tax' => $this->tax,
            'total' => $this->total,
            'item_count' => $this->items->count(),
            'placed_at' => $this->created_at->toIso8601String(),
            'customer' => new UserResource($this->whenLoaded('user')),
            'items' => OrderItemResource::collection($this->whenLoaded('items')),
            'shipping_address' => new AddressResource($this->whenLoaded('shippingAddress')),
        ];
    }
}
```

Key patterns to notice:

- **`whenLoaded()`** — only includes relationships that were eager-loaded, preventing [N+1 queries](/books/clean-code-in-laravel/database-best-practices)
- **Explicit field selection** — only the fields the consumer needs, not the entire model
- **Formatted dates** — ISO 8601 strings, not Carbon objects
- **Nested resources** — `UserResource`, `OrderItemResource` for consistent formatting at every level

### Resource Collections

Returning an entire table's worth of data in a single response is a performance and security risk — consumers receive thousands of records they never asked for, and your server pays the cost of loading them all. Always paginate list endpoints, and use resource collections to format each item consistently:

```php
// In the controller
public function index(Request $request): AnonymousResourceCollection
{
    $orders = Order::where('user_id', $request->user()->id)
        ->with(['items.product', 'shippingAddress'])
        ->latest()
        ->paginate(15);

    return OrderResource::collection($orders);
}
```

Laravel automatically wraps the collection in a `data` key and includes pagination metadata:

```json
{
    "data": [
        { "id": 1, "status": "shipped", "total": 99.99 },
        { "id": 2, "status": "pending", "total": 45.50 }
    ],
    "links": {
        "first": "https://example.com/api/orders?page=1",
        "last": "https://example.com/api/orders?page=5",
        "next": "https://example.com/api/orders?page=2"
    },
    "meta": {
        "current_page": 1,
        "per_page": 15,
        "total": 73
    }
}
```

### The Trade-Off: Magic Properties

API Resources are a significant improvement over returning raw models, but they carry a trade-off worth understanding. Resources extend `JsonResource`, which uses the same `__get()` magic as Eloquent. Inside a resource's `toArray()`, `$this->tyypoedProperty` silently returns `null` instead of throwing an error. A misspelled field name in your resource class produces a `null` in your API response, and nothing warns you — no exception, no static analysis error, no test failure unless you explicitly assert against `null` values.

This is the same class of problem that makes Eloquent models difficult to work with at scale: magic property access trades compile-time safety for runtime convenience. For most applications, API Resources are the pragmatic choice — they integrate cleanly with Laravel's pagination and response system, and the `whenLoaded()` helper is genuinely useful.

But if your API is large or consumed by external teams, consider using a [DTO](/books/clean-code-in-laravel/data-transfer-objects) library instead. A DTO with typed properties will throw an error when you access a property that does not exist. Controllers can return DTOs and accept them as arguments, and the type system catches mistakes that magic properties silently swallow. Libraries like [spatie/laravel-data](https://spatie.be/docs/laravel-data) can act as both DTOs and API responses, giving you the transformation layer of a resource without the magic property footgun.

## Flexible Filtering and Sorting

API consumers need flexibility. A mobile app wants orders sorted by date. A dashboard wants them filtered by status. An admin panel needs both, plus a search on customer name. Handling this manually means a growing chain of conditionals in your controller:

```php
// This gets out of hand fast
$query = Order::query();

if ($request->has('status')) {
    $query->where('status', $request->input('status'));
}

if ($request->has('sort')) {
    $query->orderBy($request->input('sort'), $request->input('direction', 'asc'));
}

if ($request->has('include') && $request->input('include') === 'items') {
    $query->with('items');
}

// And on and on for every parameter...
```

Every new filter or sort option adds another block. Every parameter needs manual validation. And nothing stops a consumer from sorting by a column you did not intend to expose — which is both a data leak and a potential security risk.

[`spatie/laravel-query-builder`](https://spatie.be/docs/laravel-query-builder) solves this with a declarative approach. You list what is allowed — filters, sorts, includes — and the package handles the query string parsing, validation, and query building:

```bash
composer require spatie/laravel-query-builder
```

```php
use Spatie\QueryBuilder\QueryBuilder;
use Spatie\QueryBuilder\AllowedFilter;
use Spatie\QueryBuilder\AllowedSort;

class OrderController extends Controller
{
    public function index(Request $request): AnonymousResourceCollection
    {
        $orders = QueryBuilder::for(Order::class)
            ->allowedFilters([
                AllowedFilter::exact('status'),
                AllowedFilter::scope('placed_between'),
                AllowedFilter::exact('user_id'),
                AllowedFilter::partial('customer_name', 'user.name'),
            ])
            ->allowedSorts([
                AllowedSort::field('total'),
                AllowedSort::field('placed_at', 'created_at'),
                AllowedSort::field('status'),
            ])
            ->allowedIncludes([
                'items',
                'items.product',
                'user',
                'shippingAddress',
            ])
            ->defaultSort('-created_at')
            ->where('user_id', $request->user()->id)
            ->paginate(15);

        return OrderResource::collection($orders);
    }
}
```

Now consumers can make requests like:

```
GET /api/orders?filter[status]=shipped&sort=-total&include=items.product
GET /api/orders?filter[placed_between]=2025-01-01,2025-12-31&sort=placed_at
GET /api/orders?filter[customer_name]=John&include=user
```

The beauty is that consumers get flexibility, but you control exactly which filters, sorts, and includes are allowed. Unauthorized parameters are silently ignored.

## Custom Filters

The built-in filter types — `exact()`, `partial()`, `scope()` — cover most cases. But sometimes a filter needs logic that does not fit into a single column comparison. Filtering orders by a price range (minimum and maximum) requires parsing a comma-separated value and applying two `where` clauses. Putting that logic inline would defeat the purpose of the declarative approach.

For these cases, create custom filter classes. If you are already using [Custom Query Builders](/books/clean-code-in-laravel/custom-query-builders-and-collections), Spatie Query Builder works seamlessly with scopes defined there:

```php
namespace App\Filters;

use Spatie\QueryBuilder\Filters\Filter;
use Illuminate\Database\Eloquent\Builder;

class OrderTotalRangeFilter implements Filter
{
    public function __invoke(Builder $query, mixed $value, string $property): void
    {
        $range = explode(',', $value);

        $query->when(
            isset($range[0]) && $range[0] !== '',
            fn (Builder $q): Builder => $q->where('total', '>=', $range[0]),
        )->when(
            isset($range[1]) && $range[1] !== '',
            fn (Builder $q): Builder => $q->where('total', '<=', $range[1]),
        );
    }
}

// Usage
AllowedFilter::custom('total_range', new OrderTotalRangeFilter()),

// Request: GET /api/orders?filter[total_range]=50,200
```

## Consistent Error Responses

When error responses have no consistent structure, every consumer has to handle errors differently depending on whether it is a validation failure, an authentication error, or a server exception. A mobile developer might receive `{"message": "Unauthenticated."}` for one error and `{"error": "Validation failed", "fields": {...}}` for another. This forces brittle, special-case error handling on every consumer.

A consistent error format means every error follows the same shape — a `message` field, an optional `errors` object with field-specific details, and a proper HTTP status code. Laravel handles this well out of the box, but only when it knows the consumer expects JSON. By default, if an exception is thrown during an API request, Laravel may render an HTML error page — which is useless for an API consumer expecting a structured JSON response.

To fix this, tell Laravel when to render exceptions as JSON in `bootstrap/app.php`:

```php
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Http\Request;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e): bool {
        return $request->is('api/*') || $request->expectsJson();
    });
})
```

The `shouldRenderJsonWhen` method accepts a closure that receives the current request and returns `true` when exceptions should be rendered as JSON instead of HTML. Here, it triggers for any request whose URL starts with `api/` or whose `Accept` header includes `application/json`. With this in place, every exception — whether it is a validation error, an authentication failure, or an unhandled server error — returns a structured JSON response with the appropriate HTTP status code.

For custom exceptions, implement `render()`:

```php
class InsufficientStockException extends Exception
{
    public function __construct(
        private readonly Product $product,
        private readonly int $requested,
    ) {
        parent::__construct("Insufficient stock for {$product->name}");
    }

    public function render(Request $request): JsonResponse
    {
        return response()->json([
            'message' => $this->getMessage(),
            'errors' => [
                'quantity' => ["Only {$this->product->stock} available, {$this->requested} requested."],
            ],
        ], Response::HTTP_UNPROCESSABLE_ENTITY);
    }
}
```

## API Versioning

APIs change. You add fields, rename endpoints, or restructure responses. Without versioning, every change is a breaking change — the mobile app release from six months ago stops working the moment you rename a field. Consumers lose trust when their integrations break without warning.

Versioning gives you a path to evolve without breaking existing consumers. The simplest approach in Laravel is route prefixing with version-specific [controllers](/books/clean-code-in-laravel/controllers) and resources:

```php
// routes/api.php
Route::prefix('v1')->group(function (): void {
    Route::apiResource('orders', Api\V1\OrderController::class);
});

Route::prefix('v2')->group(function (): void {
    Route::apiResource('orders', Api\V2\OrderController::class);
});
```

Each version has its own controllers and resources. When v2 needs a different response shape, you create `V2\OrderResource` without touching v1. Consumers migrate on their own schedule, and you deprecate old versions when adoption drops.

## The API Checklist

1. **Never return raw models** — use [API Resources](https://laravel.com/docs/eloquent-resources) for every response
2. **Use `whenLoaded()`** — prevent accidental [N+1 queries](/books/clean-code-in-laravel/database-best-practices) in resources
3. **Use Spatie Query Builder** — give consumers flexibility within your rules
4. **Paginate everything** — never return unbounded collections
5. **Consistent error format** — same structure for [validation](/books/clean-code-in-laravel/form-requests-and-validation), auth, and server errors
6. **Version your API** — plan for evolution from day one
7. **Use proper HTTP status codes** — 201 for created, 204 for no content, 422 for validation errors

## Summary

- An API is a contract. Every response — success or error — should follow a predictable, consistent shape that consumers can rely on without reading your source code.
- Never return Eloquent models directly from API endpoints. Use [API Resources](https://laravel.com/docs/eloquent-resources) or [DTOs](/books/clean-code-in-laravel/data-transfer-objects) to define exactly which fields to expose, how to format them, and which relationships to include. This decouples your API contract from your database schema. Resources integrate well with Laravel's pagination, but their magic property access means typos silently return `null`. For larger APIs, a typed DTO library like [spatie/laravel-data](https://spatie.be/docs/laravel-data) catches these mistakes at development time.
- Use `whenLoaded()` in resources to include relationships only when they were eager-loaded, preventing [N+1 queries](/books/clean-code-in-laravel/database-best-practices).
- Always paginate list endpoints. Returning unbounded collections is a performance and security risk.
- [`spatie/laravel-query-builder`](https://spatie.be/docs/laravel-query-builder) provides declarative filtering, sorting, and relationship inclusion. Consumers get flexibility, but you control exactly which parameters are allowed — unauthorized parameters are silently ignored.
- Create custom filter classes for complex filtering logic that does not fit into a single column comparison.
- Standardize error responses in `bootstrap/app.php` so every error — validation, authentication, server — follows the same structure with a `message` field, an optional `errors` object, and a proper HTTP status code.
- Version your API from day one using route prefixing and version-specific controllers and resources. This lets you evolve the API without breaking existing consumers.

## References

- [Eloquent: API Resources](https://laravel.com/docs/eloquent-resources) — Laravel Documentation
- [Controllers: API Resource Controllers](https://laravel.com/docs/controllers#api-resource-controllers) — Laravel Documentation
- [Error Handling](https://laravel.com/docs/errors) — Laravel Documentation
- [HTTP Responses: JSON Responses](https://laravel.com/docs/responses#json-responses) — Laravel Documentation
- [spatie/laravel-query-builder](https://spatie.be/docs/laravel-query-builder/v6/introduction) — Spatie Documentation
- [Filtering](https://spatie.be/docs/laravel-query-builder/v6/features/filtering) — Spatie Query Builder Documentation
- [Sorting](https://spatie.be/docs/laravel-query-builder/v6/features/sorting) — Spatie Query Builder Documentation
- [Including Relationships](https://spatie.be/docs/laravel-query-builder/v6/features/including-relationships) — Spatie Query Builder Documentation
- [Build an API with Laravel Resources](https://laravel-news.com/build-an-api-with-laravel-resources) — Laravel News
- [Effective API Resource Responses in Laravel](https://martinjoo.dev/laravel-api-resources) — Martin Joo
- [API Versioning in Laravel](https://laravel-news.com/api-versioning-in-laravel) — Laravel News
- [Best Practices for Designing a Pragmatic RESTful API](https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api) — Vinay Sahni
