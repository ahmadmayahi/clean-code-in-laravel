[Dependency injection](https://laravel.com/docs/container#introduction) is a simple idea with a fancy name. Instead of a class creating the things it needs, you pass them in from the outside. That is it. The class says "I need this" and someone else provides it.

Here is a controller that creates its own dependency:

```php
class OrderController extends Controller
{
    public function store(StoreOrderRequest $request): RedirectResponse
    {
        // The controller creates its own dependency
        $service = new OrderService();
        $order = $service->placeOrder($request->toDto());

        return redirect()->route('orders.show', $order);
    }
}
```

And here is the same controller with the dependency injected:

```php
class OrderController extends Controller
{
    public function store(
        StoreOrderRequest $request,
        OrderService $service,
    ): RedirectResponse {
        $order = $service->placeOrder($request->toDto());

        return redirect()->route('orders.show', $order);
    }
}
```

The difference is one line - `new OrderService()` is gone. But that one line changes everything about how testable, flexible, and maintainable your code is.

## Why `new` Is a Problem

When you write `new OrderService()` inside a class, you are making a hard decision that cannot be changed from outside. The class is permanently tied to that specific implementation. This creates three problems:

1. You cannot swap it in tests.

If `OrderService` talks to Stripe, sends emails, or queries the database, your test has no way to replace it with a fake. You are stuck testing the real thing:

```php
// This controller ALWAYS uses the real OrderService
// You cannot replace it with a mock or fake
class OrderController extends Controller
{
    public function store(StoreOrderRequest $request): RedirectResponse
    {
        $service = new OrderService(); // Hardcoded - no way to swap
        $order = $service->placeOrder($request->toDto());

        return redirect()->route('orders.show', $order);
    }
}
```

2. You cannot change behavior without editing the class.

If you need a different `OrderService` in a different context - maybe a `DiscountedOrderService` during a sale - you have to open the controller and change the code. With injection, you just bind a different class in the container.

3. You hide what the class depends on.

When dependencies are created with `new` inside methods, you have to read the entire class to figure out what it needs. When they are listed in the constructor, you can see all dependencies at a glance:

```php
// Clear: you can see all dependencies without reading the method bodies
class PlaceOrderAction
{
    public function __construct(
        private readonly OrderService $orderService,
        private readonly PaymentGateway $paymentGateway,
        private readonly InventoryService $inventoryService,
    ) {}
}
```

Given all of this, do not waste time debating whether to use `new` or the container. Always use dependency injection for services. The cost of injecting something that could have been `new`ed is zero — the container resolves it automatically, no performance penalty, no extra complexity. But the cost of `new`ing something that should have been injected is real: the moment you need to test it, swap it, or change its behavior, you have to refactor. Make this a non-negotiable rule on your team and move on. Spend your meeting time on problems that actually matter.

## Dependency Injection in Plain PHP

Before looking at Laravel's tools, it helps to understand that dependency injection is not a framework feature - it is a plain PHP pattern. You can use it without any framework at all.

Imagine a `UserService` that sends welcome emails. Without injection, the class creates its own mailer:

```php
class UserService
{
    public function register(string $email, string $name): void
    {
        $user = new User($email, $name);
        $user->save();

        // Hardcoded - this class decides HOW to send email
        $mailer = new SmtpMailer('smtp.example.com', 587);
        $mailer->send($email, 'Welcome!', "Hello {$name}");
    }
}
```

This works, but `UserService` is now permanently tied to `SmtpMailer`. You cannot test it without an SMTP server. You cannot switch to a different mailer without editing this class. The dependency is hidden inside the method.

With injection, you pass the mailer in from outside:

```php
class UserService
{
    public function __construct(
        private readonly MailerInterface $mailer,
    ) {}

    public function register(string $email, string $name): void
    {
        $user = new User($email, $name);
        $user->save();

        // The class does not know or care HOW the email is sent
        $this->mailer->send($email, 'Welcome!', "Hello {$name}");
    }
}

// In production
$service = new UserService(new SmtpMailer('smtp.example.com', 587));

// In tests
$service = new UserService(new FakeMailer());
```

The `UserService` no longer knows what kind of mailer it is using. It just knows it has something that can send emails. That is dependency injection - the caller decides what to provide, not the class itself.

This pattern works in any PHP project. But creating objects by hand gets tedious fast. In a real application, classes depend on other classes, which depend on other classes. Building the full chain manually is painful:

```php
// Building dependencies by hand - this gets old quickly
$httpClient = new HttpClient();
$paymentGateway = new StripePaymentGateway($httpClient);
$inventoryService = new InventoryService();
$orderService = new OrderService($paymentGateway, $inventoryService);
$controller = new OrderController($orderService);
```

That is where a container comes in.

## What Is a Container?

A container is an object that knows how to build other objects. Instead of manually creating dependencies and wiring them together, you tell the container "I need an `OrderService`" and it figures out the rest - what `OrderService` needs, what those dependencies need, and so on, all the way down.

Think of it like a factory that reads blueprints. You ask for the finished product, and it assembles all the parts for you.

Here is a simplified version of how a container works under the hood:

```php
class Container
{
    private array $bindings = [];

    public function bind(string $abstract, string $concrete): void
    {
        $this->bindings[$abstract] = $concrete;
    }

    public function make(string $class): object
    {
        // If there is a binding, use it
        if (isset($this->bindings[$class])) {
            $class = $this->bindings[$class];
        }

        // Read the constructor to find out what it needs
        $reflection = new ReflectionClass($class);
        $constructor = $reflection->getConstructor();

        if ($constructor === null) {
            return new $class();
        }

        // Resolve each parameter recursively
        $dependencies = [];
        foreach ($constructor->getParameters() as $parameter) {
            $type = $parameter->getType()->getName();
            $dependencies[] = $this->make($type); // Recursive!
        }

        return $reflection->newInstanceArgs($dependencies);
    }
}
```

You would use it like this:

```php
$container = new Container();
$container->bind(PaymentGateway::class, StripePaymentGateway::class);

// The container builds StripePaymentGateway, HttpClient, and everything else
$service = $container->make(OrderService::class);
```

You do not need to build your own container - this is just to show that there is no magic involved. It is PHP reflection and recursion. Laravel's [service container](https://laravel.com/docs/container) does the same thing, with a lot more features.

## Laravel's Service Container

Laravel's service container is the engine behind the entire framework. Every time Laravel creates a controller, resolves a [Form Request](https://laravel.com/docs/validation#form-request-validation), runs a [job](https://laravel.com/docs/queues), or boots a [service provider](https://laravel.com/docs/providers), it uses the container.

The good news is that most of the time you do not need to configure anything. If `OrderService` has no special requirements, Laravel just creates it:

```php
// You never write this - the container does it automatically
$service = new OrderService();

// You just type-hint it and Laravel handles the rest
public function __construct(private readonly OrderService $service) {}
```

This is called [automatic resolution](https://laravel.com/docs/container#zero-configuration-resolution). The container reads the constructor, sees what type each parameter expects, and creates the entire chain of dependencies recursively. If `OrderService` depends on `PaymentGateway`, which depends on `HttpClient`, the container builds all three - no configuration needed.

## Constructor Injection

[Constructor injection](https://laravel.com/docs/container#automatic-injection) is the most common pattern. You declare your dependencies as constructor parameters, and Laravel injects them when the class is created:

```php
class PlaceOrderAction
{
    public function __construct(
        private readonly OrderService $orderService,
        private readonly PaymentGateway $paymentGateway,
    ) {}

    public function handle(OrderData $data): Order
    {
        $order = $this->orderService->create($data);
        $this->paymentGateway->charge($order);

        return $order;
    }
}
```

Use constructor injection when the dependency is used across multiple methods, or when the class cannot function without it. The `readonly` keyword prevents the dependency from being reassigned after construction — a good safety net. We use this pattern extensively in the [Actions](/books/clean-code-in-laravel/actions) chapter.

## Method Injection

In [controllers](https://laravel.com/docs/controllers#dependency-injection-and-controllers), you can also inject dependencies directly into a method. Laravel resolves them the same way:

```php
class OrderController extends Controller
{
    public function store(
        StoreOrderRequest $request,
        PlaceOrderAction $action,
    ): RedirectResponse {
        $order = $action->handle($request->toDto());

        return redirect()->route('orders.show', $order);
    }

    public function index(OrderService $service): View
    {
        return view('orders.index', [
            'orders' => $service->getOrdersForUser(auth()->user()),
        ]);
    }
}
```

Method injection is useful when a dependency is only needed for one specific action. If `PlaceOrderAction` is only used in the `store` method, there is no reason to inject it into the constructor and make it available to every method. This is exactly how we inject [Actions](/books/clean-code-in-laravel/actions) into [thin controllers](/books/clean-code-in-laravel/thin-controllers).

A practical guideline: if a dependency is used in two or more methods, put it in the constructor. If it is used in one method, inject it into that method.

## Binding Interfaces to Implementations

Sometimes you want to type-hint an interface instead of a concrete class. This is called [binding](https://laravel.com/docs/container#binding), and it is useful when you have different implementations for different contexts - or when you want to swap an implementation in tests.

Tell the container which class to use for a given interface in a [service provider](https://laravel.com/docs/providers):

```php
// app/Providers/AppServiceProvider.php
use App\Contracts\PaymentGateway;
use App\Services\StripePaymentGateway;

public function register(): void
{
    $this->app->bind(PaymentGateway::class, StripePaymentGateway::class);
}
```

Now anywhere you type-hint `PaymentGateway`, the container gives you `StripePaymentGateway`:

```php
class PlaceOrderAction
{
    public function __construct(
        private readonly PaymentGateway $gateway, // Resolves to StripePaymentGateway
    ) {}
}
```

In tests, you can swap it out:

```php
$this->app->bind(PaymentGateway::class, FakePaymentGateway::class);
```

Do not reach for interfaces by default. As we discussed in the [YAGNI chapter](/books/clean-code-in-laravel/the-philosophy-of-simplicity#you-are-not-going-to-need-it), only create an interface when you have a real reason — multiple implementations, or a clear testing need. A concrete class works fine until then.

## Using `app()` to Resolve

Sometimes you need to resolve a class outside of a constructor or controller method - for example, in a helper function or a place where Laravel does not automatically inject dependencies. Use the [`app()` helper](https://laravel.com/docs/container#the-make-method):

```php
$service = app(OrderService::class);
```

This asks the container to build `OrderService` the same way it would for constructor injection. It respects all your bindings and resolves the full dependency chain.

You can also use `app()->make()`, which does the same thing:

```php
$service = app()->make(OrderService::class);
```

Use `app()` sparingly. It is a [service locator](https://en.wikipedia.org/wiki/Service_locator_pattern) - instead of declaring dependencies upfront, you reach into the container at runtime. This hides dependencies and makes code harder to follow. Prefer constructor or method injection wherever possible. Reserve `app()` for places where injection is not available.

That said, not everything needs to come from the container. Simple value objects, DTOs, and data structures are perfectly fine to create with `new`:

```php
// These are fine — they are plain data, not services
$dto = new OrderData(
    items: $request->items,
    shippingAddressId: $request->shipping_address_id,
);

$money = new Money(amount: 2500, currency: 'usd');

$period = new DateRange(
    from: now()->subDays(30),
    to: now(),
);
```

The rule of thumb: if the object does something (calls an API, queries a database, sends an email), inject it. If the object holds data, create it with `new`.

## Testing with Dependency Injection

The biggest practical benefit of dependency injection is [testability](https://laravel.com/docs/mocking). When dependencies are injected, you can replace them with fakes or mocks in your tests:

```php
use App\Contracts\PaymentGateway;

it('places an order', function (): void {
    // Swap the real payment gateway with a fake
    $this->app->bind(PaymentGateway::class, FakePaymentGateway::class);

    $response = $this->post('/orders', [
        'items' => [
            ['product_id' => 1, 'quantity' => 2],
        ],
        'shipping_address_id' => 1,
    ]);

    $response->assertRedirect();
    expect(Order::count())->toBe(1);
});
```

Without injection, you would have to hit the real Stripe API in every test. With injection, the test is fast, isolated, and reliable.

You can also use [Mockery](https://github.com/mockery/mockery) to create a mock on the fly:

```php
use App\Services\OrderService;

it('delegates to the order service', function (): void {
    $mock = Mockery::mock(OrderService::class);
    $mock->shouldReceive('placeOrder')->once()->andReturn(new Order());

    $this->app->instance(OrderService::class, $mock);

    $this->post('/orders', [/* ... */]);
});
```

The container makes all of this possible. Because the controller asks for `OrderService` instead of creating it, you can give it whatever you want in your tests.

## Summary

- Dependency injection means passing dependencies in instead of creating them. A class declares what it needs, and something else provides it.
- Avoid `new` for services. Always inject them. The cost of injecting something you could have `new`ed is zero. The cost of `new`ing something you should have injected is refactoring later. Do not debate it — make it a non-negotiable rule.
- Laravel's service container resolves dependencies automatically. Type-hint a class in a constructor or method, and the container builds it - including all of its own dependencies.
- Use constructor injection for shared dependencies and method injection for action-specific ones. If a dependency is used in two or more methods, put it in the constructor.
- Bind interfaces to implementations in a service provider when you need to swap classes - but do not create interfaces until you have a real reason.
- Use `app()` sparingly. It works, but it hides dependencies. Prefer constructor or method injection wherever possible.
- `new` is fine for data. DTOs, value objects, and plain data structures do not need the container. If it holds data, use `new`. If it does something, inject it.
- The biggest win is testability. When dependencies are injected, you can replace them with fakes in tests. Fast, isolated, reliable.
