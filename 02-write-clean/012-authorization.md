A user is logged in. That does not mean they can do whatever they want.

Authentication answers "who are you?" Authorization answers "what are you allowed to do?" Laravel provides two built-in mechanisms for authorization — [Gates](https://laravel.com/docs/authorization#gates) and [Policies](https://laravel.com/docs/authorization#creating-policies) — and Spatie's [`laravel-permission`](https://spatie.be/docs/laravel-permission) package adds a role-and-permission system on top. Together, they give you a clean, testable way to control access at every layer of your application.

## The Problem With Inline Authorization

Authorization logic often starts in controllers as quick `if` checks:

```php
public function update(Request $request, Post $post): RedirectResponse
{
    if ($request->user()->id !== $post->user_id) {
        abort(403);
    }

    if ($request->user()->role !== 'editor' && $request->user()->role !== 'admin') {
        abort(403);
    }

    // Update the post...
}
```

This works for a single endpoint. But authorization rules multiply fast. The same ownership check appears in `update`, `destroy`, and `show`. The role check appears in a dozen controllers. When the rules change — editors can no longer delete posts, or a new "moderator" role is introduced — you are hunting through controllers to find every `if` statement.

Laravel's authorization features give this logic a proper home.

## Gates

A [Gate](https://laravel.com/docs/authorization#gates) is a closure that determines whether a user can perform a given action. Define Gates in the `boot` method of your `AppServiceProvider`:

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::define('update-post', function (User $user, Post $post): bool {
        return $user->id === $post->user_id;
    });
}
```

Then check the Gate anywhere in your application:

```php
// In a controller
if (Gate::denies('update-post', $post)) {
    abort(403);
}

// Or let Laravel throw the exception for you
Gate::authorize('update-post', $post);
```

`Gate::authorize()` throws an `AuthorizationException` if the check fails, which Laravel automatically converts to a 403 response. This is cleaner than manual `abort(403)` calls.

### Gate Responses

Gates can return more than booleans. Return a `Response` to include an error message:

```php
use Illuminate\Auth\Access\Response;

Gate::define('edit-settings', function (User $user): Response {
    return $user->isAdmin()
        ? Response::allow()
        : Response::deny('You must be an administrator.');
});
```

Use `Gate::inspect()` to access the full response:

```php
$response = Gate::inspect('edit-settings');

if ($response->denied()) {
    echo $response->message(); // "You must be an administrator."
}
```

### Intercepting Gate Checks

Use `Gate::before()` to grant all abilities to specific users — typically administrators:

```php
Gate::before(function (User $user, string $ability): ?bool {
    if ($user->isAdministrator()) {
        return true;
    }

    return null; // Fall through to the specific gate
});
```

Returning `null` from `before` lets the specific Gate or Policy handle the check. Returning `true` or `false` short-circuits everything.

## Policies

A [Policy](https://laravel.com/docs/authorization#creating-policies) is a class that groups authorization logic around a specific model. If Gates are like closures, Policies are like [controllers](/books/clean-code-in-laravel/controllers) — they organize related checks into a single, focused class.

### Creating a Policy

```bash
php artisan make:policy PostPolicy --model=Post
```

This generates a class in `app/Policies/` with methods for `viewAny`, `view`, `create`, `update`, `delete`, `restore`, and `forceDelete` — covering the authorization checks you need for [resource controller](/books/clean-code-in-laravel/controllers) operations.

```php
namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    public function create(User $user): bool
    {
        return $user->hasVerifiedEmail();
    }
}
```

### Policy Discovery

Laravel automatically discovers Policies by convention: `Post` model maps to `PostPolicy`, `Order` maps to `OrderPolicy`. Place your Policies in `app/Policies/` and Laravel finds them without any registration.

If you need explicit registration, use the `UsePolicy` attribute on the model:

```php
use App\Policies\OrderPolicy;
use Illuminate\Database\Eloquent\Attributes\UsePolicy;

#[UsePolicy(OrderPolicy::class)]
class Order extends Model
{
    // ...
}
```

### Policy Responses

Like Gates, Policy methods can return `Response` objects for detailed feedback:

```php
use Illuminate\Auth\Access\Response;

public function update(User $user, Post $post): Response
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::deny('You do not own this post.');
}
```

To hide the existence of a resource entirely, return a 404 instead of 403:

```php
public function view(User $user, Post $post): Response
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::denyAsNotFound();
}
```

### Policy Filters

The `before` method on a Policy works exactly like `Gate::before()` — it runs before any other method and can short-circuit the check:

```php
public function before(User $user, string $ability): ?bool
{
    if ($user->isAdministrator()) {
        return true;
    }

    return null;
}
```

## Gates vs. Policies: When to Use Which

Gates and Policies are not competing tools — they solve different problems. The confusion is understandable because both answer "can this user do this thing?" But they differ in scope, structure, and where they shine.

A Gate is a standalone closure. It lives in your `AppServiceProvider` and checks whether a user can perform a general action — one that is not necessarily tied to a specific Eloquent model. "Can this user access the admin dashboard?" "Can this user export reports?" "Can this user send invitations?" These are Gate territory.

A Policy is a class organized around a specific model. It groups all authorization logic for that model — viewing, creating, updating, deleting — into one place. "Can this user update *this* post?" "Can this user delete *this* order?" These are Policy territory.

A **Gate** is a closure defined in `AppServiceProvider`. It is action-centric — standalone operations like dashboard access, feature flags, or admin overrides. Gates must be defined manually and work best for a handful of checks.

A **Policy** is a dedicated class in `app/Policies/`. It is model-centric — CRUD on a specific model like ownership checks or resource access. Policies are auto-discovered by naming convention and scale to unlimited models, one class each.

### Use Policies as Your Default

For any authorization tied to an Eloquent model, use a Policy. Policies are auto-discovered, testable, and scale cleanly — one class per model, each method maps to a controller action. They integrate deeply with Laravel's conventions: `Gate::authorize()`, the `can` middleware, `@can` Blade directives, and Form Request `authorize()` methods all resolve Policies automatically.

### Use Gates for Everything Else

Gates are best for checks that do not involve a specific model instance:

```php
// Gates work well when no model is involved
Gate::define('access-admin', fn (User $user): bool => $user->isAdmin());
Gate::define('view-analytics', fn (User $user): bool => $user->can('view analytics'));
Gate::define('send-invitations', fn (User $user): bool => $user->team->hasInvitesRemaining());
```

### Do Not Use Gates for Everything

A common mistake is using Gates as the sole authorization mechanism. One file with dozens of Gate definitions becomes unmanageable fast. If you find yourself defining Gates like `update-post`, `delete-post`, `view-post`, and `create-post`, those belong in a `PostPolicy` — not in four separate closures in your service provider.

### Use Both Together

Most applications use both. Gates handle the handful of non-model checks. Policies handle the rest. `Gate::before()` can provide a global admin override that applies to all Gates *and* all Policies:

```php
// AppServiceProvider — covers Gates AND Policies
Gate::before(function (User $user): ?bool {
    if ($user->isSuperAdmin()) {
        return true;
    }

    return null;
});

// Gate for non-model action
Gate::define('access-admin', fn (User $user): bool => $user->isAdmin());

// Policies handle model-specific checks automatically
// PostPolicy, OrderPolicy, UserPolicy — auto-discovered
```

## Authorizing Actions

Laravel provides multiple ways to check Policies. Use whichever fits the context.

### In Controllers

The `Gate::authorize()` method is the cleanest approach — it throws an exception if the check fails, so your controller stays focused on the happy path:

```php
class PostController extends Controller
{
    public function update(UpdatePostRequest $request, Post $post): RedirectResponse
    {
        Gate::authorize('update', $post);

        $post->update($request->validated());

        return redirect()->route('posts.show', $post);
    }
}
```

For `create` actions that do not have a model instance, pass the class name:

```php
public function store(StorePostRequest $request): RedirectResponse
{
    Gate::authorize('create', Post::class);

    // Create the post...
}
```

You can also use the `can` and `cannot` methods on the User model:

```php
if ($request->user()->cannot('update', $post)) {
    abort(403);
}
```

### Via Middleware

Authorize before the request reaches your controller using the `can` middleware:

```php
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->can('update', 'post');

Route::post('/posts', [PostController::class, 'store'])
    ->can('create', Post::class);
```

### In Blade Templates

Use `@can` and `@cannot` directives to conditionally render UI elements:

```blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

@can('create', App\Models\Post::class)
    <a href="{{ route('posts.create') }}">New Post</a>
@endcan

@canany(['update', 'delete'], $post)
    <div class="actions">
        <!-- Show action buttons -->
    </div>
@endcanany
```

### In Form Requests

[Form Requests](/books/clean-code-in-laravel/form-requests-and-validation) have a built-in `authorize()` method for simple checks:

```php
class UpdatePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('post'));
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'body' => ['required', 'string'],
        ];
    }
}
```

### With Inertia.js

Share authorization data with your frontend through the `HandleInertiaRequests` middleware:

```php
public function share(Request $request): array
{
    return [
        ...parent::share($request),
        'auth' => [
            'user' => $request->user(),
            'can' => [
                'create_post' => $request->user()?->can('create', Post::class),
                'manage_users' => $request->user()?->can('viewAny', User::class),
            ],
        ],
    ];
}
```

Now your Vue or React components can check `$page.props.auth.can.create_post` to show or hide UI elements. Always enforce authorization on the server — frontend checks are for UX, not security.

## Roles and Permissions

Gates and Policies let you define authorization rules in code. But what happens when the rules need to change without a deployment? A client says "editors should now be able to publish posts." With hardcoded checks, that is a code change, a PR, and a deploy. With a role-and-permission system, it is a database update.

A permission is a single, granular ability — `create posts`, `edit posts`, `delete posts`, `publish posts`. It describes one thing a user can do. Permissions are the atoms of your authorization system.

A role is a named group of permissions — `writer`, `editor`, `admin`. It is a convenience for assigning multiple permissions at once. A writer can `create posts` and `edit posts`. An editor can do everything a writer can, plus `delete posts` and `publish posts`. An admin can do everything.

The key insight: your application should check permissions, not roles. Roles are an organizational shortcut for humans. Permissions are what the code cares about. This means you can change what an "editor" can do by updating the role's permissions — without touching a single line of application code.

Laravel does not ship with a role-and-permission system, but [`spatie/laravel-permission`](https://spatie.be/docs/laravel-permission) is the de facto standard. It stores roles and permissions in the database, integrates with Laravel's Gate system, and provides middleware, Blade directives, and Eloquent scopes out of the box.

### Installation

```bash
composer require spatie/laravel-permission
```

Publish the migration and config:

```bash
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

Add the `HasRoles` trait to your User model:

```php
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles;
}
```

### Creating Roles and Permissions

Roles and permissions are Eloquent models. Create them in a seeder so they are consistent across environments:

```php
use Spatie\Permission\Models\Role;
use Spatie\Permission\Models\Permission;

class RoleAndPermissionSeeder extends Seeder
{
    public function run(): void
    {
        // Reset cached roles and permissions
        app()[\Spatie\Permission\PermissionRegistrar::class]->forgetCachedPermissions();

        // Create permissions
        Permission::create(['name' => 'create posts']);
        Permission::create(['name' => 'edit posts']);
        Permission::create(['name' => 'delete posts']);
        Permission::create(['name' => 'publish posts']);
        Permission::create(['name' => 'manage users']);

        // Create roles and assign permissions
        Role::create(['name' => 'writer'])
            ->givePermissionTo(['create posts', 'edit posts']);

        Role::create(['name' => 'editor'])
            ->givePermissionTo(['create posts', 'edit posts', 'delete posts', 'publish posts']);

        Role::create(['name' => 'admin'])
            ->givePermissionTo(Permission::all());
    }
}
```

### Assigning Roles to Users

```php
$user->assignRole('writer');
$user->assignRole('writer', 'editor');  // Multiple roles
$user->removeRole('writer');
$user->syncRoles(['editor']);           // Replace all roles
```

### Checking Permissions, Not Roles

This is the most important rule when working with Spatie Permission: always check permissions, not roles. Roles are an organizational tool for grouping permissions. Your application logic should not care whether someone is an "editor" or an "admin" — it should care whether they can `edit posts`.

```php
// Before: checking roles in business logic
if ($user->hasRole('admin')) {
    // Delete the post...
}

// After: checking permissions
if ($user->can('delete posts')) {
    // Delete the post...
}
```

Why does this matter? Because roles change. Today "editors" cannot delete posts. Tomorrow the business decides they can. If you checked `hasRole('admin')`, you need to find and update every check. If you checked `can('delete posts')`, you add the permission to the editor role in one place and every check updates automatically.

The `can()` method works with Laravel's Gate system, so Spatie permissions integrate seamlessly with `@can` Blade directives, `Gate::authorize()`, Form Request `authorize()` methods, and the `can` middleware.

### Spatie Middleware

Spatie provides three middleware classes for route-level authorization. Register them in `bootstrap/app.php`:

```php
use Illuminate\Foundation\Configuration\Middleware;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'role' => \Spatie\Permission\Middleware\RoleMiddleware::class,
        'permission' => \Spatie\Permission\Middleware\PermissionMiddleware::class,
        'role_or_permission' => \Spatie\Permission\Middleware\RoleOrPermissionMiddleware::class,
    ]);
})
```

Use them on routes:

```php
// Permission-based (preferred)
Route::middleware('permission:publish posts')->group(function (): void {
    Route::post('/posts/{post}/publish', [PublishPostController::class, '__invoke']);
});

// Multiple permissions with OR logic
Route::middleware('permission:edit posts|delete posts')->group(function (): void {
    // User needs at least one of these permissions
});

// Role-based (use sparingly)
Route::middleware('role:admin')->group(function (): void {
    Route::get('/admin/dashboard', [AdminDashboardController::class, 'index']);
});
```

Prefer the `permission` middleware over `role` in most cases. The `role` middleware is useful for top-level route groups like an admin panel where access is inherently role-based.

### Querying Users by Role or Permission

Spatie provides query scopes for finding users with specific roles or permissions:

```php
// All users with the 'editor' role
$editors = User::role('editor')->get();

// All users who can publish posts (via any role)
$publishers = User::permission('publish posts')->get();

// Users without a specific role
$nonAdmins = User::withoutRole('admin')->get();
```

### Combining Policies with Spatie Permissions

Policies and Spatie permissions are not competing patterns — they complement each other. Policies define the authorization *logic*. Spatie permissions provide the *data* that the logic checks against.

```php
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        // Ownership check OR permission check
        return $user->id === $post->user_id
            || $user->can('edit posts');
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id
            || $user->can('delete posts');
    }

    public function publish(User $user, Post $post): bool
    {
        return $user->can('publish posts');
    }
}
```

The Policy handles model-specific logic (ownership checks, status checks). Spatie permissions handle role-based access. The two work together through Laravel's Gate system.

## API Authentication With Sanctum

Authorization answers "what are you allowed to do?" But before you can authorize, you need to authenticate — "who are you?" For web applications, Laravel handles this with session cookies. For APIs, mobile apps, and SPAs, you need something else. That is where [Sanctum](https://laravel.com/docs/sanctum) comes in.

Sanctum provides two distinct authentication mechanisms:

- **API token authentication** — for third-party integrations, mobile apps, and any client that needs a Bearer token
- **SPA authentication** — for first-party single-page applications using cookie-based session auth

You do not need both. Pick whichever fits your use case — or use both if your application serves multiple types of consumers.

### Installation

```bash
php artisan install:api
```

This publishes the Sanctum configuration, creates the `personal_access_tokens` migration, and sets up `routes/api.php`. Run the migration:

```bash
php artisan migrate
```

### API Token Authentication

Add the `HasApiTokens` trait to your User model:

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

Issue tokens using the `createToken` method. The plain-text token is only available at creation time — store it securely on the client:

```php
Route::post('/tokens/create', function (Request $request): JsonResponse {
    $request->validate([
        'name' => ['required', 'string'],
    ]);

    $token = $request->user()->createToken($request->name);

    return response()->json([
        'token' => $token->plainTextToken,
    ]);
});
```

Clients include the token in the `Authorization` header:

```
Authorization: Bearer 1|abc123...
```

### Token Abilities

Tokens can be scoped to specific abilities — similar to OAuth scopes but without the complexity. This is where Sanctum and authorization intersect. A token should only grant the abilities the consumer actually needs:

```php
// A read-only integration token
$token = $user->createToken('reporting-dashboard', [
    'orders:read',
    'products:read',
]);

// A full-access token for a trusted service
$token = $user->createToken('inventory-sync', [
    'products:read',
    'products:write',
    'stock:update',
]);
```

Check abilities in your controllers or Policies using `tokenCan`:

```php
public function update(Request $request, Product $product): JsonResponse
{
    if ($request->user()->tokenCant('products:write')) {
        abort(403, 'Token does not have write access.');
    }

    $product->update($request->validated());

    return new ProductResource($product);
}
```

For first-party SPA requests authenticated via cookies, `tokenCan` always returns `true` — because the user is not using a scoped token, they are the user. Your Policies handle the actual permission checks.

### Combining Token Abilities With Policies

Token abilities and Policies solve different problems. Abilities restrict what the *token* can do. Policies restrict what the *user* can do. Use both:

```php
class ProductPolicy
{
    public function update(User $user, Product $product): bool
    {
        // First: does the user have permission?
        if (! $user->can('edit products')) {
            return false;
        }

        // Second: does the token allow this action?
        return $user->tokenCan('products:write');
    }
}
```

This way, even if a token has `products:write`, the user still needs the `edit products` permission. Defense in depth.

### Protecting Routes

Attach the `sanctum` guard to routes that require authentication:

```php
// routes/api.php
Route::middleware('auth:sanctum')->group(function (): void {
    Route::get('/user', function (Request $request): JsonResponse {
        return response()->json($request->user());
    });

    Route::apiResource('orders', OrderController::class);
});
```

The `sanctum` guard is smart — it checks for a session cookie first (SPA auth), then falls back to a Bearer token (API auth). This means the same routes work for both first-party SPAs and third-party integrations.

### SPA Authentication

For first-party SPAs, Sanctum uses cookie-based session authentication instead of tokens. This gives you CSRF protection and avoids storing tokens in JavaScript — a significant security advantage.

Enable stateful authentication in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->statefulApi();
})
```

Your SPA must first request a CSRF cookie, then authenticate:

```js
// 1. Initialize CSRF protection
await axios.get('/sanctum/csrf-cookie');

// 2. Log in using session-based auth
await axios.post('/login', {
    email: 'user@example.com',
    password: 'password',
});

// 3. Now all subsequent requests are authenticated via cookies
const response = await axios.get('/api/user');
```

No tokens to store, no `Authorization` headers to manage. The browser handles cookies automatically.

> Your SPA and API must share the same top-level domain. They can be on different subdomains (e.g., `app.example.com` and `api.example.com`), but not entirely different domains.

### Mobile Application Authentication

Mobile apps use token-based auth. Create a login endpoint that exchanges credentials for a token:

```php
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

Route::post('/sanctum/token', function (Request $request): string {
    $request->validate([
        'email' => ['required', 'email'],
        'password' => ['required'],
        'device_name' => ['required'],
    ]);

    $user = User::where('email', $request->email)->first();

    if (! $user || ! Hash::check($request->password, $user->password)) {
        throw ValidationException::withMessages([
            'email' => ['The provided credentials are incorrect.'],
        ]);
    }

    return $user->createToken($request->device_name)->plainTextToken;
});
```

### Token Expiration and Revocation

By default, Sanctum tokens never expire. You can set a global expiration in `config/sanctum.php`:

```php
'expiration' => 525600, // Minutes (1 year)
```

Or set per-token expiration:

```php
$token = $user->createToken(
    'short-lived-token',
    ['*'],
    now()->addWeek(),
);
```

Revoke tokens when they are no longer needed:

```php
// Revoke all tokens for the user
$user->tokens()->delete();

// Revoke the current token (e.g., on logout)
$request->user()->currentAccessToken()->delete();

// Revoke a specific token
$user->tokens()->where('id', $tokenId)->delete();
```

Prune expired tokens with a scheduled command:

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

### Sanctum vs. Passport

Sanctum is not a stripped-down Passport. They serve different needs:

**Sanctum** is for first-party SPAs, mobile apps, and simple APIs. It uses API tokens and cookie-based sessions, requires minimal setup (one migration, one trait), and scopes tokens with simple string abilities. Choose it when you control both the client and the API.

**Passport** is for third-party OAuth2 integrations. It provides full OAuth2 with authorization codes and client credentials, requires significant setup (encryption keys, OAuth clients, grant types), and uses full OAuth2 scopes. Choose it when external developers need to authorize against your API.

If you are building an API consumed by your own frontend or mobile app, use Sanctum. If you are building an OAuth2 provider (like GitHub or Google), use Passport.

## Testing Authorization

Because Policies are plain PHP classes, they are straightforward to test:

```php
it('allows the post owner to update', function (): void {
    $user = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user->id]);

    expect($user->can('update', $post))->toBeTrue();
});

it('denies non-owners from updating', function (): void {
    $user = User::factory()->create();
    $post = Post::factory()->create(); // Different user

    expect($user->can('update', $post))->toBeFalse();
});

it('allows editors to delete any post', function (): void {
    $editor = User::factory()->create();
    $editor->assignRole('editor');

    $post = Post::factory()->create(); // Not the editor's post

    expect($editor->can('delete', $post))->toBeTrue();
});
```

For HTTP tests, assert the correct status codes:

```php
it('returns 403 when unauthorized', function (): void {
    $user = User::factory()->create();
    $post = Post::factory()->create(); // Not the user's post

    $this->actingAs($user)
        ->put(route('posts.update', $post), ['title' => 'New Title'])
        ->assertForbidden();
});

it('allows the owner to update', function (): void {
    $user = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user->id]);

    $this->actingAs($user)
        ->put(route('posts.update', $post), ['title' => 'New Title'])
        ->assertRedirect();
});
```

For Sanctum API tests, use `actingAs` with a token and its abilities:

```php
use Laravel\Sanctum\Sanctum;

it('allows access with a valid token', function (): void {
    $user = User::factory()->create();

    Sanctum::actingAs($user, ['orders:read']);

    $this->getJson('/api/orders')
        ->assertOk();
});

it('denies access when the token lacks the required ability', function (): void {
    $user = User::factory()->create();

    Sanctum::actingAs($user, ['orders:read']);

    $this->postJson('/api/orders', ['product_id' => 1])
        ->assertForbidden();
});

it('denies access without authentication', function (): void {
    $this->getJson('/api/orders')
        ->assertUnauthorized();
});
```

## Organizing Authorization

As your application grows, keep authorization organized:

```
app/Policies/
├── PostPolicy.php
├── CommentPolicy.php
└── UserPolicy.php

database/seeders/
└── RoleAndPermissionSeeder.php
```

One Policy per model. One seeder for all roles and permissions. Gates in `AppServiceProvider` for non-model checks. This gives every authorization rule a clear home.

## The Authorization Checklist

1. **Use Policies for model-specific authorization** — one Policy per model, auto-discovered by convention
2. **Use Gates for non-model checks** — dashboard access, feature flags, global permissions
3. **Use `Gate::authorize()`** — let Laravel throw the exception instead of manual `abort(403)` calls
4. **Check permissions, not roles** — `$user->can('edit posts')`, not `$user->hasRole('editor')`
5. **Define roles and permissions in seeders** — consistent across environments, version-controlled
6. **Use the `can` middleware on routes** — authorize before the request reaches the controller
7. **Use `@can` in Blade** — show or hide UI elements based on permissions
8. **Share authorization data with Inertia** — frontend checks are for UX, server checks are for security
9. **Combine Policies with Spatie permissions** — Policies handle logic, permissions provide the data
10. **Use Sanctum for API authentication** — tokens for third parties and mobile apps, cookies for SPAs
11. **Scope tokens with abilities** — grant only the access each consumer needs
12. **Combine token abilities with Policies** — abilities restrict the token, Policies restrict the user
13. **Test authorization explicitly** — assert both the allowed and forbidden cases, including token scopes

## Summary

- Authentication answers "who are you?" Authorization answers "what are you allowed to do?" Laravel provides Gates and Policies as built-in mechanisms, and [`spatie/laravel-permission`](https://spatie.be/docs/laravel-permission) adds a database-driven role-and-permission system on top.
- A Gate is a closure for standalone authorization checks not tied to a specific model — dashboard access, feature flags, admin overrides. Define Gates in `AppServiceProvider`.
- A Policy is a class that groups authorization logic around a specific model — one Policy per model, auto-discovered by naming convention. Use Policies as your default for any model-related authorization.
- Use `Gate::authorize()` instead of manual `abort(403)` calls — it throws an `AuthorizationException` that Laravel converts to a 403 response automatically.
- Check permissions, not roles. `$user->can('edit posts')` is future-proof — changing what an "editor" can do means updating the role's permissions in one place, not hunting through application code.
- [`spatie/laravel-permission`](https://spatie.be/docs/laravel-permission) stores roles and permissions in the database and integrates with Laravel's Gate system, Blade directives, middleware, and Form Requests.
- Policies and Spatie permissions complement each other — Policies define the authorization logic (ownership checks, status checks), permissions provide the data that the logic checks against.
- [Sanctum](https://laravel.com/docs/sanctum) provides API token authentication for third-party integrations and mobile apps, and cookie-based session authentication for first-party SPAs. Scope tokens with abilities to grant only the access each consumer needs.
- Combine token abilities with Policies for defense in depth — abilities restrict what the token can do, Policies restrict what the user can do.
- Test authorization explicitly — assert both allowed and forbidden cases, including token abilities and role-based access.

## References

- [Authorization](https://laravel.com/docs/authorization) — Laravel Documentation
- [Gates](https://laravel.com/docs/authorization#gates) — Laravel Documentation
- [Creating Policies](https://laravel.com/docs/authorization#creating-policies) — Laravel Documentation
- [Laravel Sanctum](https://laravel.com/docs/sanctum) — Laravel Documentation
- [API Token Authentication](https://laravel.com/docs/sanctum#api-token-authentication) — Laravel Sanctum Documentation
- [SPA Authentication](https://laravel.com/docs/sanctum#spa-authentication) — Laravel Sanctum Documentation
- [spatie/laravel-permission](https://spatie.be/docs/laravel-permission/v7/introduction) — Spatie Documentation
- [Basic Usage](https://spatie.be/docs/laravel-permission/v7/basic-usage/basic-usage) — Spatie Permission Documentation
- [Using Permissions via Roles](https://spatie.be/docs/laravel-permission/v7/basic-usage/role-permissions) — Spatie Permission Documentation
- [Using Middleware](https://spatie.be/docs/laravel-permission/v7/basic-usage/middleware) — Spatie Permission Documentation
- [When to Use Policies vs Gates in Laravel](https://laravel-news.com/when-to-use-gate-and-policy) — Laravel News
- [Laravel Roles and Permissions: Gates and Policies Explained](https://laraveldaily.com/post/laravel-roles-permissions-gates-policies) — Laravel Daily
