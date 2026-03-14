One of the most consistent messages from [Taylor Otwell](https://github.com/taylorotwell) is a warning against being too "clever." In a [2025 episode of the Maintainable podcast](https://www.youtube.com/watch?v=U2Ah6J7X4Ks), he described the pattern clearly:

> We as developers sometimes tend to try to build these beautiful cathedrals of software... but we can sometimes create very complex cathedrals of complexity that aren't so easy to change. Software should be a little bit more simple and disposable and easy to change.

This is not a throwaway quote. It is the single most important idea in this entire book. The Laravel framework was not built to impress computer science professors. It was built so that working developers could ship real applications without drowning in abstraction.

[Clever code](https://www.reddit.com/r/programming/comments/1jxpl9c/clever_code_is_probably_the_worst_code_you_could/) is code that makes the author feel smart. Simple code is code that makes the reader feel smart. There is a world of difference between the two, and Taylor has spent over fourteen years choosing the latter.

Consider this example. A clever and compact ternary chain:

```php
$status = $user->isAdmin() ? ($order->isPaid() ? 'approved' : 'pending') : 'rejected';
```

Now look at the same logic written simply:

```php
public function determineOrderStatus(Order $order): string
{
    /** @var \App\Models\User $user */
    $user = $order->user;

    if ($user->isAdmin()) {
        if ($order->isPaid()) {
            return 'approved';
        }

        return 'pending';
    }

    return 'rejected';
}
```

The second version is longer, but any developer on your team can read it and understand what it does - including the one who joins six months from now. That is the trade Taylor makes every single time: more lines for more clarity.

## Convention Over Configuration

[Convention over configuration](https://rubyonrails.org/doctrine/convention-over-configuration) is one of the most influential ideas in modern web development, popularized by [Ruby on Rails](https://rubyonrails.org/) and deeply embedded in Laravel's DNA.

Laravel is an opinionated framework. It has opinions about where your controllers go, how your models are named, what your migration files look like, and how your routes are structured. These opinions are not arbitrary - they are conventions that eliminate thousands of small decisions from your daily work.

When you follow Laravel's conventions, every Laravel developer who reads your code already knows where to find things. The framework's tooling - [Artisan commands](https://laravel.com/docs/artisan), [route model binding](https://laravel.com/docs/routing#route-model-binding), [automatic dependency injection](https://laravel.com/docs/container) - works seamlessly because it expects things to be in certain places. And you stop wasting mental energy on decisions that do not matter.

Here is a practical example. Laravel expects a `User` model to map to a `users` table. You do not need to configure this. You do not need a mapping file. You do not need an annotation. It just works:

```php
// Laravel knows this maps to the "users" table
class User extends Model
{
    // No $table property needed
}
```

The moment you fight this convention - naming your model `UserAccount` but keeping the `users` table, or naming your table `tbl_users` - you create friction. Now you need explicit configuration:

```php
class UserAccount extends Model
{
    protected $table = 'users'; // Fighting the convention
}
```

Every piece of explicit configuration is a small tax on every developer who reads your code. Sometimes that tax is worth paying. Most of the time, it is not.

## Code That Reads Like English

One of Laravel's most distinctive qualities is how its code reads. Taylor designs APIs so that method chains feel like sentences rather than instructions. Look at this [Eloquent](https://laravel.com/docs/eloquent) query:

```php
$users = User::where('active', true)
    ->whereHas('subscription', function (Builder $query): void {
        $query->where('plan', 'premium');
    })
    ->orderBy('name')
    ->get();
```

You can read that aloud: "Get users where active is true, where they have a subscription with a premium plan, ordered by name." Now compare the same thing in raw SQL:

```sql
SELECT *
FROM users
WHERE active = 1
AND EXISTS (
    SELECT 1
    FROM subscriptions
    WHERE subscriptions.user_id = users.id
      AND subscriptions.plan = 'premium'
)
ORDER BY name ASC;
```

SQL tells the database how to fetch the data. Eloquent tells the reader what you want. Both work, but one is easier to come back to six months later.

This pattern runs through the entire framework. The [query builder](https://laravel.com/docs/queries), the [collection methods](https://laravel.com/docs/collections), the [validation](https://laravel.com/docs/validation) rules - they are all designed to chain into readable sentences. When you write `$collection->where('active', true)->sortBy('name')->values()`, even someone unfamiliar with PHP can roughly follow what it does.

When you write your own code, ask yourself: can I read this line aloud and have it make sense?

## Embrace the Framework

One of the most common mistakes in the Laravel community is fighting the framework. Developers coming from other ecosystems often try to impose patterns that do not fit - custom validation layers, hand-rolled authentication systems, or abstraction layers on top of things Laravel already handles.

Here is an example. A developer coming from a framework without built-in validation might build something like this:

```php
class UserValidator
{
    public function validate(array $data): array
    {
        $errors = [];

        if (empty($data['email'])) {
            $errors[] = 'Email is required.';
        }

        if (! filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            $errors[] = 'Email is not valid.';
        }

        if (User::where('email', $data['email'])->exists()) {
            $errors[] = 'Email is already taken.';
        }

        return $errors;
    }
}
```

Laravel already does all of this in one line:

```php
$validated = $request->validate(['email' => ['required', 'email', 'unique:users']]);
```

And that is just validation. The same pattern applies across the framework:

```php
// Model not found? 404 automatically.
$user = User::findOrFail($id);

// Caching? One line.
$users = Cache::remember('users', 3600, fn () => User::all());

// File storage? One line.
$path = $request->file('avatar')->store('avatars');
```

This does not mean you should never build custom solutions. It means you should reach for Laravel's built-in tools first. The framework is remarkably complete, and the code you do not write is the code that never has bugs.

### PHP Built-in Features Are Not Always the Answer

There is a subtler version of fighting the framework that does not involve building custom solutions from scratch. Instead, developers see a shiny new PHP feature and reach for it as a drop-in replacement for something the framework already handles.

A common example is PHP 8.4 property hooks. You might be tempted to use them inside Eloquent models:

```php
class User extends Model
{
    public string $first_name {
        set(string $value) => $this->attributes['first_name'] = ucfirst($value);
        get => ucfirst($this->attributes['first_name']);
    }
}
```

It looks clean. It uses a modern PHP feature. But it quietly breaks how Eloquent works.

The Laravel way to handle this is with the attribute system:

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => ucfirst($value),
        );
    }
}
```

The difference is not cosmetic. Eloquent's attribute system is wired into the model's lifecycle. Serialization methods like `toArray()`, `toJson()`, and `$appends` all depend on it. Property hooks bypass serialization entirely - `$user->first_name` and `$user->toArray()['first_name']` will return different values. Your model might look like it works when you access a property directly, but it will behave unexpectedly the moment you serialize it for an API response.

Beyond the technical issues, framework conventions are where your team and the ecosystem will look. Nova, Filament, API Resources, and every Laravel package that touches models understands the attribute system. New developers joining your project will check `casts()` and look for `Attribute` methods - not property hooks.

This is not a criticism of property hooks themselves. They are a great PHP feature - in plain PHP classes, DTOs, value objects, and anywhere you are not inside Eloquent's system. But inside a model, Eloquent's tools exist for a reason. Use them.

## You Are Not Going to Need It

Developers love to plan for the future. "What if we need to switch databases?" "What if we need to support multiple payment providers?" "What if this needs to scale to millions of users?" These are valid questions - for later. Right now, they lead to code that solves problems you do not have.

This idea has a name: [YAGNI](https://martinfowler.com/bliki/Yagni.html) - You Aren't Gonna Need It. It means do not build something until you actually need it.

Here is what premature abstraction looks like:

```php
// You have one payment provider: Stripe.
// But "what if we add PayPal later?"

interface PaymentGatewayInterface
{
    public function charge(int $amount, string $currency): PaymentResult;
    public function refund(string $transactionId): RefundResult;
}

class StripePaymentGateway implements PaymentGatewayInterface { /* ... */ }
class PaymentGatewayFactory { /* ... */ }
class PaymentGatewayManager { /* ... */ }
```

You now have four files to maintain instead of one, and you still only use Stripe. If PayPal comes along next year, you can refactor then - and you will have a much better understanding of what the abstraction should look like because you will have a real second use case, not an imaginary one.

The simple version:

```php
class StripeService
{
    public function charge(int $amount, string $currency): PaymentIntent
    {
        return Stripe::paymentIntents()->create([
            'amount' => $amount,
            'currency' => $currency,
        ]);
    }
}
```

One class, no interface, no factory. When you need a second payment provider, that is the time to extract an interface - not before.

### The Repository Pattern in Laravel

The most common example of YAGNI in the Laravel world is the Repository Pattern. The idea is to put a layer between your application and the database so that you can "swap the database later." It looks something like this:

```php
interface UserRepositoryInterface
{
    public function findById(int $id): ?User;
    public function findActiveUsers(): Collection;
    public function create(array $data): User;
}

class EloquentUserRepository implements UserRepositoryInterface
{
    public function findById(int $id): ?User
    {
        return User::find($id);
    }

    public function findActiveUsers(): Collection
    {
        return User::where('active', true)->get();
    }

    public function create(array $data): User
    {
        return User::create($data);
    }
}
```

Every method in this class is a thin wrapper around something Eloquent already does. The interface adds nothing - it just mirrors the methods on the other side. And the justification is always the same: "What if we need to switch from MySQL to MongoDB?"

Be honest with yourself: how often does that happen? In years of building Laravel applications, I have never seen a project that actually needed to swap its database. And even if it did, the repository would not save you - switching databases means changing queries, relationships, indexes, migrations, and data types. A thin wrapper around Eloquent does not protect you from any of that.

Laravel already gives you two powerful layers for querying data: [Eloquent](https://laravel.com/docs/eloquent) for working with models and relationships, and the [Query Builder](https://laravel.com/docs/queries) for more complex queries. Between the two, you can handle virtually anything. If your query logic gets complex, use [Eloquent scopes](https://laravel.com/docs/eloquent#query-scopes) - not a repository that repeats what Eloquent already does.

```php
// Instead of a repository, use scopes
use Illuminate\Database\Eloquent\Attributes\Scope;

class User extends Model
{
    #[Scope]
    protected function active(Builder $query): void
    {
        $query->where('active', true);
    }
}

// Clean, simple, and uses what Laravel gives you
$activeUsers = User::active()->get();
```

The Repository Pattern has its place in frameworks that do not have a built-in ORM, or in applications that genuinely talk to multiple data sources. In a typical Laravel application, it is extra code that solves a problem you do not have.

> The best abstraction is the one you write after you have two concrete implementations, not before you have one.

## Code Is Read More Than It Is Written

As [Robert C. Martin](https://en.wikipedia.org/wiki/Robert_C._Martin) wrote in [Clean Code](https://www.oreilly.com/library/view/clean-code-a/9780136083238/), "the ratio of time spent reading versus writing is well over 10 to 1." This applies even more in a framework like Laravel, where conventions make code predictable - but only when you write with the reader in mind.

You write a line of code once. Your team reads it dozens of times - in code reviews, while debugging, while adding features nearby, while onboarding new developers. Every decision you make while writing should favor the reader, not the writer.

This means:

- Choose clarity over brevity. A longer, obvious name beats a short, cryptic one. `$activeSubscriptions` is better than `$subs`.
- Choose boring over clever. A simple `if` statement that anyone can follow beats a one-liner that requires a second look.
- Choose explicit over implicit. If something is not obvious from the code, make it obvious - even if it takes an extra line.

A good rule of thumb: if you need to add a comment to explain what a piece of code does, the code itself is probably too complex. Rewrite it until the comment is unnecessary.

```php
// Bad: needs a comment to explain
// Get users who registered in the last 30 days and have verified their email
$users = User::where('created_at', '>', now()->subDays(30))
    ->whereNotNull('email_verified_at')
    ->get();

// Good: the code explains itself
$users = User::query()
    ->registeredInLast(days: 30)
    ->verified()
    ->get();
```

The second version uses [Eloquent scopes](https://laravel.com/docs/eloquent#query-scopes) to give the query conditions meaningful names. The logic is the same, but the reader does not need a comment to understand it.

## The Three Questions

Before writing any piece of code in a Laravel application, ask yourself three questions:

1. Is this the simplest way to solve this problem?

If you are reaching for a design pattern, a third-party package, or a custom abstraction, pause. Is there a simpler way? Could a plain PHP class do the job? Could a built-in Laravel feature handle it?

2. Will a new team member understand this in thirty seconds?

If your code requires a README, a diagram, or a verbal explanation to understand, it is too complex. The best code explains itself.

3. Am I following Laravel's conventions?

If you are naming things differently, putting files in unusual places, or structuring your application in a non-standard way, you should have a very good reason. Convention is a gift - accept it.

These three questions will guide every decision in this book. When we discuss [Actions](/books/clean-code-in-laravel/actions), Services, DTOs, or Domain-Driven Design in later chapters, we will always come back to these fundamentals: simplicity, clarity, and convention.

## What This Book Is (and What It Is Not)

This book is not a Laravel tutorial. It assumes you already know how to build Laravel applications - you understand routing, controllers, Eloquent, Blade, and the basics of the framework.

This book is about how to build Laravel applications well. It is a style guide, an architecture guide, and a philosophy guide - all grounded in practical, real-world code examples using Laravel 12. Every pattern we discuss, every convention we recommend, and every package we introduce serves one purpose: making your code simpler, clearer, and more maintainable.

We start with the small things - naming, code style, controller structure - and gradually build toward larger architectural patterns like Domain-Driven Design and Event Sourcing. Each chapter builds on the previous one. No concept is used before it is explained.

By the end of this book, you will not just write Laravel code. You will write Laravel code that any Laravel developer can read, understand, and maintain.

## Summary

- Simple beats clever. Write code that makes the reader feel smart, not the author. More lines for more clarity is always a good trade.
- Follow Laravel's conventions. The framework expects things in certain places. When you follow those expectations, the tooling works for you and every Laravel developer already knows their way around your code.
- Let code read like English. Eloquent, collections, and validation are designed to chain into readable sentences. Write your own code the same way.
- Use what Laravel gives you. Before building a custom solution, check if the framework already solves the problem. It usually does.
- Do not build for imaginary requirements. Solve the problem in front of you. Abstractions, interfaces, and patterns like the Repository Pattern can wait until you have a real reason to reach for them.
- Favor the reader. You write code once. Your team reads it dozens of times. Choose clarity over brevity, boring over clever, and explicit over implicit.
- Ask three questions before writing code: Is this the simplest way? Will a new team member understand it quickly? Am I following Laravel's conventions?

