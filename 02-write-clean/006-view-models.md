[Controllers](/books/clean-code-in-laravel/controllers) should be thin. Models should not contain presentation logic. So where do you prepare the data that a complex view needs?

Consider a dashboard page that shows a user's recent orders, their subscription status, a list of recommended products, and some statistics. Without a View Model, the controller becomes a data-fetching mess:

```php
// Bad: the controller is doing too much data preparation
public function index(Request $request): View
{
    $user = $request->user();
    $recentOrders = $user->orders()->latest()->limit(5)->get();
    $subscription = $user->subscription;
    $isTrialing = $subscription?->onTrial() ?? false;
    $daysLeft = $isTrialing ? now()->diffInDays($subscription->trial_ends_at) : null;
    $recommendedProducts = Product::recommended($user)->limit(8)->get();
    $orderStats = [
        'total_spent' => $user->orders()->sum('total'),
        'order_count' => $user->orders()->count(),
        'average_order' => $user->orders()->avg('total'),
    ];
    $notifications = $user->unreadNotifications()->limit(10)->get();

    return view('dashboard', compact(
        'user', 'recentOrders', 'subscription', 'isTrialing',
        'daysLeft', 'recommendedProducts', 'orderStats', 'notifications',
    ));
}
```

This controller is not handling business logic — it is preparing view data. That is a different responsibility, and it deserves its own class.

Another common anti-pattern is putting presentation logic on the model itself — methods like `formattedPrice()`, `statusBadgeColor()`, or `displayName()` on an [Eloquent model](/books/clean-code-in-laravel/eloquent-models-done-right). Models should represent data and relationships, not know how a specific page wants to display them. A product might show its price differently on an invoice than on a catalog page. That formatting logic belongs in a View Model, not on the model.

## What Is a View Model?

A View Model is a class that prepares all the data a specific view needs. It encapsulates the queries, calculations, and transformations required to render a page. The controller creates the View Model, and the view consumes it.

```php
namespace App\ViewModels;

use App\Models\Product;
use App\Models\User;
use Illuminate\Support\Collection;

class DashboardViewModel
{
    public function __construct(
        private readonly User $user,
    ) {}

    /** @return Collection<int, Order> */
    public function recentOrders(): Collection
    {
        return $this->user->orders()
            ->with('items.product')
            ->latest()
            ->limit(5)
            ->get();
    }

    public function isTrialing(): bool
    {
        return $this->user->subscription?->onTrial() ?? false;
    }

    public function trialDaysLeft(): ?int
    {
        if (! $this->isTrialing()) {
            return null;
        }

        return now()->diffInDays($this->user->subscription->trial_ends_at);
    }

    /** @return Collection<int, Product> */
    public function recommendedProducts(): Collection
    {
        return Product::recommended($this->user)->limit(8)->get();
    }

    public function totalSpent(): float
    {
        return $this->user->orders()->sum('total');
    }

    public function orderCount(): int
    {
        return $this->user->orders()->count();
    }

    public function averageOrderValue(): float
    {
        return $this->user->orders()->avg('total') ?? 0;
    }

    /** @return Collection<int, DatabaseNotification> */
    public function unreadNotifications(): Collection
    {
        return $this->user->unreadNotifications()->limit(10)->get();
    }
}
```

Now the [controller](/books/clean-code-in-laravel/controllers) is clean:

```php
public function index(Request $request): View
{
    return view('dashboard', [
        'view' => new DashboardViewModel($request->user()),
    ]);
}
```

And the Blade template accesses data through the View Model:

```blade
<h2>Recent Orders</h2>
@foreach ($view->recentOrders() as $order)
    <div>{{ $order->id }} — ${{ $order->total }}</div>
@endforeach

@if ($view->isTrialing())
    <div class="alert">
        Your trial ends in {{ $view->trialDaysLeft() }} days.
    </div>
@endif

<div class="stats">
    <span>Total Spent: ${{ number_format($view->totalSpent(), 2) }}</span>
    <span>Orders: {{ $view->orderCount() }}</span>
    <span>Average: ${{ number_format($view->averageOrderValue(), 2) }}</span>
</div>
```

## Using Spatie's View Models Package

[`spatie/laravel-view-models`](https://github.com/spatie/laravel-view-models) adds a convenient base class. View Models that extend it can be returned directly from controllers:

```bash
composer require spatie/laravel-view-models
```

```php
namespace App\ViewModels;

use Spatie\ViewModels\ViewModel;

class OrderIndexViewModel extends ViewModel
{
    public function __construct(
        private readonly User $user,
        private readonly ?string $status = null,
    ) {}

    public function orders(): LengthAwarePaginator
    {
        return $this->user->orders()
            ->when($this->status, fn (Builder $q, string $status): Builder => $q->where('status', $status))
            ->with('items.product')
            ->latest()
            ->paginate(15);
    }

    public function statusOptions(): array
    {
        return OrderStatus::cases();
    }

    public function currentStatus(): ?string
    {
        return $this->status;
    }
}
```

With Spatie's package, every public method becomes a variable in the view. The controller returns the View Model directly:

```php
public function index(Request $request): OrderIndexViewModel
{
    return new OrderIndexViewModel(
        user: $request->user(),
        status: $request->query('status'),
    );
}
```

The view automatically receives `$orders`, `$statusOptions`, and `$currentStatus` as variables — one for each public method.

## View Models for Forms

View Models are especially useful for forms that need dropdown data, defaults, and existing values:

```php
class OrderEditViewModel extends ViewModel
{
    public function __construct(
        private readonly Order $order,
    ) {}

    public function order(): Order
    {
        return $this->order;
    }

    public function statusOptions(): array
    {
        return OrderStatus::cases();
    }

    /** @return Collection<int, Address> */
    public function shippingAddresses(): Collection
    {
        return $this->order->user->addresses()
            ->orderBy('is_default', 'desc')
            ->get();
    }

    /** @return Collection<int, Coupon> */
    public function availableCoupons(): Collection
    {
        return Coupon::where('expires_at', '>', now())
            ->where('is_active', true)
            ->get();
    }
}
```

Without a View Model, all of this data-fetching would clutter the controller's `edit()` method. Notice how `OrderStatus` is an [enum](/books/clean-code-in-laravel/enums-value-objects-and-type-safety) — View Models pair naturally with enums to provide type-safe options for dropdowns and filters.

## View Models vs. View Composers

Laravel also has [View Composers](https://laravel.com/docs/views#view-composers) — callbacks that inject data into a view every time it renders. They solve a different problem:

| | View Model | View Composer |
|---|---|---|
| **Scope** | One specific page or endpoint | Any view that matches a pattern |
| **Created by** | The controller, explicitly | The service provider, globally |
| **Use case** | Dashboard data, form dropdowns | Navigation menu, auth user, notifications |
| **Testability** | Unit test directly | Requires rendering the view |

View Composers are best for data that appears on *every* page — the logged-in user's name in the navbar, a global notification count, or footer links. If you find yourself registering a View Composer for a single view, use a View Model instead.

The two patterns complement each other. Use View Composers for global, shared data. Use View Models for page-specific data preparation.

## Testing View Models

Because View Models are plain PHP classes, they are straightforward to test. No HTTP requests needed — just instantiate the class and call its methods:

```php
it('returns the correct trial days remaining', function (): void {
    $user = User::factory()
        ->has(Subscription::factory()->state([
            'trial_ends_at' => now()->addDays(7),
        ]))
        ->create();

    $viewModel = new DashboardViewModel($user);

    expect($viewModel->isTrialing())->toBeTrue();
    expect($viewModel->trialDaysLeft())->toBe(7);
});

it('returns null trial days when not trialing', function (): void {
    $user = User::factory()->create();

    $viewModel = new DashboardViewModel($user);

    expect($viewModel->isTrialing())->toBeFalse();
    expect($viewModel->trialDaysLeft())->toBeNull();
});
```

This is one of the main benefits of View Models over putting data preparation in [controllers](/books/clean-code-in-laravel/controllers) — you can test the logic in isolation without making HTTP requests or asserting against rendered HTML.

## Organizing View Models

View Models sit at the end of the data flow: [Controllers](/books/clean-code-in-laravel/controllers) call [Actions](/books/clean-code-in-laravel/actions), Actions return domain objects or [DTOs](/books/clean-code-in-laravel/data-transfer-objects), and View Models transform that data for the view.

Place View Models in `app/ViewModels`, organized by feature or page:

```
app/ViewModels/
├── DashboardViewModel.php
├── Order/
│   ├── OrderIndexViewModel.php
│   ├── OrderShowViewModel.php
│   └── OrderEditViewModel.php
└── User/
    ├── UserProfileViewModel.php
    └── UserSettingsViewModel.php
```

## When to Use View Models

| Situation | Use View Model? |
|---|---|
| Simple page with one model | No — pass the model directly |
| Page with multiple data sources | Yes |
| Form with dropdown options | Yes |
| Dashboard with statistics | Yes |
| Inertia.js page with complex props | Yes |
| Data shared across every page | No — use a View Composer |
| Page that just lists one resource | Probably not |

The rule of thumb: if your controller method needs more than two lines of data preparation, extract it into a View Model. Your controllers stay thin, your views stay clean, and your data preparation logic is testable in isolation.

## The View Model Checklist

1. **One View Model per page** — name it after the view: `DashboardViewModel`, `OrderEditViewModel`
2. **Keep presentation logic off models** — methods like `statusBadgeColor()` or `formattedPrice()` belong in View Models, not on [Eloquent models](/books/clean-code-in-laravel/eloquent-models-done-right)
3. **Pass dependencies via the constructor** — the controller creates the View Model with the data it needs
4. **Use View Composers for shared data** — navigation, auth user, footer links. View Models are for page-specific data
5. **Test in isolation** — instantiate the class and call methods directly. No HTTP requests needed

## Summary

- A View Model is a class that prepares all the data a specific view needs — queries, calculations, and transformations live here, not in the controller.
- Keep presentation logic out of Eloquent models. Methods like `statusBadgeColor()` or `formattedPrice()` belong in a View Model, not on the model itself.
- [`spatie/laravel-view-models`](https://github.com/spatie/laravel-view-models) turns public methods into view variables automatically and supports returning View Models directly from controllers.
- View Composers inject data into *every* render of a view (navbar, footer). View Models prepare data for *one specific* page. Use both when appropriate.
- View Models are plain PHP classes, making them easy to unit test without HTTP requests or rendered HTML.
- Use a View Model when your controller needs more than two lines of data preparation. For simple pages with one model, pass the model directly.

## References

- [View Models in Laravel](https://stitcher.io/blog/laravel-view-models) — Brent Roose
- [Laravel View Models vs. View Composers](https://stitcher.io/blog/laravel-view-models-vs-view-composers) — Brent Roose
- [spatie/laravel-view-models](https://github.com/spatie/laravel-view-models) — Spatie, GitHub
- [Laravel View Models](https://laravel-news.com/laravel-view-models) — Laravel News
- [Adding View Models to a Laravel Project](https://davidllop.com/posts/adding-view-models-to-a-laravel-project) — David Llop
- [Views: View Composers](https://laravel.com/docs/views#view-composers) — Laravel Documentation
