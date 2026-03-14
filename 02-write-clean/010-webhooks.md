A webhook is a simple idea: when something happens in one application, it sends an HTTP request to another application to let it know. No polling, no cron jobs, no checking every five minutes. Just a direct notification the moment something occurs.

Laravel does not ship with a built-in webhook system, but Spatie provides two packages that handle both sides of the equation cleanly: [`spatie/laravel-webhook-client`](https://spatie.be/docs/laravel-webhook-client) for receiving webhooks and [`spatie/laravel-webhook-server`](https://spatie.be/docs/laravel-webhook-server) for sending them. In this chapter, we will build both sides using patterns that keep your code clean, secure, and testable.

## Why Webhooks Matter

Consider a payment provider like Stripe. After a customer completes a payment, Stripe needs to tell your application about it. You could poll Stripe's API every few seconds, but that wastes resources and introduces latency. Instead, Stripe sends a webhook — a POST request to a URL you provide — containing the event details.

Webhooks appear everywhere in modern applications: payment confirmations, repository pushes, form submissions, shipping updates, CRM changes. If your application integrates with any external service, you will eventually need to handle webhooks.

The challenge is doing it cleanly. A naive implementation scatters webhook logic across controllers, skips signature verification, and processes payloads synchronously. Spatie's packages give you a structured, queue-first, security-conscious approach.

## Receiving Webhooks

### Installation

```bash
composer require spatie/laravel-webhook-client
```

Publish the configuration and migration:

```bash
php artisan vendor:publish --provider="Spatie\WebhookClient\WebhookClientServiceProvider" --tag="webhook-client-config"
php artisan vendor:publish --provider="Spatie\WebhookClient\WebhookClientServiceProvider" --tag="webhook-client-migrations"
php artisan migrate
```

### Registering the Route

Webhook endpoints receive POST requests from external services. These services cannot obtain a CSRF token — they are not browsers submitting forms. If you register the route without excluding it from CSRF verification, every incoming webhook will be rejected with a 419 status code.

Spatie provides a convenient route macro. In your `routes/web.php`:

```php
Route::webhooks('webhooks/stripe', 'stripe');
```

This registers a POST route at `/webhooks/stripe` using the `stripe` configuration name. Exclude webhook routes from CSRF verification in `bootstrap/app.php`:

```php
// bootstrap/app.php
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->validateCsrfTokens(except: [
            'webhooks/*',
        ]);
    })
    ->create();
```

### Configuration

The config file at `config/webhook-client.php` defines one or more webhook endpoints. Each endpoint specifies how to verify signatures, which webhooks to accept, and which job processes them:

```php
// config/webhook-client.php
return [
    'configs' => [
        [
            'name' => 'stripe',
            'signing_secret' => env('STRIPE_WEBHOOK_SECRET'),
            'signature_header_name' => 'Stripe-Signature',
            'signature_validator' => App\Webhooks\StripeSignatureValidator::class,
            'webhook_profile' => App\Webhooks\StripeWebhookProfile::class,
            'webhook_response' => Spatie\WebhookClient\WebhookResponse\DefaultRespondsTo::class,
            'webhook_model' => Spatie\WebhookClient\Models\WebhookCall::class,
            'store_headers' => ['Stripe-Signature'],
            'process_webhook_job' => App\Webhooks\Jobs\ProcessStripeWebhookJob::class,
        ],
        [
            'name' => 'github',
            'signing_secret' => env('GITHUB_WEBHOOK_SECRET'),
            'signature_header_name' => 'X-Hub-Signature-256',
            'signature_validator' => Spatie\WebhookClient\SignatureValidator\DefaultSignatureValidator::class,
            'webhook_profile' => Spatie\WebhookClient\WebhookProfile\ProcessEverythingWebhookProfile::class,
            'webhook_response' => Spatie\WebhookClient\WebhookResponse\DefaultRespondsTo::class,
            'webhook_model' => Spatie\WebhookClient\Models\WebhookCall::class,
            'store_headers' => [],
            'process_webhook_job' => App\Webhooks\Jobs\ProcessGitHubWebhookJob::class,
        ],
    ],

    'delete_after_days' => 30,
];
```

Notice how each webhook source gets its own configuration block. This keeps concerns separated — Stripe has different signature validation than GitHub.

### Signature Verification

Never trust an incoming webhook without verifying its signature. The default validator computes an HMAC-SHA256 hash and compares it to the signature header. For services like Stripe that use a different signing scheme, write a custom validator:

```php
<?php

namespace App\Webhooks;

use Illuminate\Http\Request;
use Spatie\WebhookClient\SignatureValidator\SignatureValidator;
use Spatie\WebhookClient\WebhookConfig;

class StripeSignatureValidator implements SignatureValidator
{
    public function isValid(Request $request, WebhookConfig $config): bool
    {
        $signature = $request->header('Stripe-Signature');

        if (! $signature) {
            return false;
        }

        $secret = $config->signingSecret;

        try {
            \Stripe\Webhook::constructEvent(
                $request->getContent(),
                $signature,
                $secret
            );

            return true;
        } catch (\Stripe\Exception\SignatureVerificationException) {
            return false;
        }
    }
}
```

This delegates to Stripe's own SDK for signature verification, which handles timestamp validation and replay attack prevention.

### Webhook Profiles: Filtering What You Process

Stripe sends dozens of event types — `balance.available`, `charge.refunded`, `customer.created`, and many more. Most applications only care about a handful. Processing and storing every event wastes database space and queue capacity. Worse, it means your processing job has to handle events it was never designed for.

A webhook profile determines whether an incoming webhook should be stored and processed. Filter irrelevant events at the gate, before they reach the queue:

```php
<?php

namespace App\Webhooks;

use Illuminate\Http\Request;
use Spatie\WebhookClient\WebhookProfile\WebhookProfile;

class StripeWebhookProfile implements WebhookProfile
{
    public function shouldProcess(Request $request): bool
    {
        /** @var array{type?: string} $payload */
        $payload = json_decode($request->getContent(), true);

        $allowedEvents = [
            'payment_intent.succeeded',
            'payment_intent.payment_failed',
            'customer.subscription.created',
            'customer.subscription.updated',
            'customer.subscription.deleted',
            'invoice.payment_succeeded',
            'invoice.payment_failed',
        ];

        return in_array($payload['type'] ?? '', $allowedEvents);
    }
}
```

This is clean separation. The profile decides *whether* to process. The job decides *how* to process.

### Processing Webhooks in Queued Jobs

The most important design decision: always process webhooks asynchronously. The external service expects a fast response (usually within 5 seconds). If you process synchronously, a slow database query or external API call could cause a timeout, and the sender will retry — potentially creating duplicate processing.

```php
<?php

namespace App\Webhooks\Jobs;

use App\Actions\Payments\HandlePaymentSucceeded;
use App\Actions\Payments\HandlePaymentFailed;
use App\Actions\Subscriptions\HandleSubscriptionChange;
use Spatie\WebhookClient\Jobs\ProcessWebhookJob;

class ProcessStripeWebhookJob extends ProcessWebhookJob
{
    public function handle(): void
    {
        $payload = $this->webhookCall->payload;
        $eventType = $payload['type'] ?? null;

        match ($eventType) {
            'payment_intent.succeeded' => app(HandlePaymentSucceeded::class)
                ->execute($payload['data']['object']),

            'payment_intent.payment_failed' => app(HandlePaymentFailed::class)
                ->execute($payload['data']['object']),

            'customer.subscription.created',
            'customer.subscription.updated',
            'customer.subscription.deleted' => app(HandleSubscriptionChange::class)
                ->execute($eventType, $payload['data']['object']),

            default => null,
        };
    }
}
```

Notice the pattern: the job acts as a router, delegating to [Action](/books/clean-code-in-laravel/actions) classes for the actual business logic. Each Action is a single-purpose class that can be tested independently — exactly the [Action pattern](/books/clean-code-in-laravel/actions) covered earlier.

### Handling Idempotency

Webhook senders retry on failure — if your server responds slowly or returns an error, the sender assumes the webhook was not delivered and sends it again. Without idempotency, you process the same event twice: creating duplicate payment records, sending duplicate emails, or charging a customer twice. Your processing must be idempotent — processing the same webhook twice should produce the same result as processing it once:

```php
<?php

namespace App\Actions\Payments;

use App\Models\Payment;

class HandlePaymentSucceeded
{
    /** @param array{id: string, amount: int, currency: string} $paymentIntent */
    public function execute(array $paymentIntent): void
    {
        // Idempotency: check if we already processed this event
        $existingPayment = Payment::where(
            'stripe_payment_intent_id',
            $paymentIntent['id']
        )->first();

        if ($existingPayment?->status === 'completed') {
            return; // Already processed, skip
        }

        Payment::updateOrCreate(
            ['stripe_payment_intent_id' => $paymentIntent['id']],
            [
                'amount' => $paymentIntent['amount'],
                'currency' => $paymentIntent['currency'],
                'status' => 'completed',
                'paid_at' => now(),
            ]
        );
    }
}
```

## Sending Webhooks

Everything above covers receiving webhooks from external services. But what about the other direction? When your application is the source of events — an order ships, a user upgrades, a report completes — other applications need to know. You could expose a polling API, but that puts the burden on every consumer to check repeatedly. Webhooks invert this: your application pushes notifications the moment something happens.

[`spatie/laravel-webhook-server`](https://spatie.be/docs/laravel-webhook-server) handles the sending side — payload signing, queued delivery, retries with backoff, and event logging.

### Installation

```bash
composer require spatie/laravel-webhook-server
```

### Sending a Simple Webhook

The `WebhookCall` class provides a fluent API for building and dispatching webhook requests. Each call is signed with a shared secret so the receiver can verify authenticity — the same verification you implement on the receiving side:

```php
use Spatie\WebhookServer\WebhookCall;

WebhookCall::create()
    ->url('https://partner-app.com/webhooks')
    ->payload([
        'event' => 'order.shipped',
        'data' => [
            'order_id' => $order->id,
            'tracking_number' => $order->tracking_number,
            'shipped_at' => $order->shipped_at->toIso8601String(),
        ],
    ])
    ->useSecret(config('services.partner.webhook_secret'))
    ->dispatch();
```

The `dispatch()` method queues the webhook call. The payload is signed with the secret using HMAC-SHA256, and the signature is included in the `Signature` header.

### Configuring Retries

External services go down. Network requests fail. DNS resolves slowly. If you send a webhook once and give up on failure, the receiver misses the event permanently. Retries with exponential backoff give the receiver time to recover while ensuring eventual delivery:

```php
WebhookCall::create()
    ->url('https://partner-app.com/webhooks')
    ->payload(['event' => 'order.shipped', 'data' => $data])
    ->useSecret($secret)
    ->maximumTries(5)
    ->timeoutInSeconds(10)
    ->withHeaders([
        'X-Webhook-Source' => 'your-app',
        'X-Webhook-Event' => 'order.shipped',
    ])
    ->dispatch();
```

### Wrapping Webhook Sending in a Service

If your application sends webhooks from multiple places — when an order ships, when a user upgrades, when an invoice is generated — scattering `WebhookCall::create()` calls throughout your codebase leads to duplicated configuration and inconsistent payloads. Encapsulate it in a [service](/books/clean-code-in-laravel/organizing-your-application):

```php
<?php

namespace App\Services;

use App\Models\WebhookSubscription;
use Spatie\WebhookServer\WebhookCall;

class WebhookDispatcher
{
    public function dispatch(string $event, array $data): void
    {
        $subscribers = WebhookSubscription::query()
            ->where('is_active', true)
            ->whereJsonContains('events', $event)
            ->get();

        foreach ($subscribers as $subscriber) {
            WebhookCall::create()
                ->url($subscriber->url)
                ->payload([
                    'event' => $event,
                    'data' => $data,
                    'timestamp' => now()->toIso8601String(),
                ])
                ->useSecret($subscriber->secret)
                ->maximumTries(5)
                ->timeoutInSeconds(10)
                ->meta(['subscriber_id' => $subscriber->id])
                ->dispatch();
        }
    }
}
```

Now sending a webhook from anywhere in your application is a single call:

```php
app(WebhookDispatcher::class)->dispatch('order.shipped', [
    'order_id' => $order->id,
    'tracking_number' => $order->tracking_number,
]);
```

### Listening for Webhook Events

When you send webhooks to external services, you have no visibility into what happens after the request leaves your server. Did the partner receive it? Did all five retries fail? Without monitoring, a broken webhook endpoint goes unnoticed until a partner complains — days or weeks later.

Spatie's server package dispatches events for every delivery attempt, giving you hooks for logging and alerting:

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\Event;
use Spatie\WebhookServer\Events\WebhookCallSucceededEvent;
use Spatie\WebhookServer\Events\FinalWebhookCallFailedEvent;

public function boot(): void
{
    Event::listen(WebhookCallSucceededEvent::class, function (WebhookCallSucceededEvent $event): void {
        Log::info('Webhook delivered', [
            'url' => $event->webhookUrl,
            'payload' => $event->payload,
        ]);
    });

    Event::listen(FinalWebhookCallFailedEvent::class, function (FinalWebhookCallFailedEvent $event): void {
        Log::error('Webhook permanently failed', [
            'url' => $event->webhookUrl,
            'payload' => $event->payload,
        ]);

        // Optionally disable the subscriber after repeated failures
    });
}
```

## Testing Webhooks

Webhooks cross system boundaries — they come from external services you do not control, arrive at unpredictable times, and carry payloads that can change without warning. This makes them one of the most important things to test. You want to verify the full chain: signature validation rejects forged requests, the webhook profile filters irrelevant events, and the processing [job](/books/clean-code-in-laravel/jobs) delegates to the right [Action](/books/clean-code-in-laravel/actions).

Testing webhook receiving is straightforward because you control the incoming request:

```php
use function Pest\Laravel\postJson;

it('processes a stripe payment succeeded webhook', function (): void {
    $payload = [
        'type' => 'payment_intent.succeeded',
        'data' => [
            'object' => [
                'id' => 'pi_test_123',
                'amount' => 5000,
                'currency' => 'usd',
            ],
        ],
    ];

    $signature = computeStripeSignature($payload);

    postJson('/webhooks/stripe', $payload, [
        'Stripe-Signature' => $signature,
    ])->assertOk();

    expect(Payment::where('stripe_payment_intent_id', 'pi_test_123')->exists())
        ->toBeTrue();
});

it('rejects webhooks with invalid signatures', function (): void {
    postJson('/webhooks/stripe', ['type' => 'test'], [
        'Stripe-Signature' => 'invalid',
    ])->assertStatus(500);
});

it('ignores unsubscribed event types', function (): void {
    $payload = ['type' => 'balance.available'];
    $signature = computeStripeSignature($payload);

    postJson('/webhooks/stripe', $payload, [
        'Stripe-Signature' => $signature,
    ])->assertOk();

    // No processing occurred - the profile filtered it out
    expect(WebhookCall::count())->toBe(0);
});
```

## The Clean Webhook Checklist

1. **Always verify signatures** — never skip, even in development. An unverified webhook is an open door for attackers to trigger actions in your application
2. **Always process asynchronously** — return a fast response and handle the payload in a queued [job](/books/clean-code-in-laravel/jobs). Synchronous processing risks timeouts and duplicate retries
3. **Always check for duplicates** — webhook senders retry on failure, so your processing must be idempotent. Use `updateOrCreate` or check for existing records before creating new ones
4. **Filter irrelevant events early** — use webhook profiles to reject events your application does not care about before they reach the queue
5. **Delegate to [Action](/books/clean-code-in-laravel/actions) classes** — the processing job should route events to Actions, not contain business logic itself
6. **Let jobs fail and retry** — do not swallow exceptions. Let the job fail so the queue worker retries it. Log permanent failures for investigation
7. **Wrap sending in a service** — centralize webhook dispatch logic to ensure consistent payloads, signing, and retry configuration
8. **Sign outgoing payloads** — always use HMAC-SHA256 signing so receivers can verify authenticity
9. **Retry with backoff** — external services go down. Configure retries with exponential backoff for reliable eventual delivery
10. **Clean up old records** — set `delete_after_days` in the webhook client config to avoid unbounded table growth

## Summary

- A webhook is an HTTP request sent from one application to another the moment something happens — no polling, no cron jobs. Your application needs to handle both receiving webhooks from external services and sending them to partners.
- [`spatie/laravel-webhook-client`](https://spatie.be/docs/laravel-webhook-client) provides a structured approach to receiving webhooks: route registration, signature verification, event filtering through webhook profiles, and queued processing.
- Always verify webhook signatures. An unverified webhook is an open door — any attacker who knows your endpoint can trigger actions in your application.
- Use webhook profiles to filter irrelevant events before they reach the queue. Only store and process events your application actually cares about.
- Always process webhooks asynchronously in queued [jobs](/books/clean-code-in-laravel/jobs). External services expect a fast response and will retry on timeout, which can create duplicate processing.
- Make webhook processing idempotent. Check for existing records before creating new ones, because senders will retry and your application must handle the same event multiple times without side effects.
- The processing job should act as a router, delegating to [Action](/books/clean-code-in-laravel/actions) classes for business logic. Each Action is independently testable.
- [`spatie/laravel-webhook-server`](https://spatie.be/docs/laravel-webhook-server) handles the sending side — payload signing, queued delivery, and retries with backoff. Wrap sending logic in a service to keep payloads and configuration consistent.
- Listen for webhook delivery events (`WebhookCallSucceededEvent`, `FinalWebhookCallFailedEvent`) to monitor delivery health and alert on permanent failures.
- Test the full webhook chain: signature validation rejects forged requests, profiles filter irrelevant events, and the processing job delegates to the correct Action.

## References

- [spatie/laravel-webhook-client](https://github.com/spatie/laravel-webhook-client) — Spatie, GitHub
- [spatie/laravel-webhook-server](https://github.com/spatie/laravel-webhook-server) — Spatie, GitHub
- [Laravel Webhook Client Documentation](https://spatie.be/docs/laravel-webhook-client/v3/introduction) — Spatie Documentation
- [Laravel Webhook Server Documentation](https://spatie.be/docs/laravel-webhook-server/v3/introduction) — Spatie Documentation
- [Handling Webhooks in a Laravel Application](https://freek.dev/1542-handling-webhooks-in-a-laravel-application) — Freek Van der Herten
- [Sending Webhooks with Laravel](https://freek.dev/1566-sending-webhooks-from-a-laravel-application) — Freek Van der Herten
- [Queues](https://laravel.com/docs/queues) — Laravel Documentation
- [Stripe Webhooks](https://docs.stripe.com/webhooks) — Stripe Documentation
- [Best Practices for Using Webhooks](https://docs.stripe.com/webhooks/best-practices) — Stripe Documentation
