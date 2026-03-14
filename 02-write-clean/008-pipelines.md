If you have ever used [middleware](https://laravel.com/docs/middleware) in Laravel, you have already used a Pipeline — Laravel's HTTP middleware system is built on `Illuminate\Pipeline\Pipeline`. A Pipeline passes data through a series of steps, where each step can modify the data and decide whether to pass it to the next one.

But Pipelines are not just for HTTP. They are a general-purpose tool for any multi-step process where the steps might change, be reordered, or be conditionally applied. The key word is *might*. If your process is fixed and simple, sequential code is fine. Pipelines earn their keep when the steps are dynamic, independently testable, or reusable across different contexts.

## The Problem Pipelines Solve

Consider an e-commerce checkout. When a customer places an order, you need to calculate the subtotal, apply a coupon if present, calculate tax based on the shipping address, calculate shipping costs (but only for physical products), check inventory, create the order record, and reserve stock. Here is what that looks like as sequential code in an [Action](/books/clean-code-in-laravel/actions):

```php
// Everything in one method
public function execute(PlaceOrderData $input): Order
{
    $subtotal = $input->items->sum(
        fn (OrderItemData $item): float => $item->unitPrice * $item->quantity,
    );

    $discount = 0;
    if ($input->couponCode) {
        $coupon = Coupon::where('code', $input->couponCode)->first();
        if ($coupon?->isValid()) {
            $discount = $subtotal * ($coupon->percentage / 100);
        }
    }

    $taxableAmount = $subtotal - $discount;
    $tax = $this->taxCalculator->calculate($taxableAmount, $input->country);

    $shipping = 0;
    if ($input->requiresShipping) {
        $shipping = $this->shippingCalculator->calculate($input->weight, $input->country);
    }

    $total = $taxableAmount + $tax + $shipping;

    foreach ($input->items as $item) {
        $product = Product::find($item->productId);
        if ($product->stock < $item->quantity) {
            throw new InsufficientStockException($product);
        }
    }

    $order = Order::create([
        'user_id' => $input->userId,
        'subtotal' => $subtotal,
        'discount' => $discount,
        'tax' => $tax,
        'shipping' => $shipping,
        'total' => $total,
        'status' => OrderStatus::Pending,
    ]);

    foreach ($input->items as $item) {
        Product::find($item->productId)->decrement('stock', $item->quantity);
    }

    return $order;
}
```

This method does seven different things. Testing the tax calculation means running the entire checkout. Adding a loyalty points step means editing this method. Removing the coupon logic for a specific storefront means wrapping it in a conditional. Every change risks breaking something else.

The deeper problem is that this code cannot be reconfigured. A B2B storefront might skip coupons entirely. A digital product checkout has no shipping step. A wholesale order might use a different tax calculator. With sequential code, you handle these variations with `if` statements that multiply until the method is unreadable.

## Building a Pipeline

With `Illuminate\Pipeline\Pipeline`, each step becomes its own class. The [Action](/books/clean-code-in-laravel/actions) declares the steps, and the Pipeline runs them in order:

```php
use Illuminate\Pipeline\Pipeline;

class PlaceOrderAction
{
    public function execute(PlaceOrderData $input): Order
    {
        $pipelineData = new OrderPipelineData($input);

        $result = app(Pipeline::class)
            ->send($pipelineData)
            ->through([
                CalculateSubtotalStep::class,
                ApplyCouponStep::class,
                CalculateTaxStep::class,
                CalculateShippingStep::class,
                CheckInventoryStep::class,
                CreateOrderStep::class,
                ReserveInventoryStep::class,
            ])
            ->thenReturn();

        return $result->order;
    }
}
```

Each step is a class with a `handle` method that receives the data and a `$next` closure:

```php
namespace App\Pipelines\Order;

use Closure;

class CalculateSubtotalStep
{
    public function handle(OrderPipelineData $data, Closure $next): mixed
    {
        $data->subtotal = $data->input->items->sum(
            fn (OrderItemData $item): float => $item->unitPrice * $item->quantity,
        );

        return $next($data);
    }
}
```

```php
class ApplyCouponStep
{
    public function handle(OrderPipelineData $data, Closure $next): mixed
    {
        if ($data->input->couponCode) {
            $coupon = Coupon::where('code', $data->input->couponCode)->first();

            if ($coupon?->isValid()) {
                $data->discount = $data->subtotal * ($coupon->percentage / 100);
            }
        }

        return $next($data);
    }
}
```

```php
class CalculateTaxStep
{
    public function __construct(
        private readonly TaxCalculatorService $taxCalculator,
    ) {}

    public function handle(OrderPipelineData $data, Closure $next): mixed
    {
        $taxableAmount = $data->subtotal - $data->discount;
        $data->tax = $this->taxCalculator->calculate($taxableAmount, $data->input->country);
        $data->total = $taxableAmount + $data->tax;

        return $next($data);
    }
}
```

Now testing tax calculation is a unit test — instantiate `CalculateTaxStep`, pass a prepared `OrderPipelineData`, and assert the result. No HTTP request, no database, no checkout flow. And a B2B storefront simply removes `ApplyCouponStep` from the array.

## Using a Passable Object

In the checkout example above, `OrderPipelineData` is not just a [DTO](/books/clean-code-in-laravel/data-transfer-objects) — it is a "passable" object that accumulates state as it moves through the pipeline. The subtotal step calculates a number that the tax step needs. The coupon step produces a discount that affects the total. If you passed the original `PlaceOrderData` DTO directly, each step would have nowhere to store its intermediate results for the next step to read.

A passable object solves this by combining the original input with mutable properties for each step to populate:

```php
namespace App\Pipelines\Order;

use App\Models\Order;
use App\DataTransferObjects\PlaceOrderData;

class OrderPipelineData
{
    public ?Order $order = null;
    public float $subtotal = 0;
    public float $tax = 0;
    public float $discount = 0;
    public float $shipping = 0;
    public float $total = 0;

    public function __construct(
        public readonly PlaceOrderData $input,
    ) {}
}
```

The original `PlaceOrderData` input is stored as a `readonly` property — steps can read from it but cannot modify it. The mutable properties (`subtotal`, `tax`, `discount`, `shipping`, `total`, `order`) are populated by each step in sequence. `CalculateSubtotalStep` writes `$data->subtotal`, `ApplyCouponStep` reads `$data->subtotal` and writes `$data->discount`, `CalculateTaxStep` reads both and writes `$data->tax` and `$data->total`, and so on.

This pattern keeps each step decoupled. A step does not need to know which step comes before or after it — it only needs to know which properties to read and write on the passable object.

## Pipelines for Query Filtering

Index pages with filters tend to accumulate `when()` calls. Filter by status, filter by date range, filter by customer, sort by field — each one adds another conditional clause to the query. The controller grows, the logic is hard to reuse, and adding a new filter means editing the same bloated method.

Pipelines solve this cleanly. Each filter becomes its own class that conditionally modifies the [query builder](/books/clean-code-in-laravel/custom-query-builders-and-collections). Adding a new filter means creating one class and adding it to the array — no [controller](/books/clean-code-in-laravel/controllers) changes needed:

```php
namespace App\Pipelines\Filters;

use Closure;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Http\Request;

class FilterByStatus
{
    public function __construct(
        private readonly Request $request,
    ) {}

    public function handle(Builder $query, Closure $next): mixed
    {
        if ($this->request->filled('status')) {
            $query->where('status', $this->request->input('status'));
        }

        return $next($query);
    }
}
```

```php
class FilterByDateRange
{
    public function __construct(
        private readonly Request $request,
    ) {}

    public function handle(Builder $query, Closure $next): mixed
    {
        if ($this->request->filled('from')) {
            $query->where('created_at', '>=', $this->request->input('from'));
        }

        if ($this->request->filled('to')) {
            $query->where('created_at', '<=', $this->request->input('to'));
        }

        return $next($query);
    }
}
```

```php
class SortByField
{
    private const array ALLOWED_SORTS = ['created_at', 'total', 'status'];
    private const array ALLOWED_DIRECTIONS = ['asc', 'desc'];

    public function __construct(
        private readonly Request $request,
    ) {}

    public function handle(Builder $query, Closure $next): mixed
    {
        $sort = $this->request->input('sort', 'created_at');
        $direction = $this->request->input('direction', 'desc');

        if (in_array($sort, self::ALLOWED_SORTS, true) && in_array($direction, self::ALLOWED_DIRECTIONS, true)) {
            $query->orderBy($sort, $direction);
        }

        return $next($query);
    }
}
```

The [controller](/books/clean-code-in-laravel/controllers) passes the query builder through the pipeline:

```php
public function index(Request $request): View
{
    $orders = app(Pipeline::class)
        ->send(Order::query())
        ->through([
            FilterByStatus::class,
            FilterByDateRange::class,
            FilterByCustomer::class,
            SortByField::class,
        ])
        ->thenReturn()
        ->paginate(15);

    return view('orders.index', compact('orders'));
}
```

Adding a new filter means creating one class and adding it to the array — no controller changes needed. Each filter is independently testable, and filters that do not receive input simply pass the query through unmodified.

## Error Handling in Pipelines

When a step throws an exception, the Pipeline stops immediately — no further steps execute. This is usually the behavior you want. But consider what happens when `ReserveInventoryStep` fails after `CreateOrderStep` has already inserted a row into the database. Without a transaction, you end up with an order record that has no reserved inventory — a half-completed operation that corrupts your data.

For pipelines that modify the database across multiple steps, wrap the entire pipeline in a transaction:

```php
use Illuminate\Support\Facades\DB;

class PlaceOrderAction
{
    public function execute(PlaceOrderData $input): Order
    {
        $pipelineData = new OrderPipelineData($input);

        $result = DB::transaction(function () use ($pipelineData): OrderPipelineData {
            return app(Pipeline::class)
                ->send($pipelineData)
                ->through([
                    CalculateSubtotalStep::class,
                    ApplyCouponStep::class,
                    CalculateTaxStep::class,
                    CreateOrderStep::class,
                    ReserveInventoryStep::class,
                ])
                ->thenReturn();
        });

        event(new OrderPlaced($result->order));

        return $result->order;
    }
}
```

If `ReserveInventoryStep` fails, the order created by `CreateOrderStep` is rolled back automatically. Notice that the event dispatch happens *outside* the transaction — you only want to fire events after the data is committed.

For steps that call external APIs (which cannot be rolled back), place them after the transaction or dispatch them as [Jobs](/books/clean-code-in-laravel/jobs) instead.

## Conditional Steps

Not every order needs shipping. A digital product has no weight, no address, and no shipping cost to calculate. Running a `CalculateShippingStep` for a digital order is unnecessary — and if the step expects shipping data that does not exist, it might throw an error.

Since the steps array is just a plain PHP array, you can build it dynamically before passing it to `through()`:

```php
$steps = [
    CalculateSubtotalStep::class,
    ApplyCouponStep::class,
    CalculateTaxStep::class,
];

if ($data->input->requiresShipping) {
    $steps[] = CalculateShippingStep::class;
}

$steps[] = CreateOrderStep::class;

app(Pipeline::class)
    ->send($data)
    ->through($steps)
    ->thenReturn();
```

## Testing Pipeline Steps

One of the main reasons to use Pipelines is testability. Each step is an independent class with a single responsibility, so you can test it in isolation — no need to run the entire pipeline or make HTTP requests. Instantiate the step, pass in a passable object with the state you need, and call `handle()` with a no-op closure as `$next`:

```php
it('calculates tax correctly', function (): void {
    $data = new OrderPipelineData(
        input: new PlaceOrderData(/* ... */),
    );
    $data->subtotal = 100.00;
    $data->discount = 10.00;

    $step = app(CalculateTaxStep::class);

    $step->handle($data, fn (OrderPipelineData $data): mixed => $data);

    expect($data->tax)->toBe(18.90)  // 21% of (100 - 10)
        ->and($data->total)->toBe(108.90);
});
```

## Organizing Pipelines

```
app/Pipelines/
├── Filters/
│   ├── FilterByStatus.php
│   ├── FilterByDateRange.php
│   ├── FilterByCustomer.php
│   └── SortByField.php
├── Order/
│   ├── OrderPipelineData.php
│   ├── CalculateSubtotalStep.php
│   ├── ApplyCouponStep.php
│   ├── CalculateTaxStep.php
│   └── CreateOrderStep.php
└── Import/
    ├── ImportPipelineData.php
    ├── ValidateRowStep.php
    ├── TransformDataStep.php
    └── SaveRecordStep.php
```

## When to Use Pipelines

Pipelines are a tool for a specific kind of problem. Using them in the wrong situation adds ceremony without benefit.

**Use a Pipeline when:**

- **The process has five or more sequential steps.** The checkout example has seven steps. Writing all seven in a single method creates a 50+ line block that is hard to scan, test, or modify. A Pipeline breaks it into independently understandable pieces.
- **Steps vary between contexts.** A B2B storefront skips coupons. A digital checkout skips shipping. A wholesale order uses a different tax calculator. With a Pipeline, you build the steps array dynamically — remove a class, swap a class, add a class. With sequential code, you add `if` statements until the method is unreadable.
- **Steps are reused across different operations.** The same `CalculateTaxStep` might run during checkout, invoice recalculation, and quote generation. As a Pipeline step, it is a standalone class you can drop into any pipeline. As inline code, you copy-paste it or extract a helper method — at which point you are halfway to a Pipeline anyway.
- **Each step needs its own unit test.** When tax calculation has edge cases — tax-exempt products, multi-jurisdiction rules, rounding — you want a focused test that instantiates `CalculateTaxStep` with a mock calculator and asserts the result. A Pipeline step gives you that test boundary for free.
- **Query filtering with user-controlled parameters.** Index pages with filters are a natural fit. Each filter is a step that conditionally modifies the query builder. Adding a new filter means creating one class — no controller changes.

**Do not use a Pipeline when:**

- **The operation has fewer than three steps.** A Pipeline with two steps — say, "validate" and "save" — adds a passable class, two step classes, and a Pipeline invocation to replace two lines of sequential code. That is more ceremony than value.
- **The steps never change.** If every checkout always runs the same steps in the same order with no variation, a well-structured method with extracted private methods is simpler. Pipelines earn their keep through configurability. Fixed logic does not need it.
- **Steps do not share intermediate state.** If each operation is independent — send a welcome email, create a Stripe customer, log an analytics event — these are better modeled as [event listeners](https://laravel.com/docs/events) or [Jobs](/books/clean-code-in-laravel/jobs). Pipelines are for sequential operations where step B depends on step A's output.
- **The logic branches heavily.** Pipelines are linear — data flows forward through steps. If your process requires branching into completely different flows based on intermediate results (refund vs. exchange vs. store credit), a Pipeline forces that branching into the step classes, which defeats the purpose of clear step-by-step flow. Use separate [Actions](/books/clean-code-in-laravel/actions) for each branch instead.

Pipelines are not a replacement for [Actions](/books/clean-code-in-laravel/actions) or [Services](/books/clean-code-in-laravel/organizing-your-application). They are a tool for organizing complex, multi-step logic *within* an Action when the number of steps and the need for configurability justify separate step classes.

## Pipelines vs. Middleware vs. Job Chains

Pipelines, HTTP middleware, and Job chains all pass work through a series of steps. They differ in *what* they process and *when* they run:

| Pattern | Processes | Execution | Stops On |
|---|---|---|---|
| Pipeline | Any data (DTO, query builder, etc.) | Synchronous, within a request | Exception |
| HTTP Middleware | HTTP request/response | Synchronous, per request | Exception or early response |
| Job Chain | Background tasks | Asynchronous, via queue workers | Job failure |

HTTP [middleware](https://laravel.com/docs/middleware) handles cross-cutting HTTP concerns — authentication, CORS, [rate limiting](https://laravel.com/docs/rate-limiting). You do not use middleware for business logic. Pipelines handle multi-step business operations synchronously. [Job chains](https://laravel.com/docs/queues#job-chaining) handle multi-step background work asynchronously.

If you find yourself writing a Pipeline that dispatches Jobs, or middleware that does business logic, you are using the wrong tool.

## PHP 8.5's Pipe Operator

PHP 8.5 introduced a native [pipe operator](https://wiki.php.net/rfc/pipe-operator-v3) (`|>`) that passes the left-hand value as the argument to the right-hand callable:

```php
$result = "  Hello World  "
    |> trim(...)
    |> strtolower(...)
    |> strlen(...);
// $result === 11
```

A more practical example — normalizing user input:

```php
$slug = $request->input('title')
    |> trim(...)
    |> strtolower(...)
    |> (fn (string $s): string => str_replace(' ', '-', $s));
```

The pipe operator is great for simple, linear data transformations. But it differs from Laravel's `Pipeline` in important ways:

| | PHP 8.5 `\|>` | Laravel `Pipeline` |
|---|---|---|
| **Type** | Language operator | Userland class |
| **Callables** | Single-argument functions only | Class-based steps with `$next` closure |
| **DI** | No — plain functions | Yes — steps resolved from the container |
| **Use case** | Linear data transformations | Multi-step business logic with branching, middleware-style chains |

The pipe operator replaces nested function calls like `strlen(strtolower(trim($input)))` with a readable left-to-right chain. But it does not support the middleware pattern where each step decides whether to call `$next`, it cannot inject dependencies into steps, and each callable must accept exactly one argument.

Use the pipe operator for quick transformations — formatting strings, chaining array functions, normalizing input. Use Laravel's `Pipeline` when you need class-based steps, dependency injection, conditional flow, or shared state between steps.

## The Pipelines Checklist

1. **One step, one responsibility** — each step class does exactly one thing
2. **Use a passable object** for complex pipelines — accumulate state in a dedicated class, not loose variables
3. **Validate user input in filter pipes** — never pass raw request input directly to `orderBy()` or `where()` without an allowlist
4. **Wrap database-modifying pipelines in a transaction** — if a later step fails, earlier writes should roll back
5. **Place external API calls outside the transaction** — API calls cannot be rolled back, so dispatch them as [Jobs](/books/clean-code-in-laravel/jobs) or fire events after the transaction commits
6. **Do not overuse Pipelines** — if your operation has two or three steps, sequential method calls are simpler. Let the complexity justify the pattern
7. **Test steps in isolation** — instantiate the step, pass a prepared passable, and call `handle()` with a no-op `$next` closure

## Summary

- A Pipeline passes data through a series of steps, where each step can modify the data and pass it to the next. Laravel's HTTP middleware is built on the same `Illuminate\Pipeline\Pipeline` class.
- Each step is an independent class with a `handle()` method. Steps receive dependencies through [constructor injection](/books/clean-code-in-laravel/dependency-injection), making them easy to test in isolation.
- Use a dedicated "passable" object to carry state between steps when the pipeline needs to accumulate data across multiple stages.
- Dynamic query filtering is one of the best use cases for Pipelines — each filter is a pipe that conditionally modifies a query builder.
- Wrap database-modifying pipelines in a `DB::transaction()`. Place external API calls or event dispatches outside the transaction.
- Conditionally include steps based on runtime logic by building the steps array dynamically before passing it to `through()`.
- Pipelines complement [Actions](/books/clean-code-in-laravel/actions) — they organize multi-step logic *within* an Action when the number of steps grows beyond what a single method can comfortably hold.
- Do not use a Pipeline for two-step operations. Let the complexity justify the pattern.

## References

- [Understanding Laravel Pipelines](https://jeffochoa.me/understanding-laravel-pipelines) — Jeff Ochoa
- [Laravel: The Hidden Pipeline (4-part series)](https://medium.com/swlh/laravel-the-hidden-pipeline-part-1-a4ae91fc55a4) — Italo Baeza Cabrera
- [Easy Query Filters with Laravel Pipelines](https://codecourse.com/courses/easy-query-filters-with-laravel-pipelines) — Codecourse
- [How To Use Laravel Pipelines To Implement More Advanced Filters](https://martinjoo.dev/how-to-use-laravel-pipelines-to-implement-more-advanced-filters) — Martin Joo
- [A Guide to Laravel Pipelines](https://www.honeybadger.io/blog/laravel-pipeline/) — Honeybadger
- [Pipeline Pattern in Laravel](https://dev.to/abrardev99/pipeline-pattern-in-laravel-278p) — DEV Community
- [Laravel Pipeline Query Collection](https://laravel-news.com/pipeline-query-collection) — Laravel News
- [Supercharged Pipelines for Laravel](https://laravel-news.com/supercharged-pipelines-for-laravel) — Laravel News
- [PHP RFC: Pipe Operator v3](https://wiki.php.net/rfc/pipe-operator-v3) — PHP Internals
- [The Pipe Operator in PHP 8.5](https://stitcher.io/blog/pipe-operator-in-php-85) — Brent Roose
- [The Pipe Operator is Coming to PHP 8.5](https://laravel-news.com/the-pipe-operator-is-coming-to-php-85) — Laravel News
