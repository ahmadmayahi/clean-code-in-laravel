Arrays are flexible, easy to create, and work everywhere. They are also the source of countless bugs.

What keys does this array have? What types are the values?

```php
$orderData = [
    'user_id' => $user->id,
    'items' => $items,
    'shipping' => $address,
    'coupon' => $couponCode,
];

$this->placeOrder($orderData);
```

When you pass an array to a method, you have no idea what is inside it without reading the implementation. Did someone add a `discount` key? Is `items` an array of arrays or a Collection? Is `shipping` an Address model or a string? The array tells you nothing.

A Data Transfer Object (DTO) solves this by giving your data a shape. It is a simple class whose only job is to carry data from one place to another, with explicit types and names.

## A Plain PHP DTO

At its simplest, a DTO is a readonly class with typed properties:

```php
namespace App\DataTransferObjects;

readonly class PlaceOrderData
{
    public function __construct(
        public int $userId,
        public int $shippingAddressId,
        public Collection $items,
        public ?string $couponCode = null,
    ) {}
}
```

Now the [Action](/books/clean-code-in-laravel/actions) method signature tells you exactly what it expects:

```php
public function execute(PlaceOrderData $data): Order
{
    // $data->userId is guaranteed to be an int
    // $data->items is guaranteed to be a Collection
    // $data->couponCode is either a string or null
}
```

The `readonly` keyword (PHP 8.2+) ensures that once a DTO is created, its properties cannot be changed. This makes DTOs predictable and safe to pass around.

## Spatie's Laravel Data Package

[`spatie/laravel-data`](https://spatie.be/docs/laravel-data) takes DTOs to another level. It combines data objects with transformation, serialization, and nested casting — all in one class. It also supports validation through attributes, which we will discuss shortly.

```bash
composer require spatie/laravel-data
```

Here is the same DTO using `spatie/laravel-data`:

```php
namespace App\DataTransferObjects;

use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\Min;
use Spatie\LaravelData\Attributes\Validation\Required;
use Spatie\LaravelData\Attributes\Validation\Exists;

class PlaceOrderData extends Data
{
    public function __construct(
        #[Required, Exists('users', 'id')]
        public int $userId,

        #[Required, Exists('addresses', 'id')]
        public int $shippingAddressId,

        #[Required, Min(1)]
        /** @var Collection<int, OrderItemData> */
        public Collection $items,

        public ?string $couponCode = null,
    ) {}
}
```

You can create a data object from a request, from an array, or from a model:

```php
// From a request (validates automatically)
$data = PlaceOrderData::from($request);

// From an array
$data = PlaceOrderData::from([
    'userId' => 1,
    'shippingAddressId' => 5,
    'items' => [['productId' => 1, 'quantity' => 2]],
]);

// To an array (for storage or API responses)
$array = $data->toArray();

// To JSON
$json = $data->toJson();
```

## Nested DTOs

Real-world data is nested. An order has items, and each item has its own shape. With `spatie/laravel-data`, nested DTOs are handled automatically:

```php
class OrderItemData extends Data
{
    public function __construct(
        public int $productId,
        public int $quantity,
        public ?float $unitPrice = null,
    ) {}
}

class PlaceOrderData extends Data
{
    public function __construct(
        public int $userId,
        public int $shippingAddressId,
        /** @var Collection<int, OrderItemData> */
        public Collection $items,
        public ?string $couponCode = null,
    ) {}
}

// Creating from nested arrays — items are automatically cast to OrderItemData
$data = PlaceOrderData::from([
    'userId' => 1,
    'shippingAddressId' => 5,
    'items' => [
        ['productId' => 1, 'quantity' => 2],
        ['productId' => 3, 'quantity' => 1],
    ],
]);

// $data->items is a Collection of OrderItemData objects
$data->items->each(function (OrderItemData $item): void {
    // $item->productId, $item->quantity are typed
});
```

## Keep Validation in Form Requests

While `spatie/laravel-data` supports validation through attributes and can replace Form Requests entirely, I prefer not to use it that way. I use the package strictly for what it does best — defining typed data structures. Validation stays in dedicated [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation).

There are two reasons for this. First, Laravel's Form Requests are a first-class feature with deep integration — custom error messages, authorization via `authorize()`, after-validation hooks, and conditional rules. Replacing them with a third-party package's validation attributes means giving up that integration. Second, mixing validation into DTOs blurs responsibilities. A DTO's job is to carry data. A Form Request's job is to validate input. When one class does both, developers have to guess where to look for validation rules.

The pattern is simple: the Form Request validates, then creates a DTO for the Action:

```php
class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'user_id' => ['required', 'exists:users,id'],
            'shipping_address_id' => ['required', 'exists:addresses,id'],
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'exists:products,id'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
            'coupon_code' => ['nullable', 'string'],
        ];
    }

    public function toDto(): PlaceOrderData
    {
        return PlaceOrderData::from($this->validated());
    }
}
```

The [controller](/books/clean-code-in-laravel/controllers) stays thin and the [Action](/books/clean-code-in-laravel/actions) receives a fully typed object:

```php
class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, PlaceOrderAction $action): RedirectResponse
    {
        $order = $action->execute($request->toDto());

        return redirect()->route('orders.show', $order);
    }
}
```

Each class does one thing. The Form Request handles validation and authorization. The DTO carries typed data. The Action performs the business operation. No overlap, no confusion.

## Transforming Data with DTOs

DTOs are perfect for transforming data between layers. A model might have more fields than an API response needs, or an API request might use different names than your database:

```php
class UserProfileData extends Data
{
    public function __construct(
        public string $name,
        public string $email,
        public ?string $avatarUrl,
        public string $memberSince,
        public int $orderCount,
    ) {}

    public static function fromModel(User $user): self
    {
        return new self(
            name: $user->name,
            email: $user->email,
            avatarUrl: $user->avatar_url,
            memberSince: $user->created_at->format('F Y'),
            orderCount: $user->orders()->count(),
        );
    }
}
```

## DTOs with Enums

DTOs work beautifully with PHP enums (covered in [Enums, Value Objects, and Type Safety](/books/clean-code-in-laravel/enums-value-objects-and-type-safety)):

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';
}

class UpdateOrderStatusData extends Data
{
    public function __construct(
        public int $orderId,
        public OrderStatus $status,
        public ?string $reason = null,
    ) {}
}

// From a request with status="shipped" — automatically cast to enum
$data = UpdateOrderStatusData::from($request);
$data->status; // OrderStatus::Shipped
```

## When to Use DTOs

DTOs are valuable when data crosses boundaries:

| Boundary | Example |
|---|---|
| HTTP → Business Logic | Request data → [Action](/books/clean-code-in-laravel/actions) parameter |
| Business Logic → View | Model data → [View Model](/books/clean-code-in-laravel/view-models) / API response |
| [Service](/books/clean-code-in-laravel/organizing-your-application) → Service | Data passed between Services |
| Event payload | Data attached to an event |
| [Job](/books/clean-code-in-laravel/jobs) payload | Data serialized for a queued job |

You do not need a DTO for everything. If you are passing a single model to a view, just pass the model. If you are passing two validated fields to a simple create operation, `$request->validated()` is fine. DTOs shine when the data is complex, crosses multiple boundaries, or needs to be reused.

## The DTO Checklist

1. Use `readonly` classes for plain DTOs — immutability prevents bugs
2. Use [`spatie/laravel-data`](https://spatie.be/docs/laravel-data) when you need transformation, serialization, or nested data casting
3. Nest DTOs for complex data structures — do not flatten everything
4. Create from multiple sources — requests, arrays, models, API responses
5. Keep DTOs focused — one DTO per use case, not one DTO per model
6. Keep validation in [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation) — use `toDto()` to bridge validated input to a typed DTO
7. Type everything — the whole point of a DTO is type safety

## Summary

- A DTO is a simple class whose only job is to carry typed data from one place to another. It replaces the untyped arrays that cause bugs and confusion.
- At its simplest, a DTO is a `readonly` class with typed properties and a constructor. PHP guarantees the shape of your data at the language level.
- [`spatie/laravel-data`](https://spatie.be/docs/laravel-data) takes DTOs further by combining data objects with transformation, serialization, and nested casting. While it supports validation through attributes, I prefer keeping validation in dedicated [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation) and using a `toDto()` method to bridge validated input to a typed DTO.
- DTOs can nest. An order DTO contains a collection of order item DTOs. `spatie/laravel-data` handles the casting automatically.
- DTOs are valuable when data crosses boundaries — HTTP to business logic, service to service, or event and job payloads. You do not need a DTO for everything.
- Type everything. The whole point of a DTO is type safety. If a property can be null, declare it as nullable. If it is an enum, type it as an enum.

## References

- [Introduction to laravel-data](https://spatie.be/docs/laravel-data/v4/introduction) — Spatie Documentation
- [From a Request to a Data Object](https://spatie.be/docs/laravel-data/v4/as-a-data-transfer-object/request-to-data-object) — Spatie Documentation
- [Nesting Data Objects](https://spatie.be/docs/laravel-data/v4/as-a-data-transfer-object/nesting) — Spatie Documentation
- [Avoid Describing Your Data Multiple Times Using laravel-data](https://freek.dev/2118-avoid-describing-your-data-multiple-times-in-a-laravel-app-using-laravel-data) — Freek Van der Herten
- [Domain-Driven Design with Laravel — Data Transfer Objects](https://martinjoo.dev/domain-driven-design-with-laravel-data-transfer-objects) — Martin Joo
- [Data Transfer Objects (DTOs) in PHP](https://ashallendesign.co.uk/blog/data-transfer-objects-dtos-in-php) — Ash Allen
- [Readonly Classes in PHP 8.2](https://stitcher.io/blog/readonly-classes-in-php-82) — Brent Roose
- [Modeling Data with DTOs](/books/thinking-in-domains-in-laravel/modeling-data-with-dtos) — Thinking in Domains in Laravel
