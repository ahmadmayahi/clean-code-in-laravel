A user clicks "Place Order." Behind the scenes, your application needs to charge their credit card, send a confirmation email, generate an invoice PDF, update inventory counts, and notify the warehouse. If all of this happens synchronously, the user stares at a loading spinner for fifteen seconds.

Queues solve this by moving slow work to the background. The user gets an immediate response, and the heavy lifting happens asynchronously. Laravel's queue system is one of its most powerful features, and using it cleanly is essential for any production application.

This chapter focuses on writing clean, reliable Job classes — how to structure them, handle failures, design for idempotency, and use middleware. It does not cover queue infrastructure: setting up workers, configuring [Horizon](https://laravel.com/docs/horizon), organizing queue connections, or deployment strategies. We cover that in [Queue Workers](/books/clean-code-in-laravel/queue-workers).

## Creating and Dispatching Jobs

Generate a Job class with Artisan:

```bash
php artisan make:job ProcessOrderPaymentJob
```

A Job is a class that represents a unit of work to be performed in the background:

```php
namespace App\Jobs;

use App\Models\Order;
use App\Services\Payment\StripePaymentService;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessOrderPaymentJob implements ShouldQueue
{
    use Queueable;

    public function __construct(
        private readonly Order $order,
    ) {}

    public function handle(StripePaymentService $payment): void
    {
        $payment->charge(
            paymentMethodId: $this->order->user->default_payment_method,
            amount: $this->order->total,
            currency: $this->order->currency,
            metadata: ['order_id' => $this->order->id],
        );

        $this->order->update(['status' => OrderStatus::Paid]);
    }
}
```

The `handle()` method receives its dependencies through injection — Laravel's container resolves `StripePaymentService` automatically. The constructor receives the data the Job needs, which is serialized when the Job is dispatched and deserialized when it runs.

Dispatch Jobs from [Actions](/books/clean-code-in-laravel/actions), [Controllers](/books/clean-code-in-laravel/controllers), or Event Listeners:

```php
// From an Action
class PlaceOrderAction
{
    public function execute(PlaceOrderData $data): Order
    {
        $order = DB::transaction(function () use ($data): Order {
            return Order::create([/* ... */]);
        });

        // Dispatch background jobs
        ProcessOrderPaymentJob::dispatch($order);
        SendOrderConfirmationJob::dispatch($order);
        GenerateInvoicePdfJob::dispatch($order);

        return $order;
    }
}
```

## Job Chaining

In the example above, the three Jobs are dispatched independently — they can run in any order, and if one fails, the others still execute. That works for independent tasks, but some operations have strict dependencies between steps.

Consider the order flow: you cannot generate an invoice PDF until the payment succeeds, and you should not send a confirmation email until the invoice is attached. If the payment fails, generating the invoice and sending the email are both pointless. Dispatching these Jobs independently means the invoice Job might run before the payment finishes, or the email Job might send a confirmation for a failed payment.

Chains solve this by running Jobs in sequence. Each Job starts only after the previous one completes successfully. If any Job in the chain fails, the rest are cancelled:

```php
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new ProcessOrderPaymentJob($order),
    new GenerateInvoicePdfJob($order),
    new SendOrderConfirmationJob($order),
    new NotifyWarehouseJob($order),
])->dispatch();
```

Payment runs first. Only after it succeeds does the invoice Job start. Only after the invoice is generated does the confirmation email go out. If the payment fails, none of the subsequent Jobs run — no orphaned invoices, no premature emails.

## Job Batching

Chains enforce order, but sometimes order does not matter — you just need all the work to finish before moving on. A user uploads a photo, and you need to generate a thumbnail, a medium version, a large version, and extract EXIF data. These four operations are independent of each other, so running them one after another in a chain wastes time. You want them to run in parallel, but you need to know when they are all done so you can mark the photo as processed.

Dispatching them independently does not give you that "all done" signal. Each Job runs in isolation with no awareness of the others. You would have to build your own tracking mechanism — counting completions, handling partial failures, deciding what "done" means when some Jobs fail. Batches handle all of this for you:

```php
use Illuminate\Support\Facades\Bus;

$batch = Bus::batch([
    new ResizeImageJob($photo, 'thumbnail'),
    new ResizeImageJob($photo, 'medium'),
    new ResizeImageJob($photo, 'large'),
    new ExtractExifDataJob($photo),
])
->then(function (Batch $batch) use ($photo): void {
    // All jobs completed successfully
    $photo->update(['processed' => true]);
})
->catch(function (Batch $batch, Throwable $e): void {
    // First job failure
    Log::error('Photo processing failed', ['photo' => $photo->id]);
})
->finally(function (Batch $batch): void {
    // Batch finished (success or failure)
})
->dispatch();
```

The `then` callback fires only when every Job in the batch succeeds. The `catch` callback fires on the first failure. The `finally` callback fires regardless of outcome — useful for cleanup.

## Handling Failures

Jobs fail. The payment gateway times out. The email service returns a 503. The database locks during a peak traffic window. Unlike a controller action where the user sees an error page immediately, a failed Job happens silently in the background — nobody knows unless you plan for it.

Without failure handling, a Job that throws an exception simply disappears into the failed jobs table. The order stays in "pending" forever. The user never gets notified. The operations team has no idea something went wrong until a customer complains.

Laravel gives you control over what happens when things go wrong — how many times to retry, how long to wait between attempts, and what to do when all retries are exhausted:

```php
class ProcessOrderPaymentJob implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;                    // Retry up to 3 times
    public int $backoff = 60;                 // Wait 60 seconds between retries
    public int $timeout = 120;                // Kill the job after 2 minutes
    public int $maxExceptions = 2;            // Stop retrying after 2 exceptions

    public function __construct(
        private readonly Order $order,
    ) {}

    public function handle(StripePaymentService $payment): void
    {
        $payment->charge(/* ... */);
    }

    public function failed(Throwable $exception): void
    {
        // Called when all retries are exhausted
        $this->order->update(['status' => OrderStatus::PaymentFailed]);

        $this->order->user->notify(
            new PaymentFailedNotification($this->order, $exception->getMessage()),
        );
    }

    public function retryUntil(): DateTime
    {
        // Alternative to $tries: retry for up to 5 minutes
        return now()->addMinutes(5);
    }
}
```

## Releasing Jobs Back to the Queue

Sometimes a Job cannot do its work right now, but it is not failing either. The external API is temporarily busy. A precondition has not been met yet. A shared resource is locked by another process. In these situations, throwing an exception is wrong — the Job is not broken, it just needs to try again later.

The `$this->release()` method puts the Job back on the queue with an optional delay. Unlike a failure and retry, you are making a deliberate decision: "not now, but soon." The Job's attempt count increments, and it re-enters the queue as if it were freshly dispatched:

```php
class ProcessRefundJob implements ShouldQueue
{
    use Queueable;

    public int $tries = 5;

    public function __construct(
        private readonly Order $order,
    ) {}

    public function handle(PaymentGateway $gateway): void
    {
        // The payment must settle before we can refund
        if ($this->order->payment_status !== PaymentStatus::Settled) {
            $this->release(seconds: 30); // Try again in 30 seconds

            return;
        }

        $gateway->refund($this->order->transaction_id, $this->order->total_in_cents);
        $this->order->update(['status' => OrderStatus::Refunded]);
    }
}
```

The refund cannot happen until the original payment settles — a process that might take a few seconds or a few minutes. Throwing an exception would be misleading: nothing went wrong, the precondition just has not been met yet. Releasing the Job with a 30-second delay gives the payment time to settle without wasting retries on real failures.

### When to Release

Release a Job when the work is valid but the timing is wrong:

- **Waiting for a precondition.** A downstream record has not been created yet, a payment has not settled, or an approval has not come through.
- **Temporary resource unavailability.** A rate limit was hit, a lock is held, or a service returned a "try later" response.
- **Graceful backoff.** An external API returned a `Retry-After` header telling you exactly when to come back.

```php
public function handle(): void
{
    $response = Http::post('https://api.example.com/process', $this->payload);

    if ($response->status() === 429) {
        $retryAfter = (int) $response->header('Retry-After', 60);
        $this->release($retryAfter);

        return;
    }

    $response->throw();

    // Process the successful response...
}
```

This is cleaner than letting the exception propagate — you respect the API's instructions and avoid burning attempts on a predictable outcome.

### Release vs. Failure

The difference between `$this->release()` and letting a Job fail matters. Both increment the attempt counter, but they signal different things:

| | `$this->release()` | Exception (failure) |
|---|---|---|
| **Intent** | "Try again later — nothing is wrong" | "Something broke" |
| **`failed()` called?** | No | Yes, after all retries are exhausted |
| **Logs** | No error logged | Exception logged |
| **Best for** | Preconditions, rate limits, locks | Actual errors — timeouts, bad responses, bugs |

Be mindful that each release still counts as an attempt against `$tries`. If you expect many releases before the work succeeds, use `retryUntil()` instead of `$tries` to avoid running out of attempts:

```php
public function retryUntil(): DateTime
{
    return now()->addMinutes(30); // Keep trying for 30 minutes
}
```

### Rate Limiting with Redis Throttle

When your Job hits an external API with strict rate limits, you need more than a simple `$this->release()` — you need a coordination mechanism that works across all your queue workers. If ten workers all check the rate limit independently, they will all send requests simultaneously and overwhelm the API.

[`Redis::throttle()`](https://laravel.com/docs/queues#job-middleware) provides atomic, distributed rate limiting using Redis locks. It ensures that across all workers, only a certain number of Jobs process within a given time window. Jobs that cannot acquire the lock are released back to the queue automatically:

```php
use Illuminate\Support\Facades\Redis;

class SyncProductToMarketplaceJob implements ShouldQueue
{
    use Queueable;

    public function __construct(
        private readonly Product $product,
    ) {}

    public function retryUntil(): DateTime
    {
        return now()->addMinutes(10);
    }

    public function handle(): void
    {
        Redis::throttle('marketplace-api')
            ->allow(10)   // Allow 10 jobs...
            ->every(60)   // ...per 60 seconds
            ->block(0)    // Do not block waiting for a slot
            ->then(function (): void {
                // Lock acquired — safe to call the API
                Http::throw()->post('https://marketplace.example.com/products', [
                    'sku' => $this->product->sku,
                    'name' => $this->product->name,
                    'price' => $this->product->price,
                ]);
            }, function (): void {
                // Could not acquire lock — release back to queue
                $this->release(10);
            });
    }
}
```

The `throttle()` call has four parts:

- **`allow(10)`** — the number of Jobs permitted in the time window. This should match the API's rate limit.
- **`every(60)`** — the time window in seconds. Together with `allow()`, this means "10 requests per minute."
- **`block(0)`** — how long to wait for a slot to open, in seconds. `0` means do not wait — if no slot is available, fall through to the failure callback immediately. A value like `10` would block the worker for up to 10 seconds waiting for a slot.
- **`then(success, failure)`** — the first closure runs when the lock is acquired. The second runs when it is not. In the failure callback, `$this->release(10)` puts the Job back on the queue with a 10-second delay.

The key (`'marketplace-api'`) is shared across all workers and all Job classes. If you have three different Job types that all hit the same marketplace API, give them the same throttle key — they will share the rate limit.

### Concurrency Limiting with Redis Funnel

Rate limiting controls how many Jobs run within a time window. Sometimes the constraint is different: you want at most N Jobs running at the same time, regardless of how fast they finish. A report generation service might handle only 3 concurrent requests — not 3 per minute, but 3 at once.

`Redis::funnel()` solves this. It limits concurrent executions rather than executions per time window:

```php
use Illuminate\Support\Facades\Redis;

class GenerateExternalReportJob implements ShouldQueue
{
    use Queueable;

    public function retryUntil(): DateTime
    {
        return now()->addMinutes(15);
    }

    public function handle(): void
    {
        Redis::funnel('report-service')
            ->limit(3)    // Maximum 3 concurrent jobs
            ->block(0)    // Do not block waiting
            ->then(function (): void {
                // One of at most 3 concurrent executions
                Http::throw()->post('https://reports.example.com/generate', [
                    'report_id' => $this->reportId,
                ]);
            }, function (): void {
                // All 3 slots are busy — try again later
                $this->release(15);
            });
    }
}
```

The difference from `throttle()` is subtle but important. `throttle('key')->allow(3)->every(60)` means "3 Jobs per minute" — once 3 have run, no more can start until the minute resets, even if they all finished in 2 seconds. `funnel('key')->limit(3)` means "3 at a time" — the moment one finishes, another can start immediately.

| | `Redis::throttle()` | `Redis::funnel()` |
|---|---|---|
| **Controls** | Jobs per time window | Concurrent Jobs |
| **Use when** | API has a rate limit (e.g., 100 requests/minute) | Service has a concurrency limit (e.g., 3 simultaneous connections) |
| **Slot freed** | After the time window resets | As soon as a Job finishes |

### Throttle vs. RateLimited Middleware

The `RateLimited` middleware (covered in [Job Middleware](#rate-limiting)) and `Redis::throttle()` both control how fast Jobs run, but they work at different levels:

The `RateLimited` middleware is declarative — you define a rate limiter in your `AppServiceProvider` and attach it to the Job's `middleware()` method. It is clean and requires no code inside `handle()`. But it offers less control: you cannot customize what happens when the limit is hit, and you cannot share a throttle key across different Job types without extra configuration.

`Redis::throttle()` is imperative — you call it directly inside `handle()`. It is more verbose, but it gives you fine-grained control: custom failure callbacks, blocking behavior, and shared keys that work across any Job class. Use it when `RateLimited` is not flexible enough, or when you need the failure callback to do something other than a simple release.

## Designing Idempotent Jobs

A Job might run more than once, and this is not a theoretical edge case — it happens in production. The queue worker processes a payment Job, charges the customer's card successfully, then crashes before it can acknowledge the Job. The queue thinks the Job never ran and hands it to another worker. Now the customer gets charged twice.

This is why idempotency matters. An idempotent Job produces the same result whether it runs once, twice, or ten times. The fix is simple: check whether the work has already been done before doing it:

```php
class ProcessOrderPaymentJob implements ShouldQueue
{
    use Queueable;

    public function __construct(
        private readonly Order $order,
    ) {}

    public function handle(StripePaymentService $payment): void
    {
        // Guard: skip if already paid
        if ($this->order->status === OrderStatus::Paid) {
            return;
        }

        $payment->charge(/* ... */);

        $this->order->update(['status' => OrderStatus::Paid]);
    }
}
```

The guard clause at the top checks whether the work has already been done. If a retry fires after the first run succeeded, the Job exits immediately. This pattern applies to any Job with side effects — always check before acting.

For database writes, prefer `updateOrCreate()` or `upsert()` over `create()` when the Job might run twice. For sending notifications, store a "notification sent" flag and check it before sending.

## Dispatching After Transactions

Dispatching a Job inside a database transaction is a common source of bugs that only surfaces under load. In development, the transaction commits so fast that the Job always finds its data. In production, under heavy traffic, the queue worker picks up the Job before the transaction commits. The Job tries to load the order — and it does not exist yet. Or worse, the transaction rolls back, and the Job processes an order that was never persisted:

```php
// Dangerous: the Job might run before the transaction commits
DB::transaction(function () use ($data): void {
    $order = Order::create($data);
    ProcessOrderPaymentJob::dispatch($order); // ← Job might start now
});
```

Use `afterCommit()` to ensure the Job is only dispatched after the transaction succeeds:

```php
DB::transaction(function () use ($data): void {
    $order = Order::create($data);
    ProcessOrderPaymentJob::dispatch($order)->afterCommit();
});
```

If the transaction rolls back, the Job is silently discarded. You can also set `after_commit` to `true` in your queue connection config to make this the default behavior for all Jobs.

## Job Middleware

As your Jobs grow, you will notice the same boilerplate creeping into multiple `handle()` methods — checking if the user still has an active subscription, acquiring a lock to prevent overlapping runs, or catching specific exceptions to back off from a failing API. This is the same problem that route middleware solves for HTTP requests: cross-cutting concerns that do not belong in the main logic.

[Job middleware](https://laravel.com/docs/queues#job-middleware) wraps the execution of a Job with reusable logic. You extract the concern into a middleware class, and the Job's `handle()` method stays focused entirely on its actual work.

Attach middleware by adding a `middleware()` method to your Job class. This method is not scaffolded by `make:job`, so you add it manually:

```php
public function middleware(): array
{
    return [new SomeMiddleware];
}
```

### Preventing Overlapping Execution

Some Jobs modify a shared resource that can only be safely changed by one process at a time. If two workers pick up a "sync user to CRM" Job for the same user simultaneously, they might overwrite each other's changes or send duplicate data to the external service. `WithoutOverlapping` acquires a lock before the Job runs, so only one instance executes at a time for a given key:

```php
use Illuminate\Queue\Middleware\WithoutOverlapping;

class SyncUserToExternalServiceJob implements ShouldQueue
{
    public function __construct(
        private readonly User $user,
    ) {}

    public function middleware(): array
    {
        return [
            (new WithoutOverlapping($this->user->id))
                ->expireAfter(180), // Release lock after 3 minutes
        ];
    }

    public function handle(): void
    {
        // Only one sync per user runs at a time
    }
}
```

The key you pass to `WithoutOverlapping` determines the scope — a user ID means "one sync per user," an order ID means "one per order." By default, overlapping Jobs are released back to the queue and retried. Use `dontRelease()` to discard them instead.

If two different Job classes share the same resource, use the `shared()` method so they respect the same lock:

```php
// Both jobs share a lock keyed by provider status
public function middleware(): array
{
    return [
        (new WithoutOverlapping("provider:{$this->provider->id}"))->shared(),
    ];
}
```

### Rate Limiting

External APIs have rate limits. If your queue has 500 pending Jobs that all hit the same API, your workers will fire requests as fast as they can, the API will start returning 429 errors, and every Job will fail. Even if you have retries configured, you are just delaying the flood — the retried Jobs will hit the same wall.

Rate limiting at the Job level ensures your workers respect the API's limits. First, define the limiter in your `AppServiceProvider`:

```php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

public function boot(): void
{
    RateLimiter::for('external-api', function (object $job): Limit {
        return Limit::perMinute(30);
    });
}
```

Then attach it to your Job:

```php
use Illuminate\Queue\Middleware\RateLimited;

class CallExternalApiJob implements ShouldQueue
{
    public function middleware(): array
    {
        return [
            new RateLimited('external-api'),
        ];
    }

    public function handle(): void
    {
        Http::post('https://api.example.com/webhook', $this->payload);
    }
}
```

When a Job exceeds the rate limit, it is released back to the queue with an appropriate delay. This consumes an attempt, so set your `$tries` high enough or use `retryUntil()` instead. If you are using Redis, prefer `RateLimitedWithRedis` — it is more efficient and fine-tuned for Redis's atomic operations.

### Throttling Exceptions

Rate limiting controls how fast you send requests. Throttling exceptions handles what happens when the service is already down. Without it, your workers keep hammering a failing API — every Job throws an exception, gets retried immediately, fails again, and consumes another attempt. You burn through all your retries in seconds against a service that is not going to recover in seconds.

`ThrottlesExceptions` detects the pattern. Once a Job throws a certain number of consecutive exceptions, it pauses all further attempts until a cooldown period passes — giving the external service time to recover:

```php
use Illuminate\Queue\Middleware\ThrottlesExceptions;

class FetchAnalyticsFromExternalApi implements ShouldQueue
{
    public function middleware(): array
    {
        return [
            (new ThrottlesExceptions(maxAttempts: 5, decaySeconds: 10 * 60))
                ->by('analytics-provider'), // Share the throttle across jobs
        ];
    }

    public function retryUntil(): DateTime
    {
        return now()->addHours(1);
    }

    public function handle(): void
    {
        $response = Http::throw()
            ->get('https://analytics.example.com/data');

        // Process analytics data...
    }
}
```

If the Job throws 5 exceptions, it backs off for 10 minutes before trying again — preventing a cascade of failures against an already-struggling service. The `by()` method lets multiple Job classes share the same throttle bucket, which is useful when several Jobs hit the same external API.

You can also filter which exceptions trigger the throttle using `when()`:

```php
(new ThrottlesExceptions(5, 10 * 60))
    ->when(fn (Throwable $e): bool => $e instanceof HttpClientException)
```

### Skipping Jobs

A Job is dispatched at one moment and processed at another. Between dispatch and execution, the world can change — a user unsubscribes, an account gets deactivated, a feature flag gets turned off. You could add guard clauses at the top of `handle()`, but when the condition is purely about whether the Job should run at all, the `Skip` middleware expresses this more clearly:

```php
use Illuminate\Queue\Middleware\Skip;

class SendWeeklyDigestJob implements ShouldQueue
{
    public function __construct(
        private readonly User $user,
    ) {}

    public function middleware(): array
    {
        return [
            Skip::when($this->user->hasUnsubscribed()),
        ];
    }

    public function handle(): void
    {
        // Send the digest — only runs if the user is still subscribed
    }
}
```

This is cleaner than adding guard clauses at the top of `handle()` for conditions that are purely about whether the Job should run at all. Use `Skip::unless()` for the inverse — skip when the condition is `false`.

### Failing on Specific Exceptions

Some exceptions are transient (API timeouts, rate limits) and worth retrying. Others are permanent (invalid credentials, authorization failures) and retrying them is pointless. `FailOnException` lets you short-circuit retries for exceptions that will never succeed:

```php
use Illuminate\Queue\Middleware\FailOnException;
use Illuminate\Auth\Access\AuthorizationException;

class SyncChatHistoryJob implements ShouldQueue
{
    public int $tries = 5;

    public function middleware(): array
    {
        return [
            new FailOnException([AuthorizationException::class]),
        ];
    }

    public function handle(): void
    {
        $this->user->authorize('sync-chat-history');

        Http::throw()->get("https://chat.example.com/?user={$this->user->uuid}");
    }
}
```

If the user's permissions are revoked, the Job fails immediately instead of wasting 4 more retries on an operation that can never succeed. Transient HTTP errors still get their retries.

### Writing Custom Middleware

When the built-in middleware does not fit your needs, write your own. A Job middleware is a class with a `handle` method that receives the Job and a `$next` closure — the same pattern as route middleware:

```php
namespace App\Jobs\Middleware;

use Closure;

class EnsureActiveSubscription
{
    public function handle(object $job, Closure $next): void
    {
        if (! $job->user->hasActiveSubscription()) {
            $job->delete();

            return;
        }

        $next($job);
    }
}
```

Generate the class with `php artisan make:job-middleware`. Then attach it to any Job:

```php
use App\Jobs\Middleware\EnsureActiveSubscription;

public function middleware(): array
{
    return [new EnsureActiveSubscription];
}
```

Custom middleware is the right place for domain-specific checks that apply across multiple Jobs. If you find the same guard clause at the top of several `handle()` methods, extract it into a middleware.

### Combining Middleware

You can stack multiple middlewares on a single Job — they execute in order, wrapping each other like layers:

```php
public function middleware(): array
{
    return [
        new FailOnException([AuthorizationException::class]),
        new RateLimited('external-api'),
        (new WithoutOverlapping($this->user->id))->expireAfter(180),
        new EnsureActiveSubscription,
    ];
}
```

This Job will: fail immediately on authorization errors, respect the API rate limit, prevent overlapping execution per user, and skip if the user's subscription expired. Each concern is isolated in its own middleware, keeping `handle()` focused entirely on the business logic.

## Unique Jobs

Imagine a user clicks "Generate Report" and nothing happens immediately — the button is still active, so they click it three more times. Now your queue has four identical Jobs generating the same report, wasting resources and potentially producing duplicate results. Or consider a webhook endpoint that receives the same event twice within a second — both requests dispatch the same Job before the first one finishes.

These are the problems unique Jobs solve. By implementing `ShouldBeUnique`, Laravel uses an atomic cache lock to ensure that only one instance of a Job with the same key exists on the queue at any time. If a duplicate is dispatched while the original is still queued or processing, the duplicate is silently discarded:

```php
use Illuminate\Contracts\Queue\ShouldBeUnique;

class GenerateMonthlyReportJob implements ShouldQueue, ShouldBeUnique
{
    public int $uniqueFor = 3600; // Unique for 1 hour

    public function __construct(
        private readonly int $userId,
        private readonly string $month,
    ) {}

    public function uniqueId(): string
    {
        return "generating-report-user-{$this->userId}-month-{$this->month}";
    }

    public function handle(): void
    {
        // Generate the report — guaranteed no duplicates
    }
}
```

The `uniqueId()` method defines *what* makes the Job unique. In this example, a report for user 5 in March is a different Job than a report for user 5 in April. The `$uniqueFor` property sets how long the lock lasts — after one hour, the same Job can be dispatched again even if the original never completed.

By default, the unique lock is held until the Job finishes processing or fails all its retry attempts. If you want the lock released as soon as a worker *picks up* the Job — allowing another instance to be queued while the first is still running — implement `ShouldBeUniqueUntilProcessing` instead:

```php
use Illuminate\Contracts\Queue\ShouldBeUniqueUntilProcessing;

class SyncInventoryJob implements ShouldQueue, ShouldBeUniqueUntilProcessing
{
    // The lock releases when a worker starts processing,
    // allowing the next sync to be queued immediately
}
```

### Unique Jobs vs. WithoutOverlapping Middleware

`ShouldBeUnique` and `WithoutOverlapping` both prevent duplicate work, but they operate at different stages and solve different problems:

| | `ShouldBeUnique` | `WithoutOverlapping` |
|---|---|---|
| **When it acts** | At dispatch time — prevents the Job from entering the queue | At execution time — prevents the Job from running |
| **Duplicates on queue** | No — only one instance exists on the queue | Yes — multiple instances can sit on the queue |
| **What happens to duplicates** | Silently discarded, never queued | Released back to the queue (or discarded with `dontRelease()`) |
| **Best for** | Preventing wasted queue space (reports, syncs, exports) | Protecting shared resources (one sync per user at a time) |

Use `ShouldBeUnique` when the Job itself is redundant — generating the same report twice is pointless. Use `WithoutOverlapping` when the Job is valid but must not run concurrently — two syncs for the same user running simultaneously could corrupt data, but queuing the second one to run after the first finishes is fine.

You can combine both on the same Job when you need to prevent queuing duplicates *and* protect against concurrent execution:

```php
class SyncUserDataJob implements ShouldQueue, ShouldBeUnique
{
    public function __construct(
        private readonly User $user,
    ) {}

    public function uniqueId(): string
    {
        return (string) $this->user->id;
    }

    public function middleware(): array
    {
        return [
            (new WithoutOverlapping($this->user->id))->expireAfter(180),
        ];
    }

    public function handle(ExternalCrmService $crm): void
    {
        $crm->sync($this->user);
    }
}
```

## Organizing Jobs

Like [Actions](/books/clean-code-in-laravel/actions) and [Services](/books/clean-code-in-laravel/organizing-your-application), organize Jobs by domain:

```
app/Jobs/
├── Order/
│   ├── ProcessOrderPaymentJob.php
│   ├── GenerateInvoicePdfJob.php
│   └── SendOrderConfirmationJob.php
├── User/
│   ├── SendWelcomeEmailJob.php
│   └── SyncUserToMailchimpJob.php
└── Report/
    └── GenerateMonthlyReportJob.php
```

## Jobs vs. Actions vs. Listeners

When you need to "do something" in Laravel, you have three options: an [Action](/books/clean-code-in-laravel/actions), a Job, or an Event Listener. They overlap in purpose, and choosing the wrong one leads to awkward code — an Action that does not return anything, a Job that needs to return a value, or a Listener that triggers a chain of unrelated side effects:

| Pattern | Execution | Returns Value | Use When |
|---|---|---|---|
| [Action](/books/clean-code-in-laravel/actions) | Synchronous | Yes | User-facing operations that need an immediate result |
| Job | Asynchronous | No | Background work that can happen later |
| Listener | Either | No | Reacting to an event that already happened |

A common pattern is: an [Action](/books/clean-code-in-laravel/actions) performs the core operation synchronously, then dispatches Jobs for the background work:

```php
class PlaceOrderAction
{
    public function execute(PlaceOrderData $data): Order
    {
        // Synchronous: create the order (user needs the result)
        $order = $this->createOrder($data);

        // Asynchronous: background work
        ProcessOrderPaymentJob::dispatch($order);
        SendOrderConfirmationJob::dispatch($order)->delay(now()->addSeconds(30));

        return $order;
    }
}
```

The user gets their order confirmation page immediately. The payment processing and email happen in the background.

## Testing Jobs

There are two things to test with Jobs: that the right Jobs are dispatched under the right conditions, and that a Job's `handle()` logic works correctly. These are different concerns that require different approaches.

For dispatch testing, use `Queue::fake()` or `Bus::fake()`. These replace the real queue with an in-memory fake that records what was dispatched without actually running anything:

```php
use Illuminate\Support\Facades\Queue;

it('dispatches payment and confirmation jobs when placing an order', function (): void {
    Queue::fake();

    $action = app(PlaceOrderAction::class);
    $order = $action->execute($orderData);

    Queue::assertPushed(ProcessOrderPaymentJob::class, function ($job) use ($order): bool {
        return $job->order->id === $order->id;
    });

    Queue::assertPushed(SendOrderConfirmationJob::class);
    Queue::assertNotPushed(NotifyWarehouseJob::class);
});
```

The closure passed to `assertPushed()` lets you verify that the Job received the correct data — not just that it was dispatched, but that it was dispatched *with the right arguments*.

Dispatch tests verify that Jobs are queued. They do not test what the Job actually does. For that, instantiate the Job directly and call `handle()` — this runs the Job's logic synchronously without touching the queue:

```php
it('marks the order as paid after successful charge', function (): void {
    $order = Order::factory()->create(['status' => OrderStatus::Pending]);

    $payment = Mockery::mock(StripePaymentService::class);
    $payment->shouldReceive('charge')->once();

    $job = new ProcessOrderPaymentJob($order);
    $job->handle($payment);

    expect($order->fresh()->status)->toBe(OrderStatus::Paid);
});

it('skips payment if order is already paid', function (): void {
    $order = Order::factory()->create(['status' => OrderStatus::Paid]);

    $payment = Mockery::mock(StripePaymentService::class);
    $payment->shouldNotReceive('charge');

    $job = new ProcessOrderPaymentJob($order);
    $job->handle($payment);
});
```

The second test verifies the idempotency guard — the Job should not charge twice.

For chains and batches, use `Bus::fake()` and assert on `Bus::assertChained()` or `Bus::assertBatched()`.

## Self-Contained Jobs

A Job is dispatched at one point in time and processed at another — seconds, minutes, or even hours later. The state of the world can change between those two moments. A deploy Job that fetches the "latest commit" inside `handle()` might deploy a completely different commit than the one the user clicked "deploy" for, because someone pushed new code while the Job was waiting in the queue.

The fix is to capture all necessary state at dispatch time, not at execution time:

```php
// Before: fetches the commit hash when the job runs — it may have changed
class DeploySiteJob implements ShouldQueue
{
    public function __construct(
        private readonly Site $site,
    ) {}

    public function handle(): void
    {
        // By now, a new commit may have been pushed
        $hash = $this->site->repository()->lastCommitHash();

        $this->deploy($hash);
    }
}

// After: captures the commit hash at dispatch time
class DeploySiteJob implements ShouldQueue
{
    public function __construct(
        private readonly Site $site,
        private readonly string $commitHash,
    ) {}

    public function handle(): void
    {
        $this->deploy($this->commitHash);
    }
}

// Dispatch with the current state
DeploySiteJob::dispatch($site, $site->repository()->lastCommitHash());
```

The first version fetches the commit hash inside `handle()`. If the Job is delayed in the queue, the hash might point to a different commit than the one the user intended to deploy. The second version freezes the hash at dispatch time — the Job always deploys exactly what was requested, regardless of when it runs.

This principle applies to configuration values, exchange rates, feature flags, and any other state that might change between dispatch and execution. If the Job needs it, pass it through the constructor.

## Common Pitfalls

- **Leaving `QUEUE_CONNECTION=sync` in production.** The `sync` driver runs Jobs inline with the HTTP request — the user waits for every Job to finish. This defeats the entire purpose of queues. Use `redis`, `database`, or `sqs` in production.
- **Serializing large payloads.** Jobs serialize their constructor data into the queue. Passing an entire collection of models, a large array, or a file's contents bloats the queue payload and slows dispatching. Instead, pass IDs or references and let the Job fetch what it needs:

```php
// Before: serializes the entire collection
SendBulkEmailJob::dispatch($users);

// After: pass IDs, fetch in the Job
SendBulkEmailJob::dispatch($users->pluck('id')->all());
```

- **Injecting services through the constructor.** Only pass data through the constructor — services get serialized into the queue payload, which bloats it and breaks when the class changes. Use `handle()` method injection instead — Laravel's container resolves the dependency when the Job runs:

```php
// Before: the service gets serialized into the queue
class GenerateReportJob implements ShouldQueue
{
    public function __construct(
        private readonly int $reportId,
        private readonly ReportGenerator $generator, // ← serialized!
    ) {}

    public function handle(): void
    {
        $this->generator->generate($this->reportId);
    }
}

// After: the container resolves the service at runtime
class GenerateReportJob implements ShouldQueue
{
    public function __construct(
        private readonly int $reportId,
    ) {}

    public function handle(ReportGenerator $generator): void
    {
        $generator->generate($this->reportId);
    }
}
```

- **Not separating queues.** Mixing critical Jobs (payment processing) with non-critical ones (sending marketing emails) on the same queue means a flood of emails can delay payments. Use named queues and run separate workers:

```php
ProcessOrderPaymentJob::dispatch($order)->onQueue('payments');
SendMarketingEmailJob::dispatch($user)->onQueue('emails');
```

```bash
php artisan queue:work --queue=payments
php artisan queue:work --queue=emails
```

- **Forgetting `--max-jobs` or `--max-time` on workers.** Long-running queue workers can [leak memory](https://en.wikipedia.org/wiki/Memory_leak) over time — objects that are no longer needed stay in memory because something still references them, and PHP's garbage collector cannot reclaim them. Over hours or days, this gradual accumulation can exhaust the server's available memory and crash the worker. Use `--max-jobs=1000` or `--max-time=3600` so the worker restarts periodically, and let your process manager (Supervisor, systemd) restart it automatically.
- **Relying on shared static state.** Queue workers are long-running processes that reuse a single application instance across many Jobs. Static variables set by one Job persist for the next. This is a subtle source of bugs:

```php
class Currency
{
    public static array $rates = [];

    public static function convert(float $amount, string $from, string $to): float
    {
        if (empty(static::$rates)) {
            static::$rates = ExchangeRateApi::fetch();
        }

        return $amount * static::$rates[$to] / static::$rates[$from];
    }
}
```

If a Job calls `Currency::convert()` in January, the rates are fetched and cached in `$rates`. When the next Job runs in February — in the same worker process — the stale January rates are still in memory. The second Job silently uses outdated data without ever hitting the API.

Avoid mutable static state in classes used by Jobs, or clear it explicitly. Better yet, inject a service through `handle()` that fetches fresh data each time.

## The Jobs Checklist

1. **Implement `ShouldQueue`** — without it, the Job runs synchronously
2. **Make Jobs self-contained** — capture all necessary state at dispatch time, not inside `handle()`
3. **Design for idempotency** — guard against double-processing with status checks and upserts
4. **Set `$tries` and `$backoff`** — always plan for failure
5. **Implement `failed()`** — notify someone when a Job permanently fails
6. **Use `afterCommit()`** when dispatching inside transactions — prevent Jobs from referencing uncommitted data
7. **Use the `Queueable` trait** — it combines `Dispatchable`, `InteractsWithQueue`, `Queueable`, and `SerializesModels` into a single import
8. **Pass data, not services** — use `handle()` method injection for services; only pass data through the constructor
9. **Pass IDs, not objects** — do not serialize large collections or file contents into the queue payload
10. **Use job middleware** — `WithoutOverlapping` for exclusive execution, `RateLimited` for API throttling
11. **Use `$this->release()`** for deliberate retries — when a precondition is not met or a rate limit is hit, release with a delay instead of throwing an exception
12. **Use `Redis::throttle()` or `Redis::funnel()`** for distributed rate limiting and concurrency control across workers
13. **Keep Jobs focused** — one Job, one task. Compose with chains and batches
13. **Use unique Jobs** to prevent duplicates — especially for reports and sync operations
14. **Separate queues by priority** — run critical and non-critical Jobs on different queues with dedicated workers
14. **Avoid mutable static state** — workers reuse the same process; static variables persist across Jobs
15. **Test with `Queue::fake()`** — assert that the right Jobs are dispatched with the right data

## Summary

- Queues move slow work to the background — payment processing, email sending, PDF generation — so the user gets an immediate response instead of waiting.
- A Job's `handle()` method receives dependencies through Laravel's container. The constructor receives only data, which is serialized into the queue. Never inject services through the constructor.
- Design every Job for idempotency. Guard against double-processing with status checks at the top of `handle()`, and prefer `updateOrCreate()` or `upsert()` over `create()`.
- Use `afterCommit()` when dispatching Jobs inside database transactions. Without it, the Job might start processing before the transaction commits — or reference data from a rolled-back transaction.
- Use `$this->release()` when a Job cannot do its work right now but is not failing — a precondition has not been met, a rate limit was hit, or a resource is locked. Release puts the Job back on the queue with a delay instead of wasting retries on exceptions. Use `retryUntil()` instead of `$tries` when you expect multiple releases.
- For distributed rate limiting across workers, use `Redis::throttle()` to control how many Jobs run per time window, or `Redis::funnel()` to limit concurrent executions. Both coordinate across all workers via Redis locks, and Jobs that cannot acquire a slot are released back to the queue.
- Job middleware (`WithoutOverlapping`, `RateLimited`, `ThrottlesExceptions`, `Skip`, `FailOnException`) keeps cross-cutting concerns out of `handle()`. Write custom middleware when the built-in options do not fit.
- `ShouldBeUnique` prevents duplicate Jobs from entering the queue — the duplicate is silently discarded at dispatch time. `WithoutOverlapping` prevents concurrent execution — duplicates are queued but wait their turn. Use both together when you need to prevent queuing *and* protect against concurrent execution.
- Make Jobs self-contained by capturing all necessary state at dispatch time. Values that can change between dispatch and execution — commit hashes, exchange rates, feature flags — should be passed through the constructor.
- Separate queues by priority. Critical Jobs (payments) and non-critical Jobs (marketing emails) should run on different queues with dedicated workers.
- Test dispatch logic with `Queue::fake()` and `Bus::fake()`. Test Job logic directly by instantiating the class and calling `handle()` with mocked dependencies.

## References

- [Queues](https://laravel.com/docs/queues) — Laravel Documentation
- [Horizon](https://laravel.com/docs/horizon) — Laravel Documentation
- [Laravel Queues in Action](https://learn-laravel-queues.com/) — Mohamed Said
- [Job Batching in Laravel: How It Works](https://themsaid.com/queue-job-batching-in-laravel-how-it-works) — Mohamed Said
- [How to Group Queued Jobs Using Laravel's Batch Class](https://freek.dev/1734-how-to-group-queued-jobs-using-laravel-8s-new-batch-class) — Freek Van der Herten
- [A Job Middleware to Rate Limit Jobs](https://freek.dev/1475-a-job-middleware-to-rate-limit-jobs) — Freek Van der Herten
- [A Deep Dive Into Laravel Queues](https://www.honeybadger.io/blog/laravel-queues-deep-dive/) — Honeybadger
- [Laravel Queues Under the Hood](https://wendelladriel.com/blog/laravel-queues-under-the-hood) — Wendell Adriel
- [Redis](https://laravel.com/docs/redis) — Laravel Documentation
- [Laravel Jobs and Queues 101](https://laravel-news.com/laravel-jobs-and-queues-101) — Laravel News
