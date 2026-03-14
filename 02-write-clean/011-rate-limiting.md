Rate limiting is one of those things that sounds boring until you need it. An API consumer hammers your endpoint 10,000 times in a minute. A background job hits a third-party service so fast it gets your account banned. A login form gets brute-forced. Suddenly, rate limiting is the most important thing in your application.

Laravel has built-in rate limiting for HTTP routes and queued jobs. For finer control over job throttling, Spatie's [laravel-rate-limited-job-middleware](https://github.com/spatie/laravel-rate-limited-job-middleware) provides a fluent API with configurable time windows and backoff strategies. This chapter covers both, with practical patterns for keeping your application well-behaved.

## Rate limiting HTTP routes

Laravel's rate limiter lives in the `RateLimiter` facade. You define named limiters in the `boot` method of your `AppServiceProvider`, then attach them to routes using the `throttle` middleware.

### Defining rate limiters

A rate limiter is a named configuration that describes how many requests are allowed in a given time window:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    RateLimiter::for('api', function (Request $request): Limit {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
}
```

The `by` method determines what the limit applies to. In this example, each authenticated user gets 60 requests per minute. Unauthenticated users are limited by IP address instead.

You can return different limits based on the user:

```php
RateLimiter::for('uploads', function (Request $request): Limit {
    return $request->user()?->isPremium()
        ? Limit::perMinute(100)->by($request->user()->id)
        : Limit::perMinute(10)->by($request->user()?->id ?: $request->ip());
});
```

Premium users get 100 uploads per minute. Everyone else gets 10. The logic is in one place, not scattered across controllers.

### Multiple rate limits

Sometimes a single limit is not enough. You might want to allow 10 requests per minute but also cap at 1000 per day:

```php
RateLimiter::for('sensitive', function (Request $request): array {
    return [
        Limit::perMinute(10)->by('minute:' . $request->user()->id),
        Limit::perDay(1000)->by('day:' . $request->user()->id),
    ];
});
```

Each limit is evaluated independently. If either one is exceeded, the request is rejected. Notice the prefixed `by` keys — when using multiple limits with the same user ID, you need unique keys to prevent them from sharing the same counter.

### Attaching to routes

Attach a rate limiter to routes using the `throttle` middleware:

```php
Route::middleware(['throttle:api'])->group(function (): void {
    Route::get('/users', [UserController::class, 'index']);
    Route::post('/users', [UserController::class, 'store']);
});

Route::middleware(['throttle:uploads'])->group(function (): void {
    Route::post('/photos', [PhotoController::class, 'store']);
    Route::post('/documents', [DocumentController::class, 'store']);
});
```

When a request exceeds the limit, Laravel automatically returns a `429 Too Many Requests` response with `Retry-After` and `X-RateLimit-*` headers. You do not need to handle this yourself.

### Inline rate limits

Not every endpoint needs a named rate limiter. For simple cases, you can pass the maximum number of attempts and the decay period in minutes directly to the `throttle` middleware:

```php
// 6 attempts per minute
Route::middleware(['throttle:6,1'])->group(function (): void {
    Route::post('/contact', [ContactController::class, 'store']);
});

// 3 attempts per 5 minutes
Route::post('/forgot-password', [ForgotPasswordController::class, 'store'])
    ->middleware('throttle:3,5');
```

The first parameter is the maximum number of attempts, and the second is the decay time in minutes. If you omit the second parameter, it defaults to 1 minute:

```php
// 10 attempts per minute
Route::middleware(['throttle:10'])->group(function (): void {
    Route::post('/comments', [CommentController::class, 'store']);
});
```

This approach is convenient for quick, one-off limits where you do not need per-user logic, conditional limits, or custom responses. Once you need any of those, define a named rate limiter instead.

### Custom responses

If the default 429 response does not fit your API design, customize it with the `response` method:

```php
RateLimiter::for('api', function (Request $request): Limit {
    return Limit::perMinute(60)
        ->by($request->user()?->id ?: $request->ip())
        ->response(function (Request $request, array $headers): JsonResponse {
            return response()->json([
                'message' => 'You have exceeded the rate limit. Please try again later.',
            ], 429, $headers);
        });
});
```

### Response-based rate limiting

Laravel also supports rate limiting based on the response, not just the request. This is useful when you only want to count certain outcomes toward the limit:

```php
use Symfony\Component\HttpFoundation\Response;

RateLimiter::for('resource-not-found', function (Request $request): Limit {
    return Limit::perMinute(10)
        ->by($request->user()?->id ?: $request->ip())
        ->after(function (Response $response): bool {
            return $response->status() === 404;
        });
});
```

Only 404 responses count toward the limit. This prevents enumeration attacks — someone trying thousands of IDs to discover which resources exist — without punishing users who make legitimate requests.

### Using Redis for rate limiting

By default, Laravel uses its cache driver for rate limiting. If you use Redis, you can get better performance by switching to the Redis-specific throttle middleware in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->throttleWithRedis();
})
```

This uses `ThrottleRequestsWithRedis` instead of the default `ThrottleRequests`, which handles concurrent requests more accurately using Redis's atomic operations.

## Rate limiting in application code

The `RateLimiter` facade is not limited to HTTP routes. You can use it anywhere in your application:

```php
use Illuminate\Support\Facades\RateLimiter;

$executed = RateLimiter::attempt(
    key: 'send-notification:' . $user->id,
    maxAttempts: 5,
    callback: function () use ($user, $order): void {
        $user->notify(new OrderShipped($order));
    },
);

if (! $executed) {
    // User has been notified too many times, skip
}
```

The `attempt` method returns `false` when the limit is exceeded, and the callback's return value (or `true`) otherwise. This is useful for rate limiting anything: notifications, emails, API calls to external services, or even database-intensive operations.

## Rate limiting queued jobs

HTTP rate limiting protects your application from external abuse. But what about internal abuse — your own queued jobs overwhelming a third-party API?

Laravel ships with a built-in `RateLimited` job middleware, but Spatie's [laravel-rate-limited-job-middleware](https://github.com/spatie/laravel-rate-limited-job-middleware) gives you more control with a cleaner API.

### Installation

```bash
composer require spatie/laravel-rate-limited-job-middleware
```

The package requires Redis.

### Basic usage

Add the middleware to your job's `middleware` method:

```php
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Spatie\RateLimitedMiddleware\RateLimited;

class ImportProductsJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function middleware(): array
    {
        return [new RateLimited()];
    }

    public function handle(): void
    {
        // Import products from external API
    }
}
```

By default, the middleware allows 5 jobs per second. Jobs that exceed the limit are released back to the queue and retried after 5 seconds.

### Customizing limits

The middleware has a fluent API for configuring exactly how many jobs can run in what time window:

```php
public function middleware(): array
{
    return [
        (new RateLimited())
            ->allow(30)
            ->everySeconds(60)
            ->releaseAfterSeconds(90),
    ];
}
```

This allows 30 jobs in a 60-second window. Jobs that exceed the limit are released back to the queue and retried after 90 seconds.

Other time helpers include `everyMinute()`, `everyMinutes(5)`, and `everySecond()`.

### Keying rate limits

By default, the rate limit is keyed by the job class name. All instances of `ImportProductsJob` share the same limit. If you need different keys — for example, one limit per external API provider — use the `key` method:

```php
public function middleware(): array
{
    return [
        (new RateLimited())
            ->allow(10)
            ->everyMinute()
            ->key('api-provider:' . $this->provider->id),
    ];
}
```

Now each provider gets its own rate limit of 10 jobs per minute.

### Exponential backoff

When a rate limit is hit, you might want to back off gradually instead of using a fixed delay:

```php
public function middleware(): array
{
    return [
        (new RateLimited())
            ->allow(5)
            ->everySecond()
            ->releaseAfterBackoff($this->attempts()),
    ];
}
```

The delay increases exponentially with each attempt: 1 second, 2 seconds, 4 seconds, 8 seconds, and so on. This prevents a thundering herd of jobs all retrying at the same moment.

### Time-based retry windows

When rate limiting jobs, do not use a fixed number of retries. A job might get released dozens of times before the rate limit clears. Instead, use a time-based retry window:

```php
public function retryUntil(): DateTime
{
    return now()->addHours(24);
}
```

This tells Laravel to keep retrying the job for up to 24 hours, regardless of how many times it gets released by the rate limiter.

### Dropping instead of retrying

Sometimes you want to skip a rate-limited job entirely instead of retrying it. A real-time analytics job that is stale by the time it retries is a good example:

```php
public function middleware(): array
{
    return [
        (new RateLimited())
            ->allow(100)
            ->everyMinute()
            ->dontRelease(),
    ];
}
```

The `dontRelease` method silently drops jobs that exceed the limit instead of putting them back on the queue.

## Organizing rate limiter definitions

When your application has many rate limiters, the `AppServiceProvider` boot method gets crowded. Extract them into a dedicated method or a separate service provider:

```php
class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->configureRateLimiting();
    }

    private function configureRateLimiting(): void
    {
        RateLimiter::for('api', function (Request $request): Limit {
            return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
        });

        RateLimiter::for('uploads', function (Request $request): Limit {
            return $request->user()?->isPremium()
                ? Limit::perMinute(100)->by($request->user()->id)
                : Limit::perMinute(10)->by($request->user()?->id ?: $request->ip());
        });

        RateLimiter::for('login', function (Request $request): array {
            return [
                Limit::perMinute(5)->by($request->input('email')),
                Limit::perMinute(30)->by($request->ip()),
            ];
        });
    }
}
```

The rate limiters are still registered in one place, but the boot method stays clean.

> Rate limiting is a cross-cutting concern. Define your limits where they are easy to find, attach them declaratively to routes and jobs, and let the framework handle the rest. Your controllers and job handlers should never contain rate limiting logic.

## Summary

- Rate limiting protects your application from both external abuse (HTTP requests) and internal abuse (queued jobs overwhelming third-party APIs).
- Define named rate limiters in `AppServiceProvider` using `RateLimiter::for()`, then attach them to routes with the `throttle` middleware.
- Use `->by()` to scope limits per user, IP, or any other key. Return different limits based on user type to give premium users higher thresholds.
- Return an array of `Limit` objects for layered limits (e.g., per-minute and per-day). Use unique `by` prefixes to keep counters separate.
- Laravel returns a `429 Too Many Requests` response automatically. Customize it with the `response` method on the `Limit` object.
- Use response-based rate limiting with the `after` method to only count specific outcomes — like 404 responses — toward the limit. This prevents enumeration attacks without punishing legitimate requests.
- Switch to `throttleWithRedis()` in `bootstrap/app.php` for better performance and accuracy under concurrent load.
- `RateLimiter::attempt()` works anywhere in your application — not just HTTP routes. Use it for notifications, emails, or external API calls.
- For queued jobs, Spatie's [laravel-rate-limited-job-middleware](https://github.com/spatie/laravel-rate-limited-job-middleware) provides a fluent API with `allow()`, `everySeconds()`, and `releaseAfterSeconds()` for fine-grained control.
- Use `releaseAfterBackoff()` for exponential delays instead of fixed retry intervals. Use `dontRelease()` to silently drop stale jobs that would be irrelevant by the time they retry.
- Always use time-based retry windows (`retryUntil`) instead of fixed retry counts for rate-limited jobs, since a job may be released many times before the limit clears.
- Keep rate limiter definitions organized — extract them into a dedicated method when your `AppServiceProvider` grows.

## References

- [Rate Limiting](https://laravel.com/docs/rate-limiting) — Laravel Documentation
- [Routing: Rate Limiting](https://laravel.com/docs/routing#rate-limiting) — Laravel Documentation
- [Queues: Job Middleware](https://laravel.com/docs/queues#job-middleware) — Laravel Documentation
- [spatie/laravel-rate-limited-job-middleware](https://github.com/spatie/laravel-rate-limited-job-middleware) — Spatie, GitHub
- [A Job Middleware to Rate Limit Jobs](https://freek.dev/1475-a-job-middleware-to-rate-limit-jobs) — Freek Van der Herten
- [Laravel Rate Limiting](https://laravel-news.com/laravel-rate-limiting) — Laravel News
- [Redis Rate Limiting in Laravel](https://laravel-news.com/redis-throttle) — Laravel News
