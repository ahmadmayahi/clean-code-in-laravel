In many codebases, validation is scattered — some in controllers, some in models, some in JavaScript. Laravel gives you two clean options: inline validation with `$request->validate()` for simple cases, and Form Requests for everything else.

A [Form Request](https://laravel.com/docs/validation#form-request-validation) is a dedicated class that handles validation and authorization for a single HTTP request. It keeps your [controllers](/books/clean-code-in-laravel/controllers) thin, makes your validation rules testable, and provides a single source of truth for what data a given endpoint accepts. Whether the extra class is worth it depends on the complexity of your validation — we cover that decision in detail at the end of this chapter.

## Creating a Form Request

Laravel provides an Artisan command to generate Form Request classes:

```bash
php artisan make:request StoreUserRequest
```

This creates `app/Http/Requests/StoreUserRequest.php` with a skeleton `authorize()` and `rules()` method. The naming convention is `{Action}{Model}Request` — `StoreUserRequest`, `UpdateOrderRequest`, `DestroyCommentRequest`.

For nested organization by domain, pass a path:

```bash
php artisan make:request Order/StoreOrderRequest
```

This creates the file at `app/Http/Requests/Order/StoreOrderRequest.php`.

## From Inline Validation to Form Requests

Here is the progression from inline validation to a Form Request:

```php
// Before: validation in the controller
public function store(Request $request): RedirectResponse
{
    $validated = $request->validate([
        'name' => 'required|string|max:255',
        'email' => 'required|email|unique:users',
        'password' => 'required|min:8|confirmed',
    ]);

    User::create($validated);
    return redirect()->route('users.index');
}
```

This works for simple cases, but it clutters the controller. The moment you need the same validation in an API endpoint, you duplicate the rules.

```php
// After: Form Request
class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users'],
            'password' => ['required', 'min:8', 'confirmed'],
        ];
    }
}

// Controller is now clean
public function store(StoreUserRequest $request): RedirectResponse
{
    User::create($request->validated());
    return redirect()->route('users.index');
}
```

Notice that we use array syntax for rules (`['required', 'string', 'max:255']`) instead of pipe syntax (`'required|string|max:255'`). Array syntax is easier to read, easier to modify, and required when using custom Rule objects.

## Authorization in Form Requests

The `authorize()` method determines whether the current user is allowed to make this request. Use it for simple authorization checks:

```php
class UpdateOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        // Only the order owner can update it
        return $this->user()->id === $this->route('order')->user_id;
    }

    public function rules(): array
    {
        return [
            'shipping_address_id' => ['required', 'exists:addresses,id'],
            'notes' => ['nullable', 'string', 'max:1000'],
        ];
    }
}
```

If `authorize()` returns `false`, Laravel automatically returns a 403 response. For complex authorization, use [Policies](https://laravel.com/docs/authorization#creating-policies) instead and return `true` from `authorize()`. We cover authentication and authorization in detail in the [Authorization](/books/clean-code-in-laravel/authorization) chapter.

## Custom Error Messages and Attribute Names

Form Requests let you customize error messages and attribute names:

```php
class StoreProductRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'price' => ['required', 'numeric', 'min:0.01'],
            'sku' => ['required', 'string', 'unique:products,sku'],
            'category_id' => ['required', 'exists:categories,id'],
        ];
    }

    public function messages(): array
    {
        return [
            'sku.unique' => 'This SKU is already in use by another product.',
            'price.min' => 'The price must be at least $0.01.',
        ];
    }

    public function attributes(): array
    {
        return [
            'category_id' => 'category',
            'sku' => 'SKU',
        ];
    }
}
```

## Preparing Input Before Validation

Sometimes you need to normalize or transform input before validation runs. Use the `prepareForValidation()` method:

```php
class StoreArticleRequest extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->title),
            'published_at' => $this->boolean('is_published') ? now() : null,
        ]);
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'slug' => ['required', 'string', 'unique:articles,slug'],
            'body' => ['required', 'string'],
            'is_published' => ['boolean'],
        ];
    }
}
```

## After Validation Hooks

For validation that depends on multiple fields or requires database lookups, use the `after()` method:

```php
class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'exists:products,id'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
            'coupon_code' => ['nullable', 'string'],
        ];
    }

    public function after(): array
    {
        return [
            function (Validator $validator): void {
                foreach ($this->input('items', []) as $index => $item) {
                    $product = Product::find($item['product_id']);

                    if ($product && $product->stock < $item['quantity']) {
                        $validator->errors()->add(
                            "items.{$index}.quantity",
                            "Not enough stock for {$product->name}. Available: {$product->stock}",
                        );
                    }
                }
            },
        ];
    }
}
```

## Conditional Validation

Real-world forms are rarely static. A "company" field might only be required for business accounts. A "shipping address" might only matter when the user chooses delivery instead of pickup. Laravel provides [several ways](https://laravel.com/docs/validation#conditionally-adding-rules) to express these conditions.

The simplest approach is `required_if` and `exclude_if`:

```php
class StoreRegistrationRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'account_type' => ['required', Rule::in(['personal', 'business'])],
            'company_name' => ['required_if:account_type,business', 'string', 'max:255'],
            'vat_number' => ['required_if:account_type,business', 'string'],
            'date_of_birth' => ['exclude_if:account_type,business', 'required', 'date', 'before:today'],
        ];
    }
}
```

When `account_type` is `business`, `company_name` and `vat_number` become required, while `date_of_birth` is excluded entirely - it will not appear in the validated data even if the user submits it.

For more complex conditions, use `Rule::requiredIf()` with a closure:

```php
public function rules(): array
{
    return [
        'delivery_method' => ['required', Rule::in(['pickup', 'shipping'])],
        'shipping_address' => [
            Rule::requiredIf(fn (): bool => $this->input('delivery_method') === 'shipping'),
            'string',
        ],
        'pickup_location_id' => [
            Rule::requiredIf(fn (): bool => $this->input('delivery_method') === 'pickup'),
            'exists:locations,id',
        ],
    ];
}
```

The closure receives no arguments and returns a boolean. This is cleaner than `required_if` when the condition involves multiple fields or business logic.

## Post-Processing with `passedValidation()`

We saw `prepareForValidation()` for normalizing input *before* validation. Laravel also provides `passedValidation()` for transforming data *after* validation passes:

```php
class StoreTagRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'tags' => ['required', 'string'],
        ];
    }

    protected function passedValidation(): void
    {
        $this->replace([
            'name' => $this->validated('name'),
            'tags' => collect(explode(',', $this->validated('tags')))
                ->map(fn (string $tag): string => trim($tag))
                ->filter()
                ->values()
                ->all(),
        ]);
    }
}
```

After validation, the `tags` field is transformed from a comma-separated string into a clean array. The controller receives the processed data without needing to know about the transformation.

Use `prepareForValidation()` when you need to normalize input so validation rules can run correctly (e.g., generating a slug). Use `passedValidation()` when the transformation only makes sense after you know the data is valid (e.g., parsing a string into an array).

## Custom Validation Rules

For reusable validation logic, create [custom Rule classes](https://laravel.com/docs/validation#custom-validation-rules). Generate one with `php artisan make:rule NoDisposableEmail`. Laravel places them in `app/Rules/` — as discussed in [Organizing Your Application](/books/clean-code-in-laravel/organizing-your-application), validation rules do not belong in Support:

```php
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class NoDisposableEmail implements ValidationRule
{
    private array $disposableDomains = [
        'mailinator.com',
        'guerrillamail.com',
        'tempmail.com',
    ];

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $domain = Str::after($value, '@');

        if (in_array($domain, $this->disposableDomains)) {
            $fail('Disposable email addresses are not allowed.');
        }
    }
}

// Usage in a Form Request
public function rules(): array
{
    return [
        'email' => ['required', 'email', new NoDisposableEmail()],
    ];
}
```

## Spatie Validation Rules

Before writing every custom rule from scratch, check if [`spatie/laravel-validation-rules`](https://github.com/spatie/laravel-validation-rules) already has what you need. The package provides a set of practical rules that cover common validation scenarios:

```bash
composer require spatie/laravel-validation-rules
```

### Validating Delimited Strings

The `Delimited` rule validates a string containing separated values — comma-separated emails, pipe-separated tags, or semicolon-separated IDs. Each value is validated against a rule you provide:

```php
use Spatie\ValidationRules\Rules\Delimited;

public function rules(): array
{
    return [
        'recipients' => [new Delimited('email')],
    ];
}
```

This accepts `'alice@example.com, bob@example.com'` but rejects `'alice@example.com, not-an-email'`. You can chain constraints:

```php
// At least 2, at most 10 email addresses, no duplicates
'recipients' => [(new Delimited('email'))->min(2)->max(10)],

// Semicolon-separated instead of comma
'tags' => [(new Delimited('string|max:50'))->separatedBy(';')],
```

This is a cleaner alternative to asking users to submit arrays or manually splitting strings in `prepareForValidation()`.

### Validating That Models Exist

The `ModelsExist` rule checks that every value in an array corresponds to an existing model — useful for bulk operations where you receive an array of IDs:

```php
use Spatie\ValidationRules\Rules\ModelsExist;

public function rules(): array
{
    return [
        'product_ids' => ['required', 'array', new ModelsExist(Product::class)],
    ];
}
```

By default it checks the primary key. Pass a second argument to check a different column:

```php
'user_emails' => ['array', new ModelsExist(User::class, 'email')],
```

### Validating Country Codes and Currencies

For international applications, the `CountryCode` and `Currency` rules validate against ISO standards. `CountryCode` requires the `league/iso3166` package, and `Currency` requires `alcohol/iso4217`:

```php
use Spatie\ValidationRules\Rules\CountryCode;
use Spatie\ValidationRules\Rules\Currency;

public function rules(): array
{
    return [
        'country' => ['required', new CountryCode()],   // 'NL', 'US', 'DE'
        'currency' => ['required', new Currency()],       // 'EUR', 'USD', 'GBP'
    ];
}
```

### Authorization During Validation

The `Authorized` rule checks whether the authenticated user is authorized to perform an action on a model — combining [authorization](/books/clean-code-in-laravel/authorization) with validation in a single step:

```php
use Spatie\ValidationRules\Rules\Authorized;

public function rules(): array
{
    return [
        'post_id' => ['required', new Authorized('edit', Post::class)],
    ];
}
```

This validates that the `post_id` exists *and* that the current user is authorized to edit it, based on the `PostPolicy`. It is especially useful for forms where users select from a list of models — you validate both existence and permission in one rule.

## Connecting Form Requests to DTOs

Form Requests validate data. [DTOs](/books/clean-code-in-laravel/data-transfer-objects) carry data. The cleanest pattern is to have the Form Request produce a DTO:

```php
class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'exists:products,id'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
            'shipping_address_id' => ['required', 'exists:addresses,id'],
        ];
    }

    public function toDto(): PlaceOrderData
    {
        return new PlaceOrderData(
            userId: $this->user()->id,
            shippingAddressId: $this->validated('shipping_address_id'),
            items: collect($this->validated('items'))->map(
                fn (array $item): OrderItemData => new OrderItemData(
                    productId: $item['product_id'],
                    quantity: $item['quantity'],
                    unitPrice: Product::find($item['product_id'])->price,
                ),
            ),
        );
    }
}
```

The controller stays thin:

```php
public function store(StoreOrderRequest $request, PlaceOrderAction $action): RedirectResponse
{
    $order = $action->execute($request->toDto());

    return redirect()->route('orders.show', $order);
}
```

This is the pattern we introduced in the [Actions](/books/clean-code-in-laravel/actions) chapter. The Form Request validates, produces a [DTO](/books/clean-code-in-laravel/data-transfer-objects), and the Action handles the business logic.

## Validation for Update Requests

Update requests often need to ignore the current record when checking uniqueness. Use the [`Rule`](https://laravel.com/docs/validation#rule-unique) class:

```php
class UpdateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => [
                'required',
                'email',
                Rule::unique('users')->ignore($this->route('user')),
            ],
        ];
    }
}
```

## Sharing Rules Between Store and Update

A `StoreUserRequest` and `UpdateUserRequest` often share 80% of their rules. The temptation is to use one Form Request for both, but that leads to messy conditionals. A cleaner approach is a trait:

```php
namespace App\Http\Requests\Concerns;

trait HasUserRules
{
    protected function userRules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'role' => ['required', Rule::in(['admin', 'editor', 'viewer'])],
        ];
    }
}
```

```php
class StoreUserRequest extends FormRequest
{
    use HasUserRules;

    public function rules(): array
    {
        return [
            ...$this->userRules(),
            'email' => ['required', 'email', 'unique:users'],
            'password' => ['required', 'min:8', 'confirmed'],
        ];
    }
}

class UpdateUserRequest extends FormRequest
{
    use HasUserRules;

    public function rules(): array
    {
        return [
            ...$this->userRules(),
            'email' => ['required', 'email', Rule::unique('users')->ignore($this->route('user'))],
        ];
    }
}
```

The shared rules live in one place. Each Form Request adds or overrides what it needs. The `password` field is only required on creation, and the `email` uniqueness check ignores the current record on update.

## Nested and Array Validation

Laravel handles [nested data](https://laravel.com/docs/validation#validating-nested-array-input) gracefully with dot notation:

```php
class StoreCompanyRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'address.street' => ['required', 'string'],
            'address.city' => ['required', 'string'],
            'address.country' => ['required', 'string', 'size:2'],
            'contacts' => ['required', 'array', 'min:1'],
            'contacts.*.name' => ['required', 'string'],
            'contacts.*.email' => ['required', 'email'],
            'contacts.*.role' => ['required', Rule::in(['owner', 'billing', 'technical'])],
        ];
    }
}
```

## Testing Form Requests

The most effective way to test validation is through [HTTP tests](https://laravel.com/docs/http-tests) — they exercise the full request lifecycle, including authorization, validation, and error responses. [Pest's](https://pestphp.com/docs/datasets) datasets make this especially clean:

```php
it('requires valid data to create a user', function (string $field, mixed $value) {
    $validData = [
        'name' => 'Ahmad Mayahi',
        'email' => 'ahmad@example.com',
        'password' => 'secret-password',
        'password_confirmation' => 'secret-password',
    ];

    $admin = User::factory()->admin()->create();

    $response = $this->actingAs($admin)
        ->post(route('users.store'), [...$validData, $field => $value]);

    $response->assertSessionHasErrors($field);
})->with([
    'name is required' => ['name', ''],
    'name is too long' => ['name', str_repeat('a', 256)],
    'email is required' => ['email', ''],
    'email must be valid' => ['email', 'not-an-email'],
    'password is required' => ['password', ''],
    'password is too short' => ['password', 'short'],
]);
```

Each dataset entry runs as a separate test. When one fails, the label tells you exactly which rule broke. Adding a new rule means adding one line to the dataset, not an entire test method.

For custom Rule objects, test them in isolation:

```php
it('rejects disposable email domains', function (string $email) {
    $rule = new NoDisposableEmail();

    $failed = false;

    $rule->validate('email', $email, function () use (&$failed): void {
        $failed = true;
    });

    expect($failed)->toBeTrue();
})->with([
    'mailinator.com' => ['test@mailinator.com'],
    'guerrillamail.com' => ['test@guerrillamail.com'],
]);

it('allows legitimate email domains', function (): void {
    $rule = new NoDisposableEmail();

    $failed = false;

    $rule->validate('email', 'user@gmail.com', function () use (&$failed): void {
        $failed = true;
    });

    expect($failed)->toBeFalse();
});
```

## When to Use Form Requests

Form Requests are not always the right choice. They move validation and authorization one step away from the controller, which is a trade-off: you reduce clutter in the controller, but the reader now has to open a separate class to see what the endpoint accepts and who can access it.

This trade-off is worth it when the validation or authorization logic is substantial enough to distract from the controller's core logic. Think conditional rules, cross-field validation with `after()`, shared rules between store and update, or authorization checks that involve relationship lookups. In these cases, a Form Request gives the logic a proper home and makes it independently testable.

The trade-off is not worth it when the validation is simple enough that your eyes can blur past it. A controller that validates three fields with basic rules like `required|string|max:255` is perfectly readable. Extracting that into a dedicated class creates a file, an import, and a level of indirection — all for logic that was not causing any real distraction.

There is also a subtlety that catches many teams: Form Request injection is a hidden side effect. When you type-hint `StoreUserRequest` instead of `Request`, Laravel automatically calls `authorize()` and runs validation before the controller method executes. The developer reading the controller sees a parameter and nothing else. There is no explicit `$request->validate()` call, no `$this->authorize()` — the behavior is invisible unless you know to look for it.

This is fine on teams that have adopted Form Requests as a convention and know to check the request class. But on teams where the convention is not established, developers often miss that authorization and validation are happening at all. Some teams solve this by keeping simple validation inline and only extracting to a Form Request when the complexity justifies it, making the extraction itself a signal that "there is non-trivial logic here."

Use a Form Request when:

- The validation rules span more than a handful of lines, or include conditional logic, cross-field checks, or custom Rule objects
- The same validation needs to be reused across multiple endpoints (web and API, for example)
- The authorization logic involves relationship lookups or policy checks that would clutter the controller
- You need `prepareForValidation()`, `passedValidation()`, or `after()` hooks
- You want to bridge validation to a DTO with a `toDto()` method

Keep validation inline when:

- The endpoint has a few simple rules that are easy to skim past
- The authorization is a one-liner or handled entirely by a Policy
- The validation is unique to this single endpoint and unlikely to be reused
- The controller is already thin enough that the validation does not compete for attention

The goal is not to use Form Requests everywhere. The goal is to move logic out of the controller only when leaving it in would make the controller harder to read.

## The Form Request Checklist

1. **One Form Request per controller method** — `StoreUserRequest`, `UpdateUserRequest`
2. **Array syntax for rules** — easier to read and required for Rule objects
3. **Custom messages for user-facing errors** — do not expose technical validation messages
4. **`prepareForValidation()`** for input normalization — slugs, formatting, defaults
5. **`passedValidation()`** for post-processing — parse strings into arrays, restructure data
6. **`after()`** for complex cross-field validation — stock checks, business rules
7. **Conditional rules** — use `required_if`, `exclude_if`, and `Rule::requiredIf()` for dynamic forms
8. **Shared rules via traits** — extract common rules into `Has*Rules` traits instead of duplicating them
9. **`toDto()`** method to bridge validation and business logic — clean handoff to Actions
10. **Test with datasets** — one test with labeled data covers all your rules without repetition

## Summary

- Form Requests are a trade-off: they reduce clutter in the controller at the cost of moving logic one step away. Use them when validation or authorization is complex enough to distract from the controller's core logic. Keep validation inline when the rules are simple and easy to skim past.
- A Form Request is a dedicated class for validation and authorization. It keeps [controllers](/books/clean-code-in-laravel/controllers) thin and makes validation rules testable and reusable across web and API endpoints.
- Use array syntax for rules (`['required', 'string']`) instead of pipe syntax (`'required|string'`). Array syntax is easier to read and required when using custom Rule objects.
- Use `authorize()` for simple authorization checks. For complex authorization, use [Policies](https://laravel.com/docs/authorization#creating-policies) and return `true` from `authorize()`.
- Use `prepareForValidation()` to normalize input before validation runs (generating slugs, formatting values). Use `passedValidation()` to transform data after validation passes (parsing strings into arrays).
- The `after()` method handles cross-field validation that requires database lookups or business logic — like checking stock availability across multiple items.
- Use `required_if`, `exclude_if`, and `Rule::requiredIf()` for conditional validation. Fields can be required, optional, or excluded entirely based on other input values.
- Bridge Form Requests to [Actions](/books/clean-code-in-laravel/actions) with a `toDto()` method. The Form Request validates, produces a typed [DTO](/books/clean-code-in-laravel/data-transfer-objects), and the Action handles the business logic.
- Share rules between Store and Update requests with traits. Extract common rules into `Has*Rules` traits instead of duplicating them or using conditionals in a single Form Request.
- Test validation through [HTTP tests](https://laravel.com/docs/http-tests) with Pest datasets. Each dataset entry runs as a separate test, and labels tell you exactly which rule broke.

## References

- [Validation](https://laravel.com/docs/validation) — Laravel Documentation
- [Form Request Validation](https://laravel.com/docs/validation#form-request-validation) — Laravel Documentation
- [Custom Validation Rules](https://laravel.com/docs/validation#custom-validation-rules) — Laravel Documentation
- [Laravel Form Requests](https://ahmedash.dev/blog/laravel-core-bits/laravel-form-request/) — Ahmed Ammar
- [Manipulating Request Data Before Performing Validation](https://sampo.co.uk/blog/manipulating-request-data-before-performing-validation-in-laravel) — Ben Sampson
- [Laravel Conditional Validation Based on Other Fields](https://laraveldaily.com/post/laravel-conditional-validation-other-fields-examples) — Laravel Daily
- [Mapping Requests to DTOs Inside Form Requests](https://www.sandulat.com/mapping-requests-to-dtos-inside-form-requests/) — Sandulat
- [spatie/laravel-validation-rules](https://github.com/spatie/laravel-validation-rules) — Spatie, GitHub
- [Datasets](https://pestphp.com/docs/datasets) — Pest Documentation
