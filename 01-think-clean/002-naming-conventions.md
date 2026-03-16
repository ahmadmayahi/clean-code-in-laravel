Naming is the most underrated skill in software development. A good name eliminates the need for a comment. A bad name creates confusion that ripples through your entire codebase. In Laravel, naming is especially important because the framework uses names to make automatic connections - a `User` model maps to a `users` table, a `PostController` handles `Post` routes, and a `SendWelcomeEmail` job does exactly what its name says.

Taylor Otwell's code is remarkably consistent in its naming. If you read through the [Laravel framework source code](https://github.com/laravel/framework), you will notice that every class, method, and variable follows predictable patterns. This chapter documents those patterns so you can follow them in your own applications.

## Models

[Models](https://laravel.com/docs/eloquent) are always singular nouns in PascalCase. A model represents a single record in a database table, so it makes sense to name it in the singular:

| After | Before | Why |
|---|---|---|
| `User` | `Users` | Plural - a model is one record |
| `Invoice` | `InvoiceModel` | Redundant suffix |
| `OrderItem` | `Order_Item` | Snake_case in a class name |
| `FlightBooking` | `Flightbooking` | Missing PascalCase word boundary |

Laravel automatically maps model names to table names by converting PascalCase to snake_case and pluralizing:

```php
class OrderItem extends Model
{
    // Automatically maps to "order_items" table
    // No $table property needed
}
```

When your model name has multiple words, Laravel handles it correctly. `OrderItem` becomes `order_items`. `FlightBooking` becomes `flight_bookings`. Trust the convention.

## Controllers

[Controllers](https://laravel.com/docs/controllers) follow the pattern `{SingularResource}Controller`. They are always singular because they handle operations on a type of resource, not on multiple resources. We cover controller structure in detail in the [Thin Controllers](/books/clean-code-in-laravel/thin-controllers) chapter.

| After | Before | Why |
|---|---|---|
| `UserController` | `UsersController` | Plural |
| `InvoiceController` | `InvoiceCtrl` | Abbreviated |
| `OrderItemController` | `OrderItemsController` | Plural |

For controllers that do not map to a single resource, use a descriptive name that explains what the controller does:

```php
class DashboardController extends Controller { }
class SettingsController extends Controller { }
class SearchController extends Controller { }
```

For [invokable controllers](https://laravel.com/docs/controllers#single-action-controllers) - controllers with a single `__invoke` method - name them after the action they perform. We explain when and why to use invokable controllers in the [Thin Controllers](/books/clean-code-in-laravel/thin-controllers) chapter.

```php
class ExportOrdersController extends Controller
{
    public function __invoke(Request $request): Response
    {
        // Single action: export orders
    }
}
```

## Methods

Methods in Laravel always use camelCase and typically start with a verb that describes what they do:

```php
// After: verb-first, descriptive
public function calculateTotal(): Money { }
public function sendNotification(): void { }
public function findByEmail(string $email): ?User { }
public function markAsPaid(): void { }
public function isActive(): bool { }
public function hasSubscription(): bool { }

// Before: vague, noun-first, or unclear
public function total(): Money { }        // Is this getting or calculating?
public function notification(): void { }   // Is this sending or creating?
public function email(string $email) { }   // What about the email?
```

Notice the naming patterns for boolean methods: `is` prefix for state checks (`isActive`, `isPaid`, `isAdmin`) and `has` prefix for relationship/ownership checks (`hasSubscription`, `hasPermission`, `hasVerifiedEmail`).

## Variables

Variable naming is arguably the most important naming decision you make, because variables are everywhere. A method gets named once. A variable gets read dozens of times - in conditions, loops, function arguments, and return values. If the name is unclear, every line that uses it becomes harder to understand.

The rule is simple: a variable name should tell you what it holds, not just that it exists. Use camelCase and be specific.

### Say What It Is

The biggest mistake is using short, vague names that force the reader to look elsewhere to understand what the variable contains:

```php
// Before: what is $u? What is $data? What is $inv?
$u = User::where('active', true)->get();
$data = $orders->sum('total');
$inv = Invoice::where('status', 'pending')->first();
$temp = $user->created_at->diffInDays(now());

// After: the name tells you exactly what it holds
$activeUsers = User::where('active', true)->get();
$monthlyRevenue = $orders->sum('total');
$pendingInvoice = Invoice::where('status', 'pending')->first();
$daysSinceRegistration = $user->created_at->diffInDays(now());
```

When you read `$monthlyRevenue`, you know what it is. When you read `$data`, you have to trace back to where it was assigned. That tracing adds up across an entire codebase.

### Avoid Abbreviations

It is tempting to shorten names to save keystrokes, but abbreviated names cost more time in reading than they save in typing. Your editor has autocomplete - use it:

| Before | After | Why |
|---|---|---|
| `$usr` | `$user` | Saves two characters, loses clarity |
| `$addr` | `$shippingAddress` | Which address? Billing? Shipping? |
| `$qty` | `$quantity` | Not everyone reads `qty` as "quantity" |
| `$calc` | `$calculatedDiscount` | Calculated what? |
| `$res` | `$response` | `$res` could mean result, resource, or response |
| `$e` | `$exception` | Single letters hide meaning |

The only place single-letter variables are acceptable is in short loops where the scope is tiny and the meaning is obvious:

```php
for ($i = 0; $i < $retryLimit; $i++) {
    // $i is fine here - small scope, obvious meaning
}
```

### Singular vs. Plural

Use plural names for collections and singular names for single items. This small habit tells the reader whether they are looking at one thing or many things without checking the type:

```php
$users = User::all();              // Collection - plural
$user = User::findOrFail($id);    // Single model - singular

foreach ($users as $user) {        // Iterating: plural → singular
    $user->notify(new WelcomeNotification());
}
```

This also protects you from mistakes. If you see `$user->sum('balance')`, something is wrong - a single model does not have a `sum` method. The plural name `$users->sum('balance')` immediately makes sense.

### Booleans Should Read Like Questions

A boolean variable should read naturally in an `if` statement. Prefix it with `is`, `has`, `can`, `should`, or `was`:

```php
// After: reads like English
$isActive = $user->active;
$hasSubscription = $user->subscription !== null;
$canEditPost = $user->id === $post->user_id;
$shouldSendReminder = $invoice->due_at->isToday();

// Before: what does "true" or "false" mean here?
$active = $user->active;           // Is this a boolean or the active record?
$subscription = true;              // Is this a boolean or a Subscription object?
$edit = $user->id === $post->user_id;  // Edit what?
```

When you read `if ($isActive)`, it flows like a sentence: "if is active." When you read `if ($active)`, it could mean anything.

### Avoid Generic Names

Names like `$data`, `$result`, `$info`, `$temp`, and `$value` tell you nothing. They are placeholders that never got replaced with a real name:

```php
// Before: generic names that could mean anything
$data = $request->validated();
$result = $paymentGateway->charge($amount);
$info = $user->profile;
$items = $request->input('products');

// After: specific names that describe the content
$validatedInput = $request->validated();
$chargeResult = $paymentGateway->charge($amount);
$userProfile = $user->profile;
$selectedProducts = $request->input('products');
```

Sometimes `$result` seems fine because the context is obvious. But code gets moved, refactored, and read out of context. A specific name survives those changes; a generic one does not.

### Match the Domain Language

Use the same words your team and your business use. If the business calls them "invoices," do not call them "bills" in code. If the product team says "subscription," do not use "plan":

```php
// If the business says "coupon"
$appliedCoupon = Coupon::where('code', $code)->first();

// Do not call it a "discount code" or "promo" in code
$promo = Coupon::where('code', $code)->first(); // Confusing
```

This idea comes from [Domain-Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) - using a shared language between developers and the business. When everyone uses the same words, there is less room for misunderstanding.

## Database

### Tables

[Database](https://laravel.com/docs/database) tables are always plural and use snake_case. This is the convention that [Eloquent](https://laravel.com/docs/eloquent#eloquent-model-conventions) relies on for automatic model-to-table mapping:

| Model | Table | Pivot Table |
|---|---|---|
| `User` | `users` | - |
| `Post` | `posts` | - |
| `OrderItem` | `order_items` | - |
| `User` + `Role` | - | `role_user` (alphabetical) |

[Pivot tables](https://laravel.com/docs/eloquent-relationships#many-to-many) combine the two model names in singular snake_case, sorted alphabetically, joined by an underscore. So `User` and `Role` become `role_user`, not `user_role`.

### Columns

Columns use snake_case and should describe what they store. See [migrations](https://laravel.com/docs/migrations#creating-columns) for all available column types:

```php
Schema::create('orders', function (Blueprint $table): void {
    $table->id();
    $table->foreignId('user_id')->constrained();     // Foreign key: model_id
    $table->string('shipping_address');                // Descriptive
    $table->decimal('total_amount', 10, 2);           // Clear unit
    $table->timestamp('paid_at')->nullable();          // Timestamp: verb_at
    $table->boolean('is_refundable')->default(true);   // Boolean: is_ prefix
    $table->timestamps();                              // created_at, updated_at
});
```

Notice the patterns: foreign keys use `{model}_id`, timestamps use `{verb}_at` (like `paid_at`, `shipped_at`, `verified_at`), and booleans use `is_` or `has_` prefixes.

## Routes

[Routes](https://laravel.com/docs/routing) use plural nouns in kebab-case for the URI, and follow [RESTful](https://laravel.com/docs/controllers#resource-controllers) conventions:

```php
// After: RESTful, plural, kebab-case
Route::get('/users', [UserController::class, 'index']);
Route::get('/users/{user}', [UserController::class, 'show']);
Route::get('/order-items', [OrderItemController::class, 'index']);
Route::post('/order-items', [OrderItemController::class, 'store']);

// Before: singular, camelCase, or verb-based
Route::get('/user', [UserController::class, 'index']);       // Singular
Route::get('/orderItems', [OrderItemController::class, 'index']); // camelCase
Route::get('/getUsers', [UserController::class, 'index']);   // Verb in URL
```

### Dashes vs. Underscores in URLs

When a route has multiple words, use dashes (kebab-case), not underscores. This is a web-wide convention - [Google recommends dashes](https://developers.google.com/search/docs/crawling-indexing/url-structure) over underscores in URLs, and it is what most REST APIs use:

```php
// After: dashes in the URL
Route::get('/order-items', [OrderItemController::class, 'index']);
Route::get('/payment-methods', [PaymentMethodController::class, 'index']);
Route::post('/forgot-password', ForgotPasswordController::class);

// Before: underscores or camelCase in the URL
Route::get('/order_items', [OrderItemController::class, 'index']);
Route::get('/paymentMethods', [PaymentMethodController::class, 'index']);
```

There is one exception: route parameters must use camelCase or snake_case because they map to PHP variables. Dashes are not allowed in PHP variable names, so `{order-item}` would not work:

```php
// Route parameters use camelCase — dashes do not work here
Route::get('/users/{user}', [UserController::class, 'show']);
Route::get('/order-items/{orderItem}', [OrderItemController::class, 'show']);
```

### Named Routes

[Named routes](https://laravel.com/docs/routing#named-routes) use dot notation with the resource name:

```php
Route::resource('users', UserController::class);
// Generates: users.index, users.create, users.store, users.show, etc.

// Custom named routes follow the same pattern
Route::get('/dashboard', DashboardController::class)->name('dashboard');
Route::get('/users/{user}/orders', [UserOrderController::class, 'index'])
    ->name('users.orders.index');
```

When a route name has multiple words, there is no single official convention. Some teams use camelCase, others use kebab-case. [Spatie's guidelines](https://spatie.be/guidelines/laravel#content-routing) recommend camelCase:

```php
// camelCase (Spatie convention)
Route::get('/account/billing-history', [BillingController::class, 'history'])
    ->name('account.billingHistory');

// kebab-case (matches what Laravel generates for resource routes)
Route::get('/account/billing-history', [BillingController::class, 'history'])
    ->name('account.billing-history');
```

Both work fine. What matters is that your team picks one and sticks with it. Avoid snake_case — `billing_history` — since neither the framework nor the community uses it for route names.

Dots separate resources. The style you choose separates words within a segment. If you stay consistent, you can always guess the route name without running `php artisan route:list`.

## Views

[View](https://laravel.com/docs/views) files use kebab-case and are organized in directories matching their resource:

```
resources/views/
├── users/
│   ├── index.blade.php
│   ├── show.blade.php
│   ├── create.blade.php
│   └── edit.blade.php
├── order-items/
│   ├── index.blade.php
│   └── show.blade.php
└── components/
    ├── alert.blade.php
    └── user-card.blade.php
```

Reference them with dot notation:

```php
return view('users.index', compact('users'));
return view('order-items.show', compact('orderItem'));
```

## Config Files

[Configuration](https://laravel.com/docs/configuration) files and their keys use snake_case. The filename itself is also snake_case:

```php
// config/payment.php
return [
    'stripe_key' => env('STRIPE_KEY'),
    'default_currency' => 'usd',
    'tax_rate' => 0.21,
];

// Accessing config values with dot notation
$key = config('payment.stripe_key');
$currency = config('payment.default_currency');
```

Keep config keys flat and descriptive. If you find yourself nesting deeply, it is usually a sign that the config file is doing too much and should be split into separate files.

## Language Files

Laravel supports two approaches for [localization](https://laravel.com/docs/localization): PHP files with short keys, and JSON files with full translation strings as keys.

### PHP Language Files

PHP language files live under `lang/{locale}/` and use snake_case for both the filename and the keys:

```
lang/
├── en/
│   ├── messages.php
│   ├── validation.php
│   └── auth.php
└── fr/
    ├── messages.php
    ├── validation.php
    └── auth.php
```

```php
// lang/en/messages.php
return [
    'welcome' => 'Welcome to our application!',
    'order_placed' => 'Your order has been placed.',
    'greeting' => 'Hello, :name!',
];

// Accessing translations
echo __('messages.welcome');
echo __('messages.greeting', ['name' => 'Ahmad']);
```

### JSON Language Files

For applications with many translatable strings, JSON files are simpler. Each language gets a single file named after its locale, and the keys are the full English strings:

```json
// lang/es.json
{
    "Welcome to our application!": "¡Bienvenido a nuestra aplicación!",
    "Your order has been placed.": "Tu pedido ha sido realizado."
}
```

The advantage of JSON files is that you do not need to invent short keys — the English text is the key. The trade-off is that they can get large in applications with hundreds of strings.

For territory-specific languages, use the [ISO 15897](https://en.wikipedia.org/wiki/ISO/IEC_15897) format: `en_GB` for British English, `pt_BR` for Brazilian Portuguese — not `en-gb` or `pt-br`.

## Form Requests

[Form requests](https://laravel.com/docs/validation#form-request-validation) describe the action they validate:

| After | Before |
|---|---|
| `StoreUserRequest` | `UserRequest` |
| `UpdateOrderRequest` | `OrderValidation` |
| `CreateInvoiceRequest` | `InvoiceFormRequest` |

The `Store` and `Update` prefixes match the [resource controller](/books/clean-code-in-laravel/thin-controllers) methods they serve, making the connection obvious:

```php
class UserController extends Controller
{
    public function store(StoreUserRequest $request): RedirectResponse { }
    public function update(UpdateUserRequest $request, User $user): RedirectResponse { }
}
```

## Events, Listeners, Jobs, and Notifications

These classes follow a consistent pattern: they describe what happened (events), what to do about it (listeners/jobs), or what to tell someone (notifications):

| Type | Naming Pattern | Example |
|---|---|---|
| [Event](https://laravel.com/docs/events) | Past tense verb | `OrderPlaced`, `UserRegistered`, `PaymentFailed` |
| [Listener](https://laravel.com/docs/events#defining-listeners) | Action to take | `SendOrderConfirmation`, `CreateUserProfile` |
| [Job](https://laravel.com/docs/queues) | Action to perform | `ProcessPayment`, `GenerateInvoicePdf` |
| [Notification](https://laravel.com/docs/notifications) | What is being communicated | `OrderShippedNotification`, `WelcomeNotification` |
| [Mail](https://laravel.com/docs/mail) | What is being sent | `OrderConfirmationMail`, `PasswordResetMail` |

Events use past tense because they describe something that already happened. Listeners, jobs, and notifications describe what will happen in response.

## Enums

[Enums](https://www.php.net/manual/en/language.enumerations.php) use singular PascalCase, and their cases use PascalCase as well. Laravel has [built-in support](https://laravel.com/docs/eloquent-mutators#attribute-casting) for casting model attributes to enums:

```php
enum OrderStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';
}
```

The enum name describes what it represents (`OrderStatus`, `PaymentMethod`, `UserRole`), and each case reads naturally: `OrderStatus::Shipped`, `PaymentMethod::CreditCard`.

## Middleware

[Middleware](https://laravel.com/docs/middleware) names describe what they check or enforce. Use PascalCase and keep the name short:

| After | Before | Why |
|---|---|---|
| `EnsureEmailIsVerified` | `CheckEmail` | Too vague — check what? |
| `HandleLocale` | `LocaleMiddleware` | Redundant suffix |
| `Authenticate` | `Auth` | Abbreviated |

Laravel's own middleware follow this pattern: `EnsureFrontendRequestsAreStateful`, `TrustProxies`, `HandleCors`. The name should tell you what the middleware does without opening the file.

## Policies

[Policies](https://laravel.com/docs/authorization#creating-policies) map directly to models. The naming is simple: `{Model}Policy`:

| Model | Policy |
|---|---|
| `User` | `UserPolicy` |
| `Invoice` | `InvoicePolicy` |
| `OrderItem` | `OrderItemPolicy` |

Policy methods match the action they authorize: `view`, `create`, `update`, `delete`, `restore`, `forceDelete`. These names come from Laravel's default resource methods, and the framework auto-discovers them when you follow this convention.

## Traits

[Traits](https://www.php.net/manual/en/language.oop5.traits.php) use PascalCase and describe the capability they provide. Laravel's built-in traits follow a clear pattern — they start with a verb or `Has` prefix that reads naturally when you see them in a `use` statement:

```php
use HasFactory;
use Notifiable;
use SoftDeletes;
use HasApiTokens;
```

Follow the same pattern in your own traits. The name should describe what the trait gives the class, not what the class is:

| After | Before | Why |
|---|---|---|
| `HasSubscription` | `SubscriptionTrait` | Redundant suffix |
| `Searchable` | `SearchTrait` | Redundant suffix |
| `TracksActivity` | `Activity` | Sounds like a model, not a behavior |
| `HandlesPayments` | `PaymentFunctions` | Vague and unusual |

When you read `class User extends Model` followed by `use HasSubscription, Searchable`, it reads like a description: "User has subscription, is searchable."

## Scopes

[Eloquent scopes](https://laravel.com/docs/eloquent#query-scopes) define reusable query constraints on a model. In Laravel 12, scopes use the `#[Scope]` attribute on a `protected` method:

```php
use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;

class Order extends Model
{
    #[Scope]
    protected function paid(Builder $query): void
    {
        $query->whereNotNull('paid_at');
    }

    #[Scope]
    protected function recent(Builder $query): void
    {
        $query->where('created_at', '>=', now()->subDays(30));
    }

    #[Scope]
    protected function forUser(Builder $query, User $user): void
    {
        $query->where('user_id', $user->id);
    }
}

// Reads naturally when chained
$orders = Order::paid()->recent()->forUser($user)->get();
```

The method name is what you call when chaining -`paid()`, `recent()`, `forUser()`. Pick names that are adjectives or past-tense verbs so they read well in a chain. Notice how `Order::paid()->recent()->forUser($user)` reads almost like English. That is the goal.

Avoid scope names that start with verbs like `get` or `find` — those suggest a method that returns a result, not a scope that filters a query:

| After | Before | Why |
|---|---|---|
| `active` | `getActive` | Scopes do not "get" — they filter |
| `published` | `findPublished` | Same issue — sounds like it returns a result |
| `forUser` | `byUser` | Less clear, but acceptable |

## Blade Components

[Blade components](https://laravel.com/docs/blade#components) use kebab-case in their tag names and PascalCase for the class. Laravel handles the conversion automatically:

```php
// Class: app/View/Components/UserCard.php
class UserCard extends Component { }

// Usage in Blade
<x-user-card :user="$user" />
```

For nested components, the directory structure maps to dot notation in the tag:

```
app/View/Components/
├── Form/
│   ├── Input.php          → <x-form.input />
│   └── Select.php         → <x-form.select />
└── Navigation/
    └── Breadcrumb.php     → <x-navigation.breadcrumb />
```

Anonymous components — Blade files without a class — follow the same kebab-case pattern:

```
resources/views/components/
├── alert.blade.php        → <x-alert />
├── user-card.blade.php    → <x-user-card />
└── form/
    └── input.blade.php    → <x-form.input />
```

## Constants

[Class constants](https://www.php.net/manual/en/language.oop5.constants.php) and global constants use `SCREAMING_SNAKE_CASE`:

```php
class Invoice
{
    public const MAX_LINE_ITEMS = 50;
    public const DEFAULT_CURRENCY = 'usd';
    public const TAX_RATE = 0.21;
}
```

This is a PHP-wide convention, not specific to Laravel, but it is worth mentioning because mixing constant naming styles in a codebase is a common mistake.

If a class exists only to hold a group of related constants, consider using an [enum](https://www.php.net/manual/en/language.enumerations.php) instead. Enums give you type safety, autocompletion, and a single place to manage allowed values — which plain constants do not:

```php
// Instead of a class full of constants
class Currency
{
    public const USD = 'usd';
    public const EUR = 'eur';
    public const GBP = 'gbp';
}

// Use an enum
enum Currency: string
{
    case Usd = 'usd';
    case Eur = 'eur';
    case Gbp = 'gbp';
}
```

## The Complete Naming Reference

| What | Convention | Example |
|---|---|---|
| Model | Singular, PascalCase | `OrderItem` |
| Controller | Singular + Controller | `OrderItemController` |
| Migration | snake_case, descriptive | `create_order_items_table` |
| Seeder | Singular + Seeder | `OrderItemSeeder` |
| Factory | Singular + Factory | `OrderItemFactory` |
| Table | Plural, snake_case | `order_items` |
| Pivot table | Alphabetical, singular | `item_order` |
| Column | snake_case | `total_amount` |
| Foreign key | Singular model + _id | `order_id` |
| Route | Plural, kebab-case | `/order-items` |
| Named route | Dot notation | `order-items.index` |
| View | kebab-case directory | `order-items/index.blade.php` |
| Config | snake_case | `config('payment.stripe_key')` |
| Method | camelCase, verb-first | `calculateTotal()` |
| Variable | camelCase | `$orderItem` |
| Property | camelCase | `$this->shippingAddress` |
| Constant | SCREAMING_SNAKE_CASE | `MAX_LINE_ITEMS` |
| Form Request | Verb + Resource + Request | `StoreOrderItemRequest` |
| Event | Past tense | `OrderItemAdded` |
| Listener | Action phrase | `UpdateInventoryCount` |
| Job | Action phrase | `ProcessOrderPayment` |
| Notification | Descriptive + Notification | `OrderShippedNotification` |
| Enum | Singular, PascalCase | `OrderStatus` |
| Middleware | Descriptive, PascalCase | `EnsureEmailIsVerified` |
| Policy | Model + Policy | `OrderItemPolicy` |
| Trait | PascalCase, describes capability | `HasSubscription` |
| Scope | `#[Scope]` + adjective/verb | `paid()`, `active()` |
| Blade component | kebab-case tag, PascalCase class | `<x-user-card>` / `UserCard` |

Follow these conventions consistently, and any Laravel developer can navigate your codebase without extra documentation.

## Summary

- Models are singular PascalCase (`OrderItem`). Laravel maps them to plural snake_case tables (`order_items`) automatically.
- Controllers are singular with a `Controller` suffix (`OrderItemController`). Invokable controllers are named after the action they perform.
- Methods use camelCase and start with a verb (`calculateTotal`, `sendNotification`). Boolean methods use `is`/`has`/`can` prefixes.
- Variables should say what they hold (`$activeUsers`, not `$data`). Use plural for collections, singular for single items, and boolean prefixes for true/false values.
- Database tables are plural snake_case. Columns are snake_case. Foreign keys use `{model}_id`. Timestamps use `{verb}_at`. Booleans use `is_`/`has_` prefixes.
- Routes use plural kebab-case URLs (`/order-items`). Named routes use dot notation (`users.orders.index`).
- Views use kebab-case directories and filenames (`order-items/index.blade.php`).
- Events use past tense (`OrderPlaced`). Listeners, jobs, and notifications describe the action (`SendOrderConfirmation`).
- Traits describe a capability (`HasSubscription`, `Searchable`). Scopes use the `#[Scope]` attribute with adjectives or past-tense verbs that chain naturally (`Order::paid()->recent()`).
- Blade components use kebab-case tags (`<x-user-card>`) and PascalCase classes (`UserCard`).
- When in doubt, check how Laravel names the same thing in its own source code — then do that.
