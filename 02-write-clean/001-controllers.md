A [controller](https://laravel.com/docs/controllers) has one job: take an HTTP request and return a response. It should receive the request, hand the work off to the right class, and send back the result. That is it. A controller is a traffic cop, not a factory worker.

[Taylor Otwell](https://github.com/taylorotwell) has been consistent on this point. In [Laravel Fortify](https://github.com/laravel/fortify/tree/1.x/src/Http/Controllers) for example, controllers are remarkably thin - often just a few lines. They validate input (via [Form Requests](https://laravel.com/docs/validation#form-request-validation)), call an Action or Service, and return a redirect or view. The business logic lives elsewhere.

Here is an example of the `EmailVerificationPromptController` in Fortify — an [invokable controller](#cruddy-by-design) with a single `__invoke` method:

```php
<?php
class EmailVerificationPromptController extends Controller
{
    public function __invoke(Request $request)
    {
        return $request->user()->hasVerifiedEmail()
            ? app(RedirectAsIntended::class, ['name' => 'email-verification'])
            : app(VerifyEmailViewResponse::class);
    }
}
```

## Fat Controllers

Here is a controller that does too much. This is the kind of code that starts small and grows into a problem over six months of feature additions:

```php
class OrderController extends Controller
{
    public function store(Request $request): RedirectResponse
    {
        // Validation in the controller
        $request->validate([
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'exists:products,id'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
            'shipping_address_id' => ['required', 'exists:addresses,id'],
            'coupon_code' => ['nullable', 'string'],
        ]);

        // Business logic in the controller
        $total = 0;
        foreach ($request->items as $item) {
            $product = Product::findOrFail($item['product_id']);
            if ($product->stock < $item['quantity']) {
                return back()->withErrors(['items' => "Not enough stock for {$product->name}"]);
            }
            $total += $product->price * $item['quantity'];
        }

        // Coupon logic in the controller
        if ($request->coupon_code) {
            $coupon = Coupon::where('code', $request->coupon_code)->first();
            if ($coupon && $coupon->isValid()) {
                $total = $total - ($total * $coupon->discount_percentage / 100);
            }
        }

        // Tax calculation in the controller
        $tax = $total * 0.21;
        $total += $tax;

        // Order creation in the controller
        $order = Order::create([
            'user_id' => auth()->id(),
            'shipping_address_id' => $request->shipping_address_id,
            'subtotal' => $total - $tax,
            'tax' => $tax,
            'total' => $total,
            'status' => 'pending',
        ]);

        foreach ($request->items as $item) {
            $product = Product::findOrFail($item['product_id']);
            $order->items()->create([
                'product_id' => $product->id,
                'quantity' => $item['quantity'],
                'unit_price' => $product->price,
            ]);
            $product->decrement('stock', $item['quantity']);
        }

        // Notification in the controller
        $order->user->notify(new OrderPlacedNotification($order));
        Mail::to(config('shop.admin_email'))->send(new NewOrderMail($order));

        return redirect()->route('orders.show', $order)
            ->with('success', 'Order placed successfully!');
    }
}
```

This controller is doing at least seven things: validating, checking stock, applying coupons, calculating tax, creating the order, updating inventory, and sending notifications. It is untestable in isolation, impossible to reuse, and painful to modify. If the coupon logic needs to change, you have to touch the same file that handles order creation. If the tax calculation is wrong, you are debugging inside a 60-line method.

## Thin Controllers

Here is the same functionality with a thin controller. The controller delegates everything:

```php
class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, PlaceOrderAction $action): RedirectResponse
    {
        $order = $action->execute($request->toDto());

        return redirect()->route('orders.show', $order)
            ->with('success', 'Order placed successfully!');
    }
}
```

Three lines. The validation is handled by `StoreOrderRequest`. The business logic is handled by `PlaceOrderAction`. The controller just connects them and returns a redirect. We cover Actions in detail in the [next chapter](/books/clean-code-in-laravel/actions).

The key difference is not just about line count - it is about testability. The fat controller can only be tested through an HTTP request. The thin version lets you test validation rules separately in `StoreOrderRequest`, test the order logic separately in `PlaceOrderAction`, and test the controller's HTTP behavior on its own.

## Resource Controllers

[Resource controllers](https://laravel.com/docs/controllers#resource-controllers) give you seven methods that cover all standard CRUD operations. Stick to these methods whenever possible:

| Method | HTTP Verb | URI | Purpose |
|---|---|---|---|
| `index` | GET | `/orders` | List all orders |
| `create` | GET | `/orders/create` | Show creation form |
| `store` | POST | `/orders` | Save new order |
| `show` | GET | `/orders/{order}` | Show single order |
| `edit` | GET | `/orders/{order}/edit` | Show edit form |
| `update` | PUT/PATCH | `/orders/{order}` | Update existing order |
| `destroy` | DELETE | `/orders/{order}` | Delete order |

```php
// routes/web.php
Route::resource('orders', OrderController::class);
```

One line in your routes file gives you all seven routes with proper naming (`orders.index`, `orders.store`, etc.) and correct HTTP verbs. If you only need some of these methods, use `only` or `except`:

```php
// Only registration - no editing or deleting
Route::resource('orders', OrderController::class)->only(['index', 'create', 'store', 'show']);

// Everything except destroy
Route::resource('orders', OrderController::class)->except(['destroy']);
```

## Cruddy by Design

If your controller needs more than the seven resource methods, it is a sign that you need more controllers - not bigger ones. In his [Cruddy by Design](https://www.youtube.com/watch?v=MF0jFKvS4SI) talk at Laracon 2017, [Adam Wathan](https://x.com/adamwathan) laid out a simple rule: never write custom actions on your controllers. Stick to the seven standard CRUD methods and create new controllers for everything else.

His argument was inspired by DHH's observation that Rails developers do not create enough controllers. The Basecamp 3 codebase had 206 controllers out of roughly 450 total classes — almost half the application. And those controllers averaged just four methods each, meaning most had only two or three actions.

The rule Adam proposed is straightforward. If you find yourself adding a `publish`, `subscribe`, or `updateCoverImage` method to a controller, stop and ask: can I model this as a standard CRUD operation on a new resource? With a little imagination, the answer is almost always yes:

- Subscribing to a podcast? That is storing a new `Subscription` — a `SubscriptionController@store`.
- Publishing a podcast? That is storing a new `PublishedPodcast` — a `PublishedPodcastController@store`.
- Unpublishing? That is destroying the published podcast — `PublishedPodcastController@destroy`.
- Updating a cover image separately? That is updating a `PodcastCoverImage` — `PodcastCoverImageController@update`.

By remodeling custom actions as standard CRUD on dedicated controllers, you end up with more controllers that are each smaller and simpler. In his demo, Adam took 19 actions spread over 2 controllers (an average of 9.5 actions per controller) and reorganized them into 19 actions spread over 6 controllers (an average of 3.2 actions per controller). The total code did not shrink, but every controller became easier to understand and maintain.

When one of these new controllers only needs a single action, use an [invokable controller](https://laravel.com/docs/controllers#single-action-controllers) — a controller with a single `__invoke` method. For example, if you need to "cancel" an order, do not add a `cancel` method to `OrderController`. Create a dedicated controller:

```php
class CancelOrderController extends Controller
{
    public function __invoke(Order $order, CancelOrderAction $action): RedirectResponse
    {
        $action->execute($order);

        return redirect()->route('orders.show', $order)
            ->with('success', 'Order cancelled.');
    }
}

// routes/web.php
Route::post('/orders/{order}/cancel', CancelOrderController::class)
    ->name('orders.cancel');
```

Instead of one growing controller with `cancel`, `archive`, `export`, `duplicate`, and `restore` methods, you get focused classes that are easy to find and easy to test:

```
app/Http/Controllers/
├── OrderController.php           // 7 CRUD methods
├── CancelOrderController.php     // Single action
├── ArchiveOrderController.php    // Single action
└── ExportOrderController.php     // Single action
```

Each file does one thing. The name tells you what it does without opening it.

## Route Model Binding

[Route model binding](https://laravel.com/docs/routing#route-model-binding) lets Laravel automatically resolve model instances from route parameters. Use it everywhere - it removes boilerplate and handles 404s for you:

```php
// Laravel resolves {user} to a User model instance
Route::get('/users/{user}', [UserController::class, 'show']);

class UserController extends Controller
{
    // $user is already a User model - no findOrFail needed
    public function show(User $user): View
    {
        return view('users.show', compact('user'));
    }
}
```

If the user does not exist, Laravel returns a 404 automatically. No `findOrFail`, no `if (!$user)` check, no manual error handling.

### Custom Keys

By default, route model binding resolves models by their primary key (`id`). If you want to resolve by a different column - like a slug - use the `{model:column}` syntax in your route:

```php
// Resolves the post by its "slug" column instead of "id"
Route::get('/posts/{post:slug}', [PostController::class, 'show']);

class PostController extends Controller
{
    public function show(Post $post): View
    {
        // $post was found by slug - no custom logic needed
        return view('posts.show', compact('post'));
    }
}
```

The URL `/posts/my-first-article` will query `posts WHERE slug = 'my-first-article'`. If no post matches, Laravel returns a 404. You do not need a custom `findBySlug` method or any manual query - the route definition handles it.

If a model should always resolve by a column other than `id`, you can define a `getRouteKeyName` method on the model instead of repeating the column in every route:

```php
class Post extends Model
{
    public function getRouteKeyName(): string
    {
        return 'slug';
    }
}

// Now {post} always resolves by slug - no need for {post:slug}
Route::get('/posts/{post}', [PostController::class, 'show']);
```

### Scoped Binding

For nested resources, use [scoped binding](https://laravel.com/docs/routing#implicit-model-binding-scoping) to make sure a child model belongs to its parent:

```php
// Scoped binding: the post must belong to the user
Route::get('/users/{user}/posts/{post}', [UserPostController::class, 'show'])
    ->scopeBindings();

class UserPostController extends Controller
{
    public function show(User $user, Post $post): View
    {
        // Laravel already verified that $post belongs to $user
        return view('posts.show', compact('user', 'post'));
    }
}
```

Without `scopeBindings()`, a user could access any post by guessing the ID - even posts that belong to someone else. Scoped binding prevents that.

You can combine custom keys with scoped binding. For example, to resolve a user's post by slug while ensuring it belongs to that user:

```php
Route::get('/users/{user}/posts/{post:slug}', [UserPostController::class, 'show'])
    ->scopeBindings();
```

Laravel will query for a post where `slug` matches and the post belongs to the given user. If either condition fails, you get a 404. No custom query, no manual check - the route definition does all the work.

If you want scoped binding on every route in a group, apply it to the group instead of repeating it on each route:

```php
Route::scopeBindings()->group(function (): void {
    Route::get('/users/{user}/posts/{post}', [UserPostController::class, 'show']);
    Route::get('/users/{user}/posts/{post}/comments/{comment}', [CommentController::class, 'show']);
});
```

## Dependency Injection

Controllers should never create their own dependencies. Instead, type-hint them in the constructor or method signature and let Laravel's [service container](https://laravel.com/docs/container) inject them:

```php
class OrderController extends Controller
{
    // Constructor injection: shared across all methods
    public function __construct(
        private readonly OrderService $orderService,
    ) {}

    // Method injection: specific to this action
    public function store(
        StoreOrderRequest $request,
        PlaceOrderAction $action,
    ): RedirectResponse {
        $order = $action->execute($request->toDto());

        return redirect()->route('orders.show', $order);
    }

    public function index(): View
    {
        $orders = $this->orderService->getOrdersForUser(auth()->user());

        return view('orders.index', compact('orders'));
    }
}
```

Never use `new` to create services inside a controller. If you write `$service = new OrderService()`, you cannot swap it in tests and you are hiding your dependencies. We cover this in detail in the [Dependency Injection](/books/clean-code-in-laravel/dependency-injection) chapter.

## API Controllers

For API endpoints, return JSON responses using [API Resources](https://laravel.com/docs/eloquent-resources). An API Resource is a transformation layer that sits between your Eloquent models and the JSON responses your API returns. You can generate one with Artisan:

```bash
php artisan make:resource OrderResource
```

This creates a class in `app/Http/Resources` where you define exactly which fields to expose:

```php
class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'status' => $this->status,
            'total' => $this->total,
            'items' => OrderItemResource::collection($this->whenLoaded('items')),
            'created_at' => $this->created_at,
        ];
    }
}
```

With the resource in place, the API controller stays just as thin as a web controller - the only difference is the return type:

```php
namespace App\Http\Controllers\Api;

class OrderController extends Controller
{
    public function index(Request $request): AnonymousResourceCollection
    {
        $orders = $request->user()
            ->orders()
            ->latest()
            ->paginate();

        return OrderResource::collection($orders);
    }

    public function store(StoreOrderRequest $request, PlaceOrderAction $action): JsonResponse
    {
        $order = $action->execute($request->toDto());

        return response()->json(
            new OrderResource($order),
            Response::HTTP_CREATED,
        );
    }
}
```

Register these routes with `apiResource` instead of `resource` — it excludes the `create` and `edit` methods since APIs do not serve HTML forms:

```php
// routes/api.php
Route::apiResource('orders', OrderController::class);
```

Notice that the `StoreOrderRequest` and `PlaceOrderAction` are the same classes the web controller uses. The only thing that changes is the response format. This is one of the benefits of keeping business logic out of controllers - you can reuse it across web and API without duplicating anything.

For a deeper dive into API design — including resource collections, flexible filtering and sorting with Spatie Query Builder, error responses, and API versioning — see the [Clean APIs](/books/clean-code-in-laravel/clean-apis) chapter.

## The Thin Controller Checklist

Before committing a controller, check these rules:

1. No business logic - calculations, conditionals, and data transformations belong in [Actions](/books/clean-code-in-laravel/actions) or [Services](/books/clean-code-in-laravel/organizing-your-application)
2. No complex queries - anything beyond a simple lookup belongs in a scope, query builder, or service
3. No inline validation - use [Form Requests](https://laravel.com/docs/validation#form-request-validation)
4. No more than seven resource methods - need more? Model it as CRUD on a new controller, or create an invokable controller for single actions
5. Every method fits on one screen - if you need to scroll, the method is doing too much
6. Dependencies are injected - no `new` keyword, no static calls to your own classes

A thin controller is a sign of a well-organized application. If your controllers are fat, it means logic that belongs in [Actions](/books/clean-code-in-laravel/actions), [Services](/books/clean-code-in-laravel/organizing-your-application), Form Requests, or Models has leaked into the wrong place. The [next chapter](/books/clean-code-in-laravel/actions) shows you exactly how to build those Action classes.

## Summary

- A controller's only job is to connect an HTTP request to a response. It should receive, delegate, and return - nothing more.
- Fat controllers are untestable and hard to change. When validation, business logic, and notifications all live in the same method, every change is risky.
- Thin controllers delegate everything. Validation goes in Form Requests. Business logic goes in [Actions](/books/clean-code-in-laravel/actions) or [Services](/books/clean-code-in-laravel/organizing-your-application). The controller just wires them together.
- Be [Cruddy by Design](https://www.youtube.com/watch?v=MF0jFKvS4SI). Stick to the seven resource methods and never write custom actions. If you need a `publish` or `subscribe` method, model it as standard CRUD on a new controller. When that controller only needs a single action, make it an invokable controller.
- Use route model binding. Let Laravel resolve models from route parameters and handle 404s automatically. Use scoped bindings for nested resources.
- Inject dependencies - never use `new`. Method injection for action-specific dependencies, constructor injection for shared ones.
- Web and API controllers share the same logic. Only the response format changes. Keep business logic in reusable classes, not in controllers.
- If you need to scroll to read a controller method, it is too long. Extract the logic into a dedicated class.

## References

- [Controllers](https://laravel.com/docs/controllers) — Laravel Documentation
- [Eloquent: API Resources](https://laravel.com/docs/eloquent-resources) — Laravel Documentation
- [Routing: Route Model Binding](https://laravel.com/docs/routing#route-model-binding) — Laravel Documentation
- [Service Container](https://laravel.com/docs/container) — Laravel Documentation
- [Cruddy by Design](https://www.youtube.com/watch?v=MF0jFKvS4SI) — Adam Wathan, Laracon US 2017
- [Cruddy by Design Demo App](https://github.com/adamwathan/laracon2017) — Adam Wathan, GitHub
- [CRUDdy by Design: Summary, Examples, Opinions](https://laraveldaily.com/post/cruddy-by-design-adam-wathan-summary-examples-opinions) — Laravel Daily
- [The Beauty of Single Action Controllers](https://driesvints.com/blog/the-beauty-of-single-action-controllers/) — Dries Vints
- [Single Action Controllers in Laravel](https://dyrynda.com.au/blog/single-action-controllers-in-laravel) — Michael Dyrynda
- [Put Your Laravel Controllers on a Diet](https://matthewdaly.co.uk/blog/2018/02/18/put-your-laravel-controllers-on-a-diet/) — Matthew Daly
- [Laravel Fortify Controllers](https://github.com/laravel/fortify/tree/1.x/src/Http/Controllers) — Laravel, GitHub
