"Where do I put this logic?" is the most common question in the Laravel community. We already have two answers: [thin controllers](/books/clean-code-in-laravel/controllers) that delegate everything, and [Actions](/books/clean-code-in-laravel/actions) that handle discrete business operations. But what about reusable calculations, third-party API wrappers, shared traits, and utility functions that do not fit into a single operation?

That is where Services, Concerns, and the Support folder come in. Together with Actions, they form a clear hierarchy for organizing logic in a Laravel application.

## Services

A [Service](https://martinfowler.com/eaaCatalog/serviceLayer.html) is a stateless class that provides reusable logic to the rest of your application. Controllers call it, [Actions](/books/clean-code-in-laravel/actions) call it, [jobs](https://laravel.com/docs/queues) call it - but it does not own the workflow. It just does its job when asked.

Unlike an Action, a Service can have multiple public methods. Unlike a [Model](https://laravel.com/docs/eloquent), a Service does not represent data. It is a capability provider - it knows how to do something, and any part of your application can use it.

Here is the simplest possible Service - a class that formats money values:

```php
namespace App\Services;

class MoneyFormatter
{
    public function format(int $cents, string $currency = 'USD'): string
    {
        return match ($currency) {
            'USD' => '$' . number_format($cents / 100, 2),
            'EUR' => number_format($cents / 100, 2, ',', '.') . ' €',
            default => number_format($cents / 100, 2) . ' ' . $currency,
        };
    }

    public function formatCompact(int $cents, string $currency = 'USD'): string
    {
        if ($cents >= 100_00) {
            return match ($currency) {
                'USD' => '$' . round($cents / 100_00, 1) . 'k',
                default => round($cents / 100_00, 1) . 'k ' . $currency,
            };
        }

        return $this->format($cents, $currency);
    }
}
```

Two methods, no state, no side effects. Given the same input, it returns the same output. Any [Action](/books/clean-code-in-laravel/actions), Controller, or [Job](https://laravel.com/docs/queues) can [inject](/books/clean-code-in-laravel/dependency-injection) it and use it.

Here is a more practical example - a Service that calculates tax:

```php
namespace App\Services;

use App\Models\Order;

class TaxCalculator
{
    public function calculate(int $subtotalInCents, string $country): int
    {
        $rate = match ($country) {
            'NL' => 0.21,
            'DE' => 0.19,
            'FR' => 0.20,
            default => 0.0,
        };

        return (int) round($subtotalInCents * $rate);
    }

    public function calculateForOrder(Order $order): int
    {
        return $this->calculate(
            $order->subtotal_in_cents,
            $order->shippingAddress->country,
        );
    }
}
```

Notice that neither of these classes has a `Service` suffix. Whether to use the suffix or not is a team decision. I prefer not to use it - a well-named class like `TaxCalculator` or `ShippingRateCalculator` does not need the extra word. Either approach works, as long as you are consistent across your project.

### Services vs. Actions

The [Actions](/books/clean-code-in-laravel/actions) chapter covers this distinction in detail, but here is the short version: an Action is something your application does, a Service is something your application uses.

An Action is a business operation. A user clicks "Buy Now" and `PlaceOrderAction` validates the cart, charges the payment, creates the order, and dispatches an event. An admin clicks "Suspend" and `SuspendUserAction` revokes sessions, cancels subscriptions, and sends a notification. Actions have side effects, one public method (`execute`), and they orchestrate a workflow.

A Service is a reusable capability. `TaxCalculator` takes a subtotal and a country and returns a tax amount - it does not care whether it is called from `PlaceOrderAction` or `GenerateInvoiceAction`. `CurrencyConverter` takes an amount and two currency codes and returns the converted value. Services have multiple methods, no side effects on their own, and they do not own the workflow.

The rule is simple: Actions call Services, not the other way around.

```php
class PlaceOrderAction
{
    public function __construct(
        private readonly TaxCalculator $taxCalculator,
        private readonly InventoryChecker $inventory,
        private readonly PaymentGateway $payment,
    ) {}

    public function execute(PlaceOrderData $data): Order
    {
        $this->inventory->reserveItems($data->items);
        $tax = $this->taxCalculator->calculate($data->subtotal, $data->country);
        $this->payment->charge($data->paymentMethodId, $data->subtotal + $tax);

        $order = Order::create([/* ... */]);

        event(new OrderPlaced($order));

        return $order;
    }
}
```

Three Services, one Action. The Action decides what happens and in what order. The Services do the calculating, checking, and charging.

### Service Anti-Patterns

Here is what a Service should *not* become. These are the three most common mistakes in Laravel projects.

**The "Model Action Service."** A class named `UserService` with `create()`, `update()`, `delete()`, `suspend()`, `resetPassword()`, and `sendWelcomeEmail()`. It looks organized at first - everything about users in one place. But it grows into a God class with dozens of methods, hundreds of lines, and no clear boundary. Every time something user-related needs to happen, it gets another method.

The fix is to break it apart. `SuspendUserAction` handles suspension. `ResetPasswordAction` handles password resets. `UserAvatarService` handles avatar uploads and resizing. Each class has one reason to change.

**The empty controller, bloated service.** Some teams move *all* controller logic into a service class one-to-one. The controller becomes an empty shell that calls `$this->userService->store($request)`, and the service handles validation, authorization, response formatting, and business logic. You have not separated concerns - you have just moved the mess to a different file.

[Controllers](/books/clean-code-in-laravel/controllers) should stay thin, but they still own HTTP concerns: reading request data, calling an Action or Service, and returning a response. If your service accepts a `Request` object or returns a `JsonResponse`, it is doing controller work.

**The catch-all service.** When a team has no clear convention for where logic goes, "just put it in a service" becomes the default answer. The result is an `app/Services/` directory full of classes with vague names like `HelperService`, `UtilityService`, or `DataService` - each one a grab bag of unrelated methods.

The answer is naming. A well-named class answers the question of where logic goes before it is even asked. `TaxCalculator`, `ShippingClient`, `ExchangeRateProvider` - these names make it obvious what the class does and what it does not do. If you cannot give your service a specific name, it probably should not be a single class.

### Wrapping Third-Party APIs

One of the most common uses for a Service is wrapping a third-party API. Instead of scattering [HTTP client](https://laravel.com/docs/http-client) calls across your Actions, you put them in one class:

```php
namespace App\Services\Shipping;

use App\Data\ShippingRateData;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Collection;

class ShippingClient
{
    /** @return Collection<int, ShippingRateData> */
    public function getRatesForPackage(
        string $originZip,
        string $destinationZip,
        float $weightKg,
    ): Collection {
        $response = Http::withToken(config('services.shipping.api_key'))
            ->post(config('services.shipping.base_url') . '/rates', [
                'origin' => $originZip,
                'destination' => $destinationZip,
                'weight' => $weightKg,
            ])
            ->throw();

        return collect($response->json('rates'))
            ->map(fn (array $rate): ShippingRateData => ShippingRateData::from($rate))
            ->sortBy('price');
    }

    public function getCheapestRate(
        string $originZip,
        string $destinationZip,
        float $weightKg,
    ): ShippingRateData {
        return $this->getRatesForPackage($originZip, $destinationZip, $weightKg)
            ->first();
    }

    public function trackShipment(string $trackingNumber): ShipmentStatusData
    {
        $response = Http::withToken(config('services.shipping.api_key'))
            ->get(config('services.shipping.base_url') . "/track/{$trackingNumber}")
            ->throw();

        return ShipmentStatusData::from($response->json());
    }
}
```

The API key and base URL come from [`config('services.shipping.*')`](https://laravel.com/docs/configuration) - never hardcode credentials or pass them as constructor parameters. Laravel's `config()` helper reads from your configuration files, which in turn pull values from your `.env` file. This keeps secrets out of your code and makes them easy to change per environment.

Now any class can [inject](/books/clean-code-in-laravel/dependency-injection) `ShippingClient` and use it:

```php
class CalculateShippingAction
{
    public function __construct(
        private readonly ShippingClient $shipping,
    ) {}

    public function execute(Cart $cart, Address $address): ShippingRateData
    {
        return $this->shipping->getCheapestRate(
            originZip: config('shop.warehouse_zip'),
            destinationZip: $address->zip_code,
            weightKg: $cart->totalWeight(),
        );
    }
}
```

### Where to Put Services

This is where the Laravel community splits. Some teams create an `app/Services/` directory and organize by domain:

```
app/Services/
├── Payment/
│   ├── StripeGateway.php
│   └── PayPalGateway.php
├── Shipping/
│   └── ShippingClient.php
└── Tax/
    └── TaxCalculator.php
```

Others - including Taylor Otwell himself - skip the `Services` folder entirely. None of Laravel's first-party packages use a `Services/` directory. [Horizon](https://laravel.com/docs/horizon) has over 30 classes sitting flat in its root namespace - `AutoScaler`, `Supervisor`, `WaitTimeCalculator` - without a `Services` folder in sight. [Cashier](https://laravel.com/docs/billing) puts domain classes like `Subscription`, `Payment`, and `Invoice` flat alongside each other. When Taylor does create directories, they are domain-specific and descriptive: `Watchers/` in [Telescope](https://laravel.com/docs/telescope), `Recorders/` in [Pulse](https://laravel.com/docs/pulse), `Drivers/` in [Pennant](https://laravel.com/docs/pennant). [Jetstream](https://jetstream.laravel.com/) and [Fortify](https://laravel.com/docs/fortify) scaffold `app/Actions/`, not `app/Services/`. The default Laravel skeleton has never included a `Services` directory.

The flat approach works well for small-to-medium projects. When you only have three or four service-like classes, a dedicated folder adds ceremony without value. As the number grows, grouping them makes classes easier to find. Use your judgment — there is no wrong answer here, as long as you are consistent.

### Interfaces for Swappable Services

Earlier we saw `StripeGateway` and `PayPalGateway` sitting side by side in the Services directory. But how does the rest of your application choose between them? This is where interfaces come in.

Define an interface — a contract that describes what a payment gateway can do, without specifying how. In Laravel, the term "contract" simply means a PHP interface that defines a set of methods a class must implement. Laravel's own core features are defined as [contracts](https://laravel.com/docs/contracts) (for example, `Illuminate\Contracts\Queue\Queue`), and the same pattern works for your application code:

```php
namespace App\Services\Payment;

interface PaymentGateway
{
    public function charge(int $amountInCents, string $paymentToken): PaymentResult;

    public function refund(string $transactionId, int $amountInCents): PaymentResult;
}
```

Both `StripeGateway` and `PayPalGateway` implement this interface:

```php
class StripeGateway implements PaymentGateway
{
    public function charge(int $amountInCents, string $paymentToken): PaymentResult
    {
        // Stripe-specific implementation
    }

    public function refund(string $transactionId, int $amountInCents): PaymentResult
    {
        // Stripe-specific implementation
    }
}
```

Then bind the interface to the active implementation in a [service provider](https://laravel.com/docs/providers):

```php
// AppServiceProvider.php
public function register(): void
{
    $this->app->bind(
        PaymentGateway::class,
        config('services.payment.driver') === 'paypal'
            ? PayPalGateway::class
            : StripeGateway::class,
    );
}
```

Now any class that type-hints `PaymentGateway` gets the right implementation automatically. Switching payment providers means changing one config value - not hunting through dozens of files.

You do not need interfaces for every Service. `TaxCalculator` and `MoneyFormatter` work fine as concrete classes - there is no reason to swap them. Use interfaces when you have multiple implementations of the same capability, or when you need to replace a Service with a fake during testing.

### Testing Services

Stateless Services are the easiest classes to test. No database, no HTTP faking, no mocking - just call a method and check the result:

```php
it('calculates Dutch tax at 21%', function (): void {
    $calculator = new TaxCalculator();

    $tax = $calculator->calculate(10000, 'NL');

    expect($tax)->toBe(2100);
});

it('returns zero tax for unknown countries', function (): void {
    $calculator = new TaxCalculator();

    $tax = $calculator->calculate(10000, 'XX');

    expect($tax)->toBe(0);
});
```

Services that wrap external APIs or depend on other Services need mocking or [HTTP faking](https://laravel.com/docs/http-client#testing). [Inject](/books/clean-code-in-laravel/dependency-injection) a mock or a fake, and test the behavior in isolation:

```php
it('calculates shipping with the cheapest rate', function (): void {
    $fakeClient = Mockery::mock(ShippingClient::class);
    $fakeClient->shouldReceive('getCheapestRate')
        ->andReturn(new ShippingRateData(price: 599, carrier: 'DHL'));

    $action = new CalculateShippingAction($fakeClient);

    $rate = $action->execute($cart, $address);

    expect($rate->price)->toBe(599);
});
```

If a Service is hard to test, it is usually doing too much. Difficulty writing tests is a design signal - break the class apart until each piece is easy to test in isolation.

## Concerns

Laravel uses the term Concerns for [traits](https://www.php.net/manual/en/language.oop5.traits.php) that belong to a specific layer of your application. If you look at Laravel's own source code, you will find `Illuminate\Database\Eloquent\Concerns`, `Illuminate\Auth\Access\HandlesAuthorization`, and similar namespaces. The convention is clear: traits that add behavior to a specific type of class live in a `Concerns` directory next to those classes.

In your application, this means model-specific traits go in `app/Models/Concerns/`:

```php
// app/Models/Concerns/HasSubscription.php
namespace App\Models\Concerns;

use App\Models\Subscription;
use Illuminate\Database\Eloquent\Relations\HasOne;

trait HasSubscription
{
    /** @return HasOne<Subscription, $this> */
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
// app/Models/User.php
class User extends Authenticatable
{
    use HasSubscription;
    use Searchable;
}
```

The naming conventions from the [Naming Conventions](/books/clean-code-in-laravel/naming-conventions) chapter apply here: traits use PascalCase and describe the capability they provide - `HasSubscription`, `Searchable`, `TracksActivity`. When you read `use HasSubscription`, it reads like a description of what the model can do.

### Concerns vs. Support Traits

Not all traits are Concerns. The difference is scope:

| Type | Location | Purpose | Example |
| --- | --- | --- | --- |
| Concern | `app/Models/Concerns/` | Behavior for a specific layer (models, controllers) | `HasSubscription`, `Searchable` |
| Support Trait | `app/Support/Traits/` | Cross-cutting behavior used anywhere | `HasUuid`, `Sluggable` |

A `HasSubscription` trait only makes sense on a model - it defines a relationship and business methods. That is a Concern. A `HasUuid` trait could be used on any model - it is generic infrastructure. That belongs in Support.

If you are unsure, ask: "Does this trait contain business logic specific to one layer?" If yes, it is a Concern. If no, it is a Support trait.

### Organizing Concerns

As your application grows, the `Concerns` directory keeps model traits manageable:

```
app/Models/
├── Concerns/
│   ├── HasSubscription.php
│   ├── HasTeam.php
│   ├── Searchable.php
│   └── TracksActivity.php
├── User.php
├── Team.php
├── Order.php
└── Subscription.php
```

This is the same pattern Laravel uses internally. It keeps traits discoverable - when you want to know what behaviors are available to your models, you look in one place.

### Keeping Traits Focused

A trait should do one thing. `HasSubscription` manages subscription relationships and status checks. `Searchable` adds search capabilities. When a trait starts handling subscriptions *and* billing *and* invoicing, it has become a hidden God class.

Ask yourself: can you describe what the trait does in one short sentence? If you need "and" in the description, split it into two traits.

One practical concern with traits is name collisions. If two traits define a method with the same name, PHP throws a fatal error. You can resolve this with `insteadof` and `as`:

```php
class User extends Authenticatable
{
    use HasSubscription, HasMembership {
        HasSubscription::isActive insteadof HasMembership;
        HasMembership::isActive as isMembershipActive;
    }
}
```

But if you find yourself resolving trait conflicts, it is a sign that your traits are too broad. Well-focused traits with descriptive method names rarely collide.

## Support

If [Actions](/books/clean-code-in-laravel/actions) are "what the application does" and Services are "how it does it," then the Support folder is "what everything is built on." The Support folder contains cross-cutting utilities that serve the entire application but are not business logic.

Laravel itself uses this pattern extensively. The `Illuminate\Support` namespace contains [`Str`](https://laravel.com/docs/strings), [`Arr`](https://laravel.com/docs/helpers#arrays-and-objects-method-list), [`Collection`](https://laravel.com/docs/collections), and other utilities that every part of the framework depends on. Your application's `Support` folder serves the same purpose.

Here is what goes in `Support`:

```
app/Support/
├── Traits/
│   ├── HasUuid.php
│   ├── Sluggable.php
│   └── HasMoney.php
├── Casts/
│   ├── MoneyCast.php
│   └── AddressCast.php
├── ValueObjects/
│   ├── Money.php
│   └── Address.php
├── Enums/
│   └── Currency.php
└── helpers.php
```

We cover value objects and enums in detail in [Enums, Value Objects, and Type Safety](/books/clean-code-in-laravel/enums-value-objects-and-type-safety). They live in Support because they are generic infrastructure - a `Money` value object or a `Currency` enum is not tied to any specific domain.

Traits - Generic, cross-cutting behaviors that are not tied to a specific layer. Unlike [Concerns](#concerns), these can be used anywhere:

```php
namespace App\Support\Traits;

use Illuminate\Support\Str;

trait HasUuid
{
    protected static function bootHasUuid(): void
    {
        static::creating(function (self $model): void {
            if (empty($model->{$model->getKeyName()})) {
                $model->{$model->getKeyName()} = Str::uuid()->toString();
            }
        });
    }

    public function getIncrementing(): bool
    {
        return false;
    }

    public function getKeyType(): string
    {
        return 'string';
    }
}
```

Custom Casts - Shared [Eloquent casts](https://laravel.com/docs/eloquent-mutators#custom-casts):

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
            currency: Currency::from($attributes['currency'] ?? 'USD'),
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

Custom [validation rules](https://laravel.com/docs/validation#custom-validation-rules) do not go in Support. Laravel has a dedicated `app/Rules/` directory for them - use `php artisan make:rule` to create them there.

### What Does NOT Go in Support

The Support folder is not a dumping ground. These things do not belong there:

| Does Not Belong       | Where It Goes                                                         |
| --------------------- | --------------------------------------------------------------------- |
| Business logic        | [Actions](/books/clean-code-in-laravel/actions)                 |
| API wrappers          | Services                                                              |
| Models                | `Models`                                                              |
| Model-specific traits | `Models/Concerns`                                                     |
| Validation rules      | `app/Rules/` (use `php artisan make:rule`)                            |
| Domain-specific enums | With their domain (or `Enums/`)                                       |

The test is simple: if it contains business rules or domain knowledge, it does not belong in Support. Support is for infrastructure - the plumbing that everything else is built on.

### The `helpers.php` File

You may have noticed `helpers.php` in the Support directory tree above. This is a file for global helper functions - small, stateless utilities that you want available everywhere without importing a class.

```php
// app/Support/helpers.php

if (! function_exists('money')) {
    function money(int $cents, string $currency = 'USD'): string
    {
        return match ($currency) {
            'USD' => '$' . number_format($cents / 100, 2),
            'EUR' => number_format($cents / 100, 2, ',', '.') . ' €',
            default => number_format($cents / 100, 2) . ' ' . $currency,
        };
    }
}

if (! function_exists('initials')) {
    function initials(string $name): string
    {
        return collect(explode(' ', $name))
            ->map(fn (string $part): string => mb_strtoupper(mb_substr($part, 0, 1)))
            ->implode('');
    }
}
```

To make these functions available globally, add the file to the `autoload` section in your `composer.json`:

```json
{
    "autoload": {
        "files": [
            "app/Support/helpers.php"
        ]
    }
}
```

After adding the entry, run `composer dump-autoload` so Composer picks it up. From that point, you can call `money(2500)` or `initials('Ahmad Mayahi')` anywhere in your application - controllers, views, Artisan commands, tests - without any imports.

Keep this file small. If a helper function grows beyond a few lines, or if it needs dependencies, it belongs in a Service class instead. Good candidates for helpers are formatting shortcuts, simple string transformations, and thin wrappers around verbose framework calls. If you find yourself adding dozens of functions here, you are putting too much in one file.

## When to Introduce This Structure

Not every project needs Actions, Services, and a Support folder from day one. A five-route application with basic CRUD is perfectly served by Laravel's default structure. Adding layers prematurely is over-engineering.

Here are the signs that your application has outgrown flat MVC:

- **Controllers are getting long.** When a controller method exceeds 30-40 lines, it is doing too much. When the controller class exceeds 200 lines, it needs to delegate.
- **Logic is duplicated.** The same shipping calculation appears in a controller, a job, and an Artisan command. That logic needs a home.
- **Tests are painful.** If testing a feature requires booting the entire application, seeding the database, and making HTTP requests just to verify a tax calculation, the logic is trapped in the wrong layer.
- **"Where does this go?" keeps coming up.** When code reviews repeatedly debate where new logic should live, you need clearer conventions.

Start simple. Extract your first Action when a controller does too much. Create your first Service when two Actions need the same calculation. Add a Support trait when three models share the same behavior. Let the architecture grow with the application, not ahead of it.

## The Complete Architecture So Far

Here is how a well-organized Laravel application looks at this point:

```
app/
├── Actions/                  ← Business operations
│   ├── Order/
│   │   ├── PlaceOrderAction.php
│   │   └── CancelOrderAction.php
│   └── User/
│       └── RegisterUserAction.php
├── Services/                 ← Reusable stateless logic
│   ├── Payment/
│   │   └── StripeGateway.php
│   ├── Shipping/
│   │   └── ShippingClient.php
│   └── Tax/
│       └── TaxCalculator.php
├── Support/                  ← Cross-cutting utilities
│   ├── Traits/
│   ├── Casts/
│   └── ValueObjects/
├── Http/
│   └── Controllers/          ← Thin controllers
├── Models/
│   ├── Concerns/             ← Model-specific traits
│   │   ├── HasSubscription.php
│   │   └── Searchable.php
│   ├── User.php
│   └── Order.php
└── Providers/
```

Every piece of logic has a clear home. [Controllers](/books/clean-code-in-laravel/controllers) delegate to [Actions](/books/clean-code-in-laravel/actions). Actions use Services. Services and Actions are built on Support utilities. Concerns keep model traits discoverable. No class does too much, and no logic is in the wrong place.

### Growing Beyond: Domain-First Organization

As your application continues to grow - dozens of models, hundreds of Actions, multiple teams - you may find that organizing by *type* (all Actions together, all Services together) makes related code harder to find. The billing Action is far from the billing Service, which is far from the billing Model.

At that point, some teams flip the hierarchy and organize by *domain* first:

```
app/Domain/
├── Billing/
│   ├── Actions/
│   ├── Models/
│   └── Services/
├── Shipping/
│   ├── Actions/
│   ├── Models/
│   └── Services/
└── Catalog/
    ├── Actions/
    ├── Models/
    └── Services/
```

This is the starting point of Domain-Driven Design. Each domain folder contains everything related to one area of your business, making it easier for teams to work independently and for developers to understand a feature by looking in one place.

You do not need to adopt this early. The type-based structure in this chapter scales comfortably to medium-large applications. But knowing that domain-first organization exists helps you recognize when it is time to make the switch - typically when navigating between related files across multiple top-level folders becomes a daily friction.

If you want to dive deeper into this approach, the [Thinking in Domains in Laravel](/books/thinking-in-domains-in-laravel/thinking-in-domains) book covers it in full detail - from structuring domains and defining boundaries to building an application layer on top.

Throughout the book, we add [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation), [DTOs](/books/clean-code-in-laravel/data-transfer-objects), View Models, [Jobs](/books/clean-code-in-laravel/jobs), and Pipelines to this structure. Each one has a specific role, and together they form a clean, maintainable architecture that scales from a solo project to a team of fifty.

## Summary

- A Service is a stateless class that provides reusable logic - calculations, API wrappers, formatters. Unlike Actions, it can have multiple methods. Unlike Models, it does not represent data.
- Actions are what your application does (business operations with side effects). Services are what your application uses (reusable capabilities). Actions call Services, not the other way around.
- Avoid common Service anti-patterns: the God-class `UserService` that does everything, the empty controller that just forwards to a bloated service, and the catch-all `HelperService`. Name your services specifically - `TaxCalculator`, not `OrderService`.
- Whether to suffix with `Service` is a team decision. Descriptive names like `TaxCalculator` and `ShippingClient` do not need it. Pick one convention and stick with it.
- Use interfaces when you have multiple implementations of the same capability (e.g., payment gateways) or when you need to swap a service for a test fake. You do not need interfaces for every service.
- Stateless Services are easy to test without mocking. Services that wrap external APIs need [HTTP faking](https://laravel.com/docs/http-client#testing) or mock injection. If a service is hard to test, it is doing too much.
- You do not need an `app/Services/` folder. None of Laravel's first-party packages use one. A dedicated folder is fine for larger projects, but do not create it out of obligation.
- Concerns are traits that belong to a specific layer. Model traits go in `app/Models/Concerns/`. They describe what a model can do - `HasSubscription`, `Searchable`, `TracksActivity`.
- Keep traits focused on a single responsibility. If you need "and" to describe what a trait does, split it.
- Support traits are cross-cutting. Generic behaviors like `HasUuid` or `Sluggable` that could apply to any model go in `app/Support/Traits/`.
- The Support folder is for infrastructure, not business logic. Custom casts, value objects, generic traits, and helper functions. If it contains domain knowledge, it belongs somewhere else. Validation rules go in `app/Rules/`, not Support.
- Use `helpers.php` for small global functions. Register it in `composer.json` under `autoload.files`, run `composer dump-autoload`, and keep the file small. If a helper outgrows a few lines, promote it to a class.
- Do not adopt this entire structure on day one. Start simple and extract Actions, Services, and Support classes as the application grows and the need becomes clear.
- For very large applications with multiple teams, consider flipping to domain-first organization where each domain folder contains its own Actions, Models, and Services.
- Every piece of logic has a clear home. Controllers delegate to Actions. Actions use Services. Concerns add behavior to models. Support provides the plumbing. When someone asks "where does this go?" - there is always an answer.

## References

- [Domain Oriented Laravel](https://stitcher.io/blog/laravel-beyond-crud-01-domain-oriented-laravel) — Brent Roose
- [Conciliating Laravel and DDD](https://lorisleiva.com/conciliating-laravel-and-ddd) — Loris Leiva
- [Service Pattern in Laravel: Why It Is Meaningless](https://nabilhassen.com/laravel-service-pattern-issues) — Nabil Hassen
- [Working with Third Party Services in Laravel](https://laravel-news.com/working-with-third-party-services-in-laravel) — Laravel News
- [Working with 3rd Parties in Laravel](https://martinjoo.dev/working-with-3rd-parties-in-laravel) — Martin Joo
- [Binding API Connection Interfaces to Implementations](https://arievisser.com/blog/binding-api-connection-interfaces-to-implementations/) — Arie Visser
- [Traits in Laravel Eloquent: Practical Examples](https://laraveldaily.com/post/traits-laravel-eloquent-examples) — Laravel Daily
- [Creating Your Own PHP Helpers in a Laravel Project](https://laravel-news.com/creating-helpers) — Laravel News
- [Contracts](https://laravel.com/docs/contracts) — Laravel Documentation
- [Service Container](https://laravel.com/docs/container) — Laravel Documentation
- [Eloquent: Mutators & Casting](https://laravel.com/docs/eloquent-mutators) — Laravel Documentation
- [Service Layer](https://martinfowler.com/eaaCatalog/serviceLayer.html) — Martin Fowler
