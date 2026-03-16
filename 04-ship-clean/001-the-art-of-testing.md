You have spent the earlier chapters of this book learning how to structure your application cleanly — thin [controllers](/books/clean-code-in-laravel/controllers), focused [Actions](/books/clean-code-in-laravel/actions), typed [DTOs](/books/clean-code-in-laravel/data-transfer-objects), dedicated [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation). Every one of those patterns was chosen for the same reason: they make your code easier to test. A clean architecture and a strong test suite are not separate goals — they reinforce each other.

Testing is not a chore you do after the "real" work is done. It is the mechanism that lets you change code with confidence. Without tests, every refactor is a gamble. Every new feature risks breaking something you cannot see. Every deployment is a prayer. With tests, you have a safety net that catches mistakes before your users do.

This chapter covers how to write clean, effective tests in a Laravel application using [Pest](https://pestphp.com/). We start with what makes a good test, move through practical examples for each pattern in this book, and end with the common pitfalls that make test suites unreliable.

## Why Tests Matter for Clean Code

Clean code is code that is easy to change. Tests are what make change safe. The two are inseparable.

When you extract business logic from a controller into an [Action](/books/clean-code-in-laravel/actions), you are not just organizing code — you are making that logic independently testable. When you move validation into a [Form Request](/books/clean-code-in-laravel/form-requests-and-validation), you can verify your rules without making HTTP requests. When you type your data with [DTOs](/books/clean-code-in-laravel/data-transfer-objects), your tests become clearer because inputs and outputs have explicit shapes.

The patterns in this book produce code that is easy to test by design. This chapter shows you how to take advantage of that.

## Pest: A Modern Testing Framework

[Pest](https://pestphp.com/) is a testing framework built on top of [PHPUnit](https://phpunit.de/) that brings a clean, expressive syntax to PHP testing. It is the default testing framework in Laravel since version 11, and it is what this book uses throughout.

A Pest test looks like this:

```php
it('creates an order from valid data', function (): void {
    $user = User::factory()->create();

    $order = app(CreateOrderAction::class)->execute(
        new OrderData(
            userId: $user->id,
            amountInCents: 5000,
        ),
    );

    expect($order)
        ->toBeInstanceOf(Order::class)
        ->user_id->toBe($user->id)
        ->amount_in_cents->toBe(5000);
});
```

Pest provides two functions for defining tests: `it()` and `test()`. Both do the same thing — the difference is purely stylistic. `it()` reads like a sentence: "it creates an order from valid data." `test()` reads like a label: "test that orders can be created." This book uses `it()` because the sentence style encourages descriptive test names.

### `expect()`

Pest's [expectation API](https://pestphp.com/docs/expectations) replaces PHPUnit's `assert` methods with a chainable, readable syntax:

```php
// PHPUnit style
$this->assertInstanceOf(Order::class, $order);
$this->assertEquals(5000, $order->amount_in_cents);
$this->assertNotNull($order->created_at);

// Pest style
expect($order)
    ->toBeInstanceOf(Order::class)
    ->amount_in_cents->toBe(5000)
    ->created_at->not->toBeNull();
```

Both styles work in Pest, but the `expect()` API is more expressive. You can chain expectations on the same subject without repeating yourself.

## Feature Tests vs. Unit Tests

Laravel generates two test directories: `tests/Feature` and `tests/Unit`. The distinction matters.

A feature test boots the entire Laravel application. It has access to the [service container](https://laravel.com/docs/container), the database, [routes](https://laravel.com/docs/routing), [middleware](https://laravel.com/docs/middleware), and everything else the framework provides. Feature tests verify that your application works correctly from the outside — an HTTP request goes in, and you assert on the response, the database state, or the side effects.

A unit test does not boot the framework. It tests a single class or function in isolation — pure logic with no dependencies on Laravel. Unit tests are fast but limited in scope.

In a Laravel application, most of your tests will be feature tests. This is by design. Laravel's features — Eloquent, the service container, events, queues — are deeply integrated. Testing them in isolation often means mocking so much that the test no longer reflects reality. As [Kent C. Dodds](https://kentcdodds.com/blog/write-tests) put it: "Write tests. Not too many. Mostly integration."

Use unit tests for pure logic that has no framework dependencies — a discount calculator, a data transformer, a state machine transition. Use feature tests for everything that touches the framework.

```bash
# Create a feature test
php artisan make:test CreateOrderTest --pest

# Create a unit test
php artisan make:test DiscountCalculatorTest --pest --unit
```

## What Makes a Good Test

A good test has four qualities:

1. **It tests one behavior.** A test should verify a single behavior or scenario. If your test name contains the word "and," it is probably testing too much. "It creates an order and sends an email and updates inventory" should be three separate tests.
2. **It is self-contained.** A test should set up everything it needs and not rely on state from another test. If test B fails because test A did not run first, both tests are broken.
3. **It has a descriptive name.** The name should describe the scenario and the expected outcome. When a test fails, the name alone should tell you what went wrong. "It returns a 403 when the user is not authorized" is better than "it tests permissions."
4. **It follows the Arrange-Act-Assert pattern.** Set up the preconditions, perform the action, and verify the result. Keep these three sections visually distinct in your test.

```php
it('applies a percentage discount to the order total', function (): void {
    // Arrange
    $coupon = Coupon::factory()->create(['discount_percentage' => 20]);
    $order = Order::factory()->create(['total_in_cents' => 10000]);

    // Act
    $discounted = app(ApplyDiscountAction::class)->execute($order, $coupon);

    // Assert
    expect($discounted->total_in_cents)->toBe(8000);
});
```

## Testing Actions

[Actions](/books/clean-code-in-laravel/actions) are the easiest pattern to test. They accept typed input, return a domain object, and have no knowledge of HTTP. You create the input, call `execute()`, and verify the result.

### Testing the Happy Path

```php
use App\Actions\Order\CreateOrderAction;
use App\DataTransferObjects\OrderData;
use App\Models\Order;
use App\Models\User;

it('creates an order with the correct attributes', function (): void {
    $user = User::factory()->create();

    $data = new OrderData(
        userId: $user->id,
        amountInCents: 7500,
    );

    $order = app(CreateOrderAction::class)->execute($data);

    expect($order)
        ->toBeInstanceOf(Order::class)
        ->user_id->toBe($user->id)
        ->amount_in_cents->toBe(7500);

    $this->assertDatabaseHas('orders', [
        'user_id' => $user->id,
        'amount_in_cents' => 7500,
    ]);
});
```

### Testing Side Effects

Actions often dispatch [events](https://laravel.com/docs/events) or interact with external systems. Use Laravel's fakes to verify side effects without triggering them:

```php
use Illuminate\Support\Facades\Event;
use App\Events\OrderCreated;

it('dispatches an OrderCreated event', function (): void {
    Event::fake();

    $user = User::factory()->create();
    $data = new OrderData(userId: $user->id, amountInCents: 5000);

    app(CreateOrderAction::class)->execute($data);

    Event::assertDispatched(OrderCreated::class, function (OrderCreated $event) use ($user): bool {
        return $event->order->user_id === $user->id;
    });
});
```

### Testing Failure Scenarios

Do not only test the happy path. Test what happens when things go wrong:

```php
use App\Exceptions\FraudDetectedException;
use App\Services\FraudChecker;

it('throws an exception when fraud is detected', function (): void {
    $this->mock(FraudChecker::class)
        ->shouldReceive('isFraudulent')
        ->andReturn(true);

    $user = User::factory()->create();
    $data = new OrderData(userId: $user->id, amountInCents: 5000);

    expect(fn () => app(PlaceOrderAction::class)->execute($data))
        ->toThrow(FraudDetectedException::class);
});
```

## Testing Controllers

Controller tests verify the HTTP layer — that the right status code is returned, the right redirect happens, and the right data appears in the response. Because your [controllers](/books/clean-code-in-laravel/controllers) are thin, these tests are straightforward.

### Testing Web Endpoints

```php
use App\Models\User;
use App\Models\Order;

it('displays the order details page', function (): void {
    $user = User::factory()->create();
    $order = Order::factory()->for($user)->create();

    $response = $this->actingAs($user)
        ->get(route('orders.show', $order));

    $response->assertOk()
        ->assertViewIs('orders.show')
        ->assertViewHas('order', $order);
});

it('redirects to the order after successful creation', function (): void {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->post(route('orders.store'), [
            'amount' => 50,
        ]);

    $order = Order::latest()->first();

    $response->assertRedirect(route('orders.show', $order));
});

it('returns a 403 when the user is not authorized', function (): void {
    $user = User::factory()->create();
    $otherUser = User::factory()->create();
    $order = Order::factory()->for($otherUser)->create();

    $this->actingAs($user)
        ->get(route('orders.show', $order))
        ->assertForbidden();
});
```

### Testing API Endpoints

API tests follow the same pattern but assert on JSON structure instead of views:

```php
use App\Models\User;

it('returns a paginated list of orders as JSON', function (): void {
    $user = User::factory()->create();
    Order::factory()->count(3)->for($user)->create();

    $response = $this->actingAs($user, 'sanctum')
        ->getJson(route('api.orders.index'));

    $response->assertOk()
        ->assertJsonCount(3, 'data')
        ->assertJsonStructure([
            'data' => [
                '*' => ['id', 'status', 'total', 'created_at'],
            ],
        ]);
});
```

## Testing Form Requests

[Form Requests](/books/clean-code-in-laravel/form-requests-and-validation) handle validation and authorization. Test both.

### Testing Validation Rules

The cleanest way to test validation is to make HTTP requests that hit the endpoint using the Form Request. This tests the rules in context — including how they interact with route model binding, authorization, and custom error messages:

```php
it('requires an email address', function (): void {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('orders.store'), [
            'amount' => 50,
            // email is missing
        ])
        ->assertSessionHasErrors('email');
});

it('rejects an invalid email format', function (): void {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('orders.store'), [
            'email' => 'not-an-email',
            'amount' => 50,
        ])
        ->assertSessionHasErrors('email');
});

it('accepts valid input', function (): void {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('orders.store'), [
            'email' => 'customer@example.com',
            'amount' => 50,
        ])
        ->assertSessionDoesntHaveErrors();
});
```

### Testing Authorization

If your Form Request has an `authorize()` method, test that it correctly allows or denies access:

```php
it('allows admins to update any order', function (): void {
    $admin = User::factory()->admin()->create();
    $order = Order::factory()->create();

    $this->actingAs($admin)
        ->put(route('orders.update', $order), [
            'status' => 'shipped',
        ])
        ->assertSessionDoesntHaveErrors();
});

it('prevents regular users from updating orders they do not own', function (): void {
    $user = User::factory()->create();
    $order = Order::factory()->create(); // belongs to another user

    $this->actingAs($user)
        ->put(route('orders.update', $order), [
            'status' => 'shipped',
        ])
        ->assertForbidden();
});
```

## Testing Eloquent Models

[Model](https://laravel.com/docs/eloquent) tests verify relationships, scopes, casts, and custom methods. These are feature tests because they need the database.

### Testing Relationships

```php
use App\Models\Order;
use App\Models\User;

it('belongs to a user', function (): void {
    $user = User::factory()->create();
    $order = Order::factory()->for($user)->create();

    expect($order->user)
        ->toBeInstanceOf(User::class)
        ->id->toBe($user->id);
});

it('has many order items', function (): void {
    $order = Order::factory()
        ->hasItems(3)
        ->create();

    expect($order->items)->toHaveCount(3);
});
```

### Testing Scopes

```php
it('filters active users', function (): void {
    User::factory()->count(2)->create(['active' => true]);
    User::factory()->count(3)->create(['active' => false]);

    expect(User::active()->count())->toBe(2);
});
```

### Testing Casts and Accessors

```php
it('casts the status to an enum', function (): void {
    $order = Order::factory()->create(['status' => 'pending']);

    expect($order->status)->toBe(OrderStatus::Pending);
});

it('returns the total in dollars', function (): void {
    $order = Order::factory()->create(['total_in_cents' => 2599]);

    expect($order->total_in_dollars)->toBe(25.99);
});
```

## Testing Jobs

[Jobs](/books/clean-code-in-laravel/jobs) are tested in two ways: verify that the job is dispatched under the right conditions, and verify that the job's `handle()` method does the right thing when executed.

### Testing Job Dispatch

```php
use Illuminate\Support\Facades\Queue;
use App\Jobs\ProcessOrderPaymentJob;

it('dispatches a payment job when an order is placed', function (): void {
    Queue::fake();

    $user = User::factory()->create();
    $data = new OrderData(userId: $user->id, amountInCents: 5000);

    app(PlaceOrderAction::class)->execute($data);

    Queue::assertPushed(ProcessOrderPaymentJob::class, function (ProcessOrderPaymentJob $job): bool {
        return $job->order->amount_in_cents === 5000;
    });
});
```

### Testing Job Execution

```php
it('charges the customer and updates the order status', function (): void {
    $order = Order::factory()->create([
        'payment_status' => 'pending',
        'amount_in_cents' => 5000,
    ]);

    $this->mock(PaymentGateway::class)
        ->shouldReceive('charge')
        ->once()
        ->andReturn(new PaymentResult(transactionId: 'txn_123'));

    (new ProcessOrderPaymentJob($order))->handle(
        app(PaymentGateway::class),
    );

    expect($order->fresh())
        ->payment_status->toBe('paid')
        ->transaction_id->toBe('txn_123');
});
```

## Laravel's Fakes

One of Laravel's most powerful testing features is its built-in [fakes](https://laravel.com/docs/mocking). Fakes replace real implementations with test doubles that record what happened without performing the actual work. They are cleaner than traditional mocks because they are first-party and understand Laravel's conventions.

### When to Use Fakes

Use fakes when you want to verify that your code triggers a side effect without actually performing it:

| Facade | Fake | What It Captures |
|---|---|---|
| `Event::fake()` | [Event fake](https://laravel.com/docs/events#testing) | Dispatched events |
| `Queue::fake()` | [Queue fake](https://laravel.com/docs/queues#testing) | Pushed jobs |
| `Mail::fake()` | [Mail fake](https://laravel.com/docs/mail#testing) | Sent mailables |
| `Notification::fake()` | [Notification fake](https://laravel.com/docs/notifications#testing) | Sent notifications |
| `Storage::fake()` | [Storage fake](https://laravel.com/docs/filesystem#testing) | File operations |
| `Http::fake()` | [HTTP fake](https://laravel.com/docs/http-client#testing) | Outgoing HTTP requests |
| `Bus::fake()` | [Bus fake](https://laravel.com/docs/queues#testing) | Dispatched jobs and chains |

### Faking Events

```php
use Illuminate\Support\Facades\Event;

it('dispatches OrderShipped when an order is marked as shipped', function (): void {
    Event::fake();

    $order = Order::factory()->create();

    app(ShipOrderAction::class)->execute($order, trackingNumber: 'TRACK123');

    Event::assertDispatched(OrderShipped::class, function (OrderShipped $event): bool {
        return $event->order->id === $order->id
            && $event->trackingNumber === 'TRACK123';
    });
});
```

### Faking Mail

```php
use Illuminate\Support\Facades\Mail;
use App\Mail\OrderReceipt;

it('sends a receipt email after payment', function (): void {
    Mail::fake();

    $order = Order::factory()->create(['email' => 'customer@example.com']);

    app(SendOrderReceiptAction::class)->execute($order);

    Mail::assertSent(OrderReceipt::class, function (OrderReceipt $mail): bool {
        return $mail->hasTo('customer@example.com');
    });
});
```

### Faking HTTP Requests

When your code calls external APIs, use `Http::fake()` to stub the responses:

```php
use Illuminate\Support\Facades\Http;

it('checks for fraud via the external API', function (): void {
    Http::fake([
        'fraud-check.service/*' => Http::response(['is_fraud' => false]),
    ]);

    $result = app(FraudChecker::class)->isFraudulent('customer@example.com');

    expect($result)->toBeFalse();

    Http::assertSent(function ($request): bool {
        return $request['email'] === 'customer@example.com';
    });
});
```

### Fakes vs. Mocks

Fakes and mocks serve different purposes. Laravel's fakes are pre-built test doubles for framework services — they are the right tool when you are testing interactions with events, queues, mail, notifications, storage, or HTTP. Use [Mockery](https://github.com/mockery/mockery) mocks (via `$this->mock()`) when you need to stub your own classes — Services, third-party SDKs, or any dependency you want to isolate.

A practical rule: if a `Facade::fake()` exists for what you are testing, use it. If not, use `$this->mock()`.

## Database Testing

Most feature tests need a database. Laravel provides two strategies for managing test data: [RefreshDatabase](https://laravel.com/docs/database-testing#resetting-the-database-after-each-test) and [DatabaseTransactions](https://laravel.com/docs/database-testing#resetting-the-database-after-each-test).

`RefreshDatabase` runs your migrations once and wraps each test in a transaction that is rolled back after the test completes. It is the default choice and works for the vast majority of test suites:

```php
// tests/TestCase.php or tests/Pest.php
uses(Illuminate\Foundation\Testing\RefreshDatabase::class);
```

### Use Factories, Not Manual Inserts

Always use [model factories](https://laravel.com/docs/eloquent-factories) to create test data. Factories are expressive, maintainable, and produce realistic data:

```php
// Before: manual insert — brittle, verbose, missing required columns
$user = User::create([
    'name' => 'John',
    'email' => 'john@example.com',
    'password' => bcrypt('password'),
]);

// After: factory — fills all required fields automatically
$user = User::factory()->create();

// After: factory with specific overrides
$user = User::factory()->create(['email' => 'john@example.com']);
```

Factories support states for common scenarios. Check if the factory already defines the state you need before setting attributes manually:

```php
// Define states in the factory
class UserFactory extends Factory
{
    public function admin(): static
    {
        return $this->state(fn (array $attributes): array => [
            'role' => 'admin',
        ]);
    }

    public function suspended(): static
    {
        return $this->state(fn (array $attributes): array => [
            'suspended_at' => now(),
        ]);
    }
}

// Use states in tests
$admin = User::factory()->admin()->create();
$suspended = User::factory()->suspended()->create();
```

## Test Organization and Naming

Organize tests to mirror your application structure. When a developer changes a class, they should know exactly where to find the corresponding test:

```
tests/
├── Feature/
│   ├── Actions/
│   │   └── Order/
│   │       ├── CreateOrderActionTest.php
│   │       └── PlaceOrderActionTest.php
│   ├── Http/
│   │   └── Controllers/
│   │       └── OrderControllerTest.php
│   ├── Jobs/
│   │   └── ProcessOrderPaymentJobTest.php
│   └── Models/
│       ├── OrderTest.php
│       └── UserTest.php
└── Unit/
    └── Services/
        └── DiscountCalculatorTest.php
```

### Naming Tests

Test names should describe the scenario and the expected outcome. Read the test name as a sentence:

```php
// After: describes the scenario and expected outcome
it('creates an order with the correct total')
it('returns a 404 when the order does not exist')
it('sends a receipt email to the customer')
it('prevents non-admins from deleting orders')
it('applies a 20% discount when the coupon is valid')

// Before: vague, does not describe what is being tested
it('works correctly')
it('test order')
it('handles edge case')
```

### Grouping with `describe()`

Use Pest's `describe()` blocks to group related tests when a file covers multiple scenarios for the same subject:

```php
describe('CreateOrderAction', function (): void {
    it('creates an order with the correct attributes', function (): void {
        // ...
    });

    it('dispatches an OrderCreated event', function (): void {
        // ...
    });

    it('wraps the operation in a database transaction', function (): void {
        // ...
    });
});
```

## Common Testing Pitfalls

### Testing Implementation Instead of Behavior

Test what your code does, not how it does it. If you refactor the internals without changing the behavior, your tests should still pass:

```php
// Before: coupled to implementation details
it('calls the repository save method', function (): void {
    $repo = $this->mock(OrderRepository::class);
    $repo->shouldReceive('save')->once();

    app(CreateOrderAction::class)->execute($data);
});

// After: tests the observable outcome
it('persists the order to the database', function (): void {
    app(CreateOrderAction::class)->execute($data);

    $this->assertDatabaseHas('orders', [
        'user_id' => $data->userId,
        'amount_in_cents' => $data->amountInCents,
    ]);
});
```

### Over-Mocking

When you mock everything, your test only proves that your mocks work — not that your code works. Mock external boundaries (APIs, third-party services) but let your own code run:

```php
// Before: mocking Eloquent — what are you even testing?
$this->mock(Order::class)
    ->shouldReceive('create')
    ->andReturn(new Order());

// After: let Eloquent do its job, assert on the database
$order = app(CreateOrderAction::class)->execute($data);

$this->assertDatabaseHas('orders', ['id' => $order->id]);
```

### Asserting Too Much

A test that asserts every field on every object is brittle. It breaks whenever you add a column, even if the behavior under test has not changed. Assert on what matters for the behavior you are testing:

```php
// Before: asserts everything — breaks when you add a field
expect($order->toArray())->toBe([
    'id' => 1,
    'user_id' => 1,
    'amount_in_cents' => 5000,
    'status' => 'pending',
    'created_at' => '...',
    'updated_at' => '...',
]);

// After: asserts only what this test cares about
expect($order)
    ->user_id->toBe($user->id)
    ->amount_in_cents->toBe(5000);
```

### Not Testing Edge Cases

Happy-path-only test suites give a false sense of security. Think about boundary conditions:

- What happens with zero items?
- What if the user does not exist?
- What if the amount is negative?
- What happens when the external service is down?

```php
it('rejects an order with zero amount', function (): void {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('orders.store'), ['amount' => 0])
        ->assertSessionHasErrors('amount');
});

it('handles a failed payment gracefully', function (): void {
    $this->mock(PaymentGateway::class)
        ->shouldReceive('charge')
        ->andThrow(new PaymentFailedException('Card declined'));

    $order = Order::factory()->create();

    expect(fn () => (new ProcessOrderPaymentJob($order))->handle(app(PaymentGateway::class)))
        ->toThrow(PaymentFailedException::class);

    expect($order->fresh()->payment_status)->toBe('pending');
});
```

### Slow Tests

A slow test suite is a test suite nobody runs. Keep tests fast by:

- Using `RefreshDatabase` instead of re-running migrations for every test
- Using `Queue::fake()`, `Mail::fake()`, and other fakes instead of processing real jobs or sending real emails
- Only creating the database records you need — do not seed the entire database for every test
- Using `Http::fake()` instead of making real HTTP requests

## Running Tests

Run your tests frequently — ideally after every change. Pest makes this easy:

```bash
# Run all tests
php artisan test --compact

# Run a specific test file
php artisan test --compact tests/Feature/Actions/Order/CreateOrderActionTest.php

# Run tests matching a name
php artisan test --compact --filter="creates an order"

# Run only tests that failed last time
php artisan test --compact --retry
```

The `--compact` flag gives you a condensed output that shows just the results. Use `--filter` to run a subset of tests while you are working on a specific feature.

## Summary

- Tests are not optional — they are what make clean code maintainable. Without tests, every change is a risk. With tests, you refactor with confidence.
- Use [Pest](https://pestphp.com/) for clean, expressive tests. The `it()` syntax and `expect()` API produce tests that read like specifications.
- Most Laravel tests should be feature tests. They boot the framework, hit the database, and verify behavior in context. Reserve unit tests for pure logic with no framework dependencies.
- Every test should test one behavior, be self-contained, have a descriptive name, and follow the Arrange-Act-Assert pattern.
- [Actions](/books/clean-code-in-laravel/actions) are the easiest code to test. Create the input, call `execute()`, verify the result. No HTTP setup needed.
- Test [controllers](/books/clean-code-in-laravel/controllers) through HTTP. Assert on status codes, redirects, view data, and JSON structure — not on internal implementation.
- Test [Form Requests](/books/clean-code-in-laravel/form-requests-and-validation) by making HTTP requests that trigger validation. Assert on `assertSessionHasErrors` and `assertSessionDoesntHaveErrors`.
- Use Laravel's built-in [fakes](https://laravel.com/docs/mocking) for events, queues, mail, notifications, storage, and HTTP. Use [Mockery](https://github.com/mockery/mockery) mocks for your own classes and third-party SDKs.
- Use [model factories](https://laravel.com/docs/eloquent-factories) to create test data. Factories are expressive, maintainable, and leverage states for common scenarios.
- Test behavior, not implementation. If your tests break when you refactor internals without changing behavior, they are too coupled.
- Test edge cases, not just the happy path. Zero values, missing records, failed external calls, and unauthorized access are all scenarios your tests should cover.
- Keep tests fast. Use fakes, create only the data you need, and use `RefreshDatabase` to avoid re-running migrations.

## References

- [Pest Documentation](https://pestphp.com/docs) — Pest
- [Expectations API](https://pestphp.com/docs/expectations) — Pest
- [Testing: Getting Started](https://laravel.com/docs/testing) — Laravel Documentation
- [HTTP Tests](https://laravel.com/docs/http-tests) — Laravel Documentation
- [Database Testing](https://laravel.com/docs/database-testing) — Laravel Documentation
- [Mocking](https://laravel.com/docs/mocking) — Laravel Documentation
- [Eloquent Factories](https://laravel.com/docs/eloquent-factories) — Laravel Documentation
- [Testing Tips for Laravel](https://www.youtube.com/watch?v=vej0XCjBBMo) — Nuno Maduro, Laracon EU 2024
- [Write Tests. Not Too Many. Mostly Integration.](https://kentcdodds.com/blog/write-tests) — Kent C. Dodds
- [Test Desiderata](https://testdesiderata.com/) — Kent Beck
- [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) — Martin Fowler
- [Mockery Documentation](https://docs.mockery.io/) — Mockery
