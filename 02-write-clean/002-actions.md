In the [previous chapter](/books/clean-code-in-laravel/controllers), we established that a controller's only job is to receive an HTTP [request](https://laravel.com/docs/requests) and return a [response](https://laravel.com/docs/responses). But if the controller is not doing the work, something else has to. In a typical Laravel application, business logic ends up in one of two places: bloated controllers or catch-all service classes. Both lead to code that is difficult to test, reuse, and maintain.

The Action Pattern offers a better alternative. An Action is a single-purpose class that performs one discrete business operation. It is the most popular pattern in the Laravel community for keeping controllers thin and business logic organized.

## Fat Controllers

Consider a controller that handles placing an order:

```php
class OrderController extends Controller
{
    public function store(Request $request): RedirectResponse
    {
        $request->validate([
            'email' => ['required', 'email'],
            'amount' => ['required', 'numeric', 'min:1'],
        ]);

        // 1. Check for fraud
        $isFraud = Http::post('https://fraud-check.service', [
            'email' => $request->email,
        ])->json('is_fraud');

        if ($isFraud) {
            return back()->withErrors('Your order could not be processed.');
        }

        // 2. Create the order
        $order = new Order();
        $order->email = $request->email;
        $order->amount = $request->amount * 100;
        $order->save();

        // 3. Send an email receipt
        Mail::to($request->email)->send(new OrderReceipt($order));

        // 4. Notify the sales team
        Notification::route('slack', '#sales')
            ->notify(new NewOrderNotification($order));

        return redirect()->route('orders.show', $order);
    }
}
```

This controller is doing five different things: [validation](https://laravel.com/docs/validation), fraud checking, database persistence, [email](https://laravel.com/docs/mail) sending, and [notification](https://laravel.com/docs/notifications) dispatching. It is difficult to test any one of these operations in isolation. And if you need to create an order from a console command or a queued job, you would have to duplicate all of this logic.

## What is an Action?

An Action is a class with a single public method - typically called `execute()` - that performs one discrete business operation. The key characteristics of a good Action are:

1. **It has one purpose.** An Action should represent a single, well-defined business operation. That operation may involve multiple steps — a `PlaceOrderAction` might orchestrate fraud checking, order creation, and inventory reservation. That is still one cohesive operation: "place an order." What you want to avoid is cramming unrelated operations into one Action. If sending a receipt is a side effect rather than a core part of placing the order, handle it through an [event listener](https://laravel.com/docs/events) instead of baking it into the Action.
2. **It accepts typed input.** An Action receives its input as a [DTO](/books/clean-code-in-laravel/data-transfer-objects) or as typed parameters. It never accesses the HTTP [request](https://laravel.com/docs/requests) directly. This decoupling is what makes Actions reusable across controllers, commands, and jobs. We will cover DTOs in depth in the [Data Transfer Objects](/books/clean-code-in-laravel/data-transfer-objects) chapter.
3. **It returns a domain object or void.** An Action that creates or retrieves something returns a domain object (like an [Eloquent model](https://laravel.com/docs/eloquent)). An Action that performs a side effect — publishing a post, sending a notification — returns void. It never returns an HTTP [response](https://laravel.com/docs/responses).
4. **It is context-agnostic.** An Action has no knowledge of HTTP, [views](https://laravel.com/docs/views), or [redirects](https://laravel.com/docs/responses#redirects). It can be called from a [controller](https://laravel.com/docs/controllers), a [console command](https://laravel.com/docs/artisan), a [queued job](https://laravel.com/docs/queues), or a test - and it will behave exactly the same way every time.

## Refactoring to Actions

Let us refactor the fat controller above into a set of focused Actions.

1. Create Order Action:

```php
// src/Domain/Order/Actions/CreateOrderAction.php
namespace Domain\Order\Actions;

use Domain\Order\Data\OrderData;
use Domain\Order\Events\OrderCreated;
use Domain\Order\Models\Order;

class CreateOrderAction
{
    public function execute(OrderData $data): Order
    {
        $order = Order::create([
            'email' => $data->email,
            'amount_in_cents' => $data->amount_in_cents,
        ]);

        event(new OrderCreated($order));

        return $order;
    }
}
```

2. Fraud Checker Service — notice this is a Service, not an Action. Fraud checking wraps an external API — it is something the application *uses*, not a business operation the application *does*. We cover the distinction between Actions and Services in detail in [Organizing Your Application](/books/clean-code-in-laravel/organizing-your-application):

```php
// src/Domain/Order/Services/FraudChecker.php
namespace Domain\Order\Services;

use Illuminate\Support\Facades\Http; // https://laravel.com/docs/http-client

class FraudChecker
{
    public function isFraudulent(string $email): bool
    {
        return Http::post('https://fraud-check.service', [
            'email' => $email,
        ])->json('is_fraud', false);
    }
}
```

3. Thin Controller

```php
// app/Http/Controllers/OrderController.php
class OrderController extends Controller
{
    public function store(
        OrderData $data,
        FraudChecker $fraudChecker,
        CreateOrderAction $createOrder,
    ): RedirectResponse {
        if ($fraudChecker->isFraudulent($data->email)) {
            return back()->withErrors('Your order could not be processed.');
        }

        $order = $createOrder->execute($data);

        return redirect()->route('orders.show', $order);
    }
}
```

The controller is now thin and focused. It coordinates the flow - check for fraud, then create the order - but it does not contain any business logic itself. The Action can be [tested](/books/clean-code-in-laravel/the-art-of-testing) independently, the Service can be mocked in tests, and the email notification is handled by a [listener](https://laravel.com/docs/events) on the `OrderCreated` event.

## Composing Actions

Actions can call other Actions and use Services. This is how you build complex workflows from simple, reusable building blocks:

```php
// src/Domain/Order/Actions/PlaceOrderAction.php
namespace Domain\Order\Actions;

use Domain\Cart\Models\Cart;
use Domain\Order\Data\OrderData;
use Domain\Order\Models\Order;
use Domain\Order\Services\FraudChecker;

class PlaceOrderAction
{
    public function __construct(
        private readonly FraudChecker $fraudChecker,
        private readonly CreateOrderAction $createOrder,
        private readonly CreateOrderItemsAction $createItems,
    ) {}

    public function execute(Cart $cart, OrderData $data): Order
    {
        if ($this->fraudChecker->isFraudulent($data->email)) {
            throw new FraudDetectedException($data->email);
        }

        $order = $this->createOrder->execute($data);
        $this->createItems->execute($order, $cart->items);

        return $order;
    }
}
```

The `PlaceOrderAction` composes two smaller Actions and one Service. The fraud checker is a Service because it wraps an external API. The order creation and item creation are Actions because they perform business operations. Each piece is independently testable, and Laravel's [service container](https://laravel.com/docs/container) automatically resolves the dependencies via constructor injection.

## Making Actions Queueable

Sometimes an Action performs a time-consuming operation - generating a PDF, processing a large file, or calling a slow external API. In these cases, you want the option to run the Action asynchronously in a [queue](https://laravel.com/docs/queues) without duplicating code.

The [`spatie/laravel-queueable-action`](https://github.com/spatie/laravel-queueable-action) package solves this elegantly. It lets you keep the standard Action structure with an `execute()` method while adding the ability to dispatch it to a queue:

```php
// src/Domain/Invoice/Actions/GenerateInvoicePdfAction.php
namespace Domain\Invoice\Actions;

use Domain\Invoice\Models\Invoice;
use Spatie\QueueableAction\QueueableAction;

class GenerateInvoicePdfAction
{
    use QueueableAction;

    public function execute(Invoice $invoice): void
    {
        $pdf = Pdf::loadView('invoices.pdf', ['invoice' => $invoice]);
        Storage::put("invoices/{$invoice->id}.pdf", $pdf->output());

        $invoice->update(['pdf_generated_at' => now()]);
    }
}
```

You can run the Action synchronously as usual:

```php
$action = app(GenerateInvoicePdfAction::class);
$action->execute($invoice);
```

Or dispatch it to the queue using `onQueue()`:

```php
$action = app(GenerateInvoicePdfAction::class);
$action->onQueue()->execute($invoice);
```

You can also specify a queue name, connection, or delay:

```php
$action->onQueue('pdfs')
    ->onConnection('redis')
    ->execute($invoice);
```

The beauty of this approach is that the Action itself stays clean - it has no knowledge of queues, jobs, or serialization. The `QueueableAction` trait handles all of that behind the scenes. Your Action remains a simple class with a single `execute()` method, whether it runs synchronously or asynchronously.

## Actions vs. Services

You might be wondering how Actions differ from the "Service" classes. The confusion is understandable - both live outside of controllers and both contain business logic. But they serve fundamentally different purposes. We cover Services in depth in [Organizing Your Application](/books/clean-code-in-laravel/organizing-your-application), but here is the key distinction.

An Action performs a single, discrete operation. It has one public method - `execute()` - and it does one thing. Think of it as a verb: `CreateOrder`, `SendInvoice`, `SuspendUser`.

A [Service](/books/clean-code-in-laravel/organizing-your-application#services) provides a stateless utility or wraps an external integration. It exposes multiple methods that share a common dependency or configuration. Think of it as a tool: `PaymentGateway`, `Geocoder`, `CurrencyConverter`.

| Aspect             | Action                           | Service                            |
| ------------------ | -------------------------------- | ---------------------------------- |
| Purpose        | Perform a business operation     | Provide a utility or integration   |
| Methods        | Single `execute()`               | Multiple related methods           |
| Naming         | Verb + Noun: `CreateOrderAction` | Noun: `PaymentGateway`             |
| State          | Stateless per call               | Stateless, shared configuration    |
| Growth         | Stays focused by design          | Grows with the integration it wraps |

### When to Use an Action

Use an Action when you are performing a business operation - something that changes state in your application or triggers a workflow:

- `CreateOrderAction` - creates an order record and dispatches an event
- `SuspendUserAction` - suspends a user account and revokes their tokens
- `ApplyDiscountAction` - calculates and applies a discount to a cart
- `GenerateInvoicePdfAction` - generates a PDF and stores it on disk
- `ImportProductsFromCsvAction` - parses a CSV file and creates product records

Each of these represents a single "thing your application does." If someone asks "what can your app do?", the answer is a list of Actions.

### When to Use a Service

Use a Service when you need to wrap an external system, API, or a utility that does not represent a business operation on its own:

```php
class PaymentGateway
{
    public function __construct(
        private readonly string $apiKey,
        private readonly string $merchantId,
    ) {}

    public function charge(int $amountInCents, string $token): PaymentResult
    {
        // Call Stripe/Braintree/etc.
    }

    public function refund(string $transactionId, int $amountInCents): RefundResult
    {
        // Process a refund
    }

    public function getTransaction(string $transactionId): Transaction
    {
        // Fetch transaction details
    }
}
```

A `PaymentGateway` is not a business operation - it is a tool that Actions use. The `charge()`, `refund()`, and `getTransaction()` methods belong together because they share the same API connection, credentials, and error handling. Splitting them into three separate Actions would be artificial - they are not independent operations, they are facets of one integration.

Other examples of good Services:

- `Geocoder` - converts addresses to coordinates and vice versa
- `CurrencyConverter` - converts between currencies using exchange rates
- `SmsClient` - sends SMS messages through a provider
- `PdfRenderer` - renders HTML to PDF documents

### How They Work Together

Actions and Services are not competing patterns - they complement each other. Actions orchestrate business logic and call Services when they need external capabilities:

```php
class ChargeOrderAction
{
    public function __construct(
        private readonly PaymentGateway $gateway,
    ) {}

    public function execute(Order $order, string $paymentToken): void
    {
        $result = $this->gateway->charge(
            $order->total_in_cents,
            $paymentToken,
        );

        $order->update([
            'payment_status' => 'paid',
            'transaction_id' => $result->transactionId,
        ]);

        event(new OrderPaid($order));
    }
}
```

The `ChargeOrderAction` is the business operation - it charges the customer, updates the order, and dispatches an event. The `PaymentGateway` is the tool it uses to talk to the payment provider. The Action knows *what* to do; the Service knows *how* to talk to the external system.

### The Litmus Test

If you are unsure whether something should be an Action or a Service, ask yourself:

"Does this represent something my application *does*, or something my application *uses*?"

- If it *does* something (creates, updates, processes, sends, imports) - make it an Action.
- If it *uses* something (an API, a library, an external system) - make it a Service.

A `RefundOrderAction` *does* something - it refunds an order, updates records, and notifies the customer. A `PaymentGateway` is something the Action *uses* to process the refund through Stripe.

## Testing Actions

Because Actions are simple classes with typed inputs and outputs, they are a joy to test:

```php
it('creates an order from valid data', function (): void {
    $data = new OrderData(
        email: 'customer@example.com',
        amount_in_cents: 5000,
    );

    $order = app(CreateOrderAction::class)->execute($data);

    expect($order)
        ->toBeInstanceOf(Order::class)
        ->email->toBe('customer@example.com')
        ->amount_in_cents->toBe(5000);
});

it('dispatches an event when an order is created', function (): void {
    Event::fake();

    $data = new OrderData(
        email: 'customer@example.com',
        amount_in_cents: 5000,
    );

    app(CreateOrderAction::class)->execute($data);

    Event::assertDispatched(OrderCreated::class);
});
```

Notice how clean these tests are. There is no HTTP request to set up, no response to assert against. You create a [DTO](/books/clean-code-in-laravel/data-transfer-objects), call the Action, and verify the result. This is the testability dividend of the Action pattern. For a deeper dive into testing strategies, see [The Art of Testing](/books/clean-code-in-laravel/the-art-of-testing).

## Where to Put Actions

If you are following a standard Laravel structure, place Actions in `app/Actions` grouped by domain:

```
app/Actions/
├── Order/
│   ├── CreateOrderAction.php
│   ├── CancelOrderAction.php
│   └── PlaceOrderAction.php
├── Invoice/
│   └── GenerateInvoicePdfAction.php
└── User/
    ├── CreateUserAction.php
    └── SuspendUserAction.php
```

For a domain-oriented structure where Actions live inside each domain folder, see [Thinking in Domains in Laravel](/books/thinking-in-domains-in-laravel).

## The Action Checklist

When creating a new Action, follow these guidelines:

Name it with a verb-noun pattern. `CreateOrderAction`, `PublishPostAction`, `CheckoutCartAction`. The name should describe exactly what the Action does.

Accept typed input. Use [DTOs](/books/clean-code-in-laravel/data-transfer-objects) or typed parameters. Never accept a [Request](https://laravel.com/docs/requests) object.

Return a domain object or void. Never return an HTTP [response](https://laravel.com/docs/responses). The caller decides how to present the result.

Keep it focused on a single business operation. An Action can orchestrate multiple steps that serve one purpose. If the steps are unrelated — updating user settings and sending a marketing email — split them into separate Actions or use events.

Use [events](https://laravel.com/docs/events) for side effects. If creating an order should also send an email, dispatch an `OrderCreated` event and handle the email in a listener. This keeps the Action focused and makes side effects configurable.

Wrap multi-step operations in a [transaction](https://laravel.com/docs/database#database-transactions). If your Action creates multiple database records that must succeed or fail together, use `DB::transaction()` to ensure data consistency.

## Summary

- An Action is a single-purpose class with one public method - typically `execute()` - that performs one discrete business operation.
- Actions are the answer to fat controllers. They take the business logic that does not belong in a controller and give it a proper home.
- Actions accept typed input and return domain objects. They never touch the HTTP [request](https://laravel.com/docs/requests) or return an HTTP [response](https://laravel.com/docs/responses). This makes them context-agnostic - usable from [controllers](https://laravel.com/docs/controllers), [commands](https://laravel.com/docs/artisan), [jobs](/books/clean-code-in-laravel/jobs), and tests.
- Actions compose well. Complex workflows are built by injecting smaller Actions and Services into larger Actions. Each piece stays independently testable.
- Actions can be queueable. The [`spatie/laravel-queueable-action`](https://github.com/spatie/laravel-queueable-action) package lets you run the same Action synchronously or asynchronously - no code duplication needed.
- Actions and [Services](/books/clean-code-in-laravel/organizing-your-application) serve different purposes. An Action performs a business operation (something your application *does*). A Service wraps a utility or external integration (something your application *uses*). Actions call Services, not the other way around.
- Name Actions with a verb-noun pattern. `CreateOrderAction`, `PublishPostAction`, `CheckoutCartAction` — the name tells you exactly what it does without opening the file.
- Actions are easy to test. No HTTP setup, no response assertions. Create a DTO, call the Action, verify the result.
- Use [events](https://laravel.com/docs/events) for side effects, [transactions](https://laravel.com/docs/database#database-transactions) for data integrity. Keep the Action itself focused on the core operation.

## References

- [Refactoring to Actions](https://freek.dev/1371-refactoring-to-actions) — Freek Van der Herten
- [Queueable Actions in Laravel](https://stitcher.io/blog/laravel-queueable-actions) — Brent Roose
- [spatie/laravel-queueable-action](https://github.com/spatie/laravel-queueable-action) — Spatie, GitHub
- [Why I Wrote Laravel Actions](https://lorisleiva.com/why-i-wrote-laravel-actions) — Loris Leiva
- [Laravel Actions Documentation](https://www.laravelactions.com/) — Loris Leiva
- [Organize Laravel Applications With Actions](https://laravel-news.com/organize-laravel-applications-with-actions) — Laravel News
- [Action Pattern in Laravel: Concept, Benefits, Best Practices](https://nabilhassen.com/action-pattern-in-laravel-concept-benefits-best-practices) — Nabil Hassen
- [Service Container](https://laravel.com/docs/container) — Laravel Documentation

