Every `composer require` is a decision. It is a line you add to your `composer.json` that says: "I trust this code, I trust the people who maintain it, and I accept that my application now depends on it." That trust is not free. It comes with a maintenance cost, an upgrade path, and a risk — the risk that the package is abandoned, introduces a breaking change, or does something subtly different from what you expected.

Packages are one of the greatest strengths of the Laravel ecosystem. The community has built extraordinary tools that save weeks of development time. But every package you install is a dependency, and every dependency is a bet. This chapter is about making those bets deliberately — knowing when a package is the right call, when it is not, and which packages have earned their place in the ecosystem.

## Every Package Is a Dependency

When you install a package, you are not just adding functionality. You are adding:

- **A maintenance obligation.** When Laravel releases a new major version, you need every package to support it before you can upgrade. One unmaintained package can block your entire upgrade path.
- **An implicit trust relationship.** You are running someone else's code in your application. That code has access to your database, your environment variables, your users' data. You trust the maintainers not to introduce vulnerabilities, backdoors, or careless bugs.
- **A learning curve for your team.** Every package has its own API, its own conventions, its own quirks. A new developer joining your team does not just need to learn Laravel — they need to learn every package you have installed.
- **A potential point of failure.** If the package has a bug, you are now debugging someone else's code. If the package is abandoned, you are maintaining a fork or finding a replacement.

None of this means you should avoid packages. It means you should be deliberate. Before running `composer require`, ask yourself three questions:

1. **Does Laravel already solve this?** Laravel ships with authentication, authorization, rate limiting, caching, queues, file storage, mail, notifications, and much more. Check the framework's documentation before reaching for a package.
2. **Can I build this myself in a reasonable amount of time?** If the feature is straightforward and you can implement it in an hour, you probably do not need a package for it. Your own code is code you understand, control, and can change without waiting for a pull request to be merged.
3. **Is this package well-maintained?** Check the last commit date, the number of open issues, the release cadence, and whether the maintainer responds to bug reports. A package with thousands of stars but no activity in two years is a liability, not an asset.

## You Do Not Always Need a Package

There is a tendency in the Laravel ecosystem — and in the PHP world generally — to reach for a package before thinking about the problem. Need to call an API? Find a package. Need to generate a slug? Find a package. Need to format a phone number? Find a package.

Sometimes this makes sense. A package for image manipulation, PDF generation, or role-based permissions handles genuinely complex problems with edge cases you do not want to discover yourself. But for simple integrations — especially API calls — a package is often overkill.

### The OpenAI Example

Take the [OpenAI API](https://platform.openai.com/docs/api-reference). There are dedicated PHP packages for it, complete with typed responses, multiple client methods, and Laravel-specific wrappers. I am not against these packages — they are well-built and useful for applications that use AI extensively. But let me be honest: I do not like having dependencies for things I can easily implement.

Calling the OpenAI API to generate text is an HTTP POST request with a JSON body. Laravel's [HTTP client](https://laravel.com/docs/http-client) handles this in a few lines:

```php
use Illuminate\Support\Facades\Http;

class OpenAiService
{
    public function generateText(string $prompt, string $model = 'gpt-4o'): string
    {
        $response = Http::withToken(config('services.openai.api_key'))
            ->post('https://api.openai.com/v1/chat/completions', [
                'model' => $model,
                'messages' => [
                    ['role' => 'user', 'content' => $prompt],
                ],
            ]);

        return $response->json('choices.0.message.content');
    }
}
```

That is it. No package, no abstraction layer, no additional dependency. You have full control over the request, the response, and the error handling. You can test it with `Http::fake()` — Laravel's built-in HTTP fake that you already know and use.

```php
it('generates text from OpenAI', function (): void {
    Http::fake([
        'api.openai.com/*' => Http::response([
            'choices' => [
                ['message' => ['content' => 'Hello from the AI.']],
            ],
        ]),
    ]);

    $result = app(OpenAiService::class)->generateText('Say hello');

    expect($result)->toBe('Hello from the AI.');
});
```

Compare this to installing a dedicated package: you gain typed response objects and method autocompletion, but you also gain a dependency that must be updated when the API changes, that may lag behind new features, and that adds its own abstraction on top of what is fundamentally a simple HTTP call.

The point is not that OpenAI packages are bad. The point is that you should ask yourself: *does this package save me enough time and complexity to justify the dependency?* For a simple API call, the answer is usually no. For an application that uses streaming responses, function calling, embeddings, and multiple models, the answer might be yes.

This principle applies broadly. Stripe, Twilio, SendGrid — many third-party services have PHP SDKs. Some are genuinely useful. Others are thin wrappers around HTTP calls that you could make yourself. Evaluate each one on its own merits.

## Packages Worth Installing

Not every dependency is a burden. Some packages solve problems so well, so completely, and with such reliability that building your own version would be irresponsible. These are the packages that have proven themselves across thousands of Laravel applications and are maintained by teams you can trust.

### Development Tools

These packages belong in your `require-dev` section. They make development faster and debugging easier, but they never touch production.

**[Laravel Debugbar](https://github.com/fruitcake/laravel-debugbar)** (`fruitcake/laravel-debugbar`) is the single most useful development tool in the Laravel ecosystem. It adds a toolbar to your browser that shows every database query, every view rendered, every cache hit, every mail sent, and how long each part of the request took. It catches N+1 queries before they reach production. If you are not using it, you are debugging blind.

**[Larastan](https://github.com/larastan/larastan)** (`larastan/larastan`) brings static analysis to Laravel. It is a [PHPStan](https://phpstan.org/) extension that understands Eloquent, Facades, collections, and the service container. It catches type errors, undefined methods, and incorrect return types without running your code. Pair it with the [Code Quality and Automation](/books/clean-code-in-laravel/code-quality-and-automation) tools from earlier in this book to catch bugs before they are committed.

**[IDE Helper](https://github.com/barryvdh/laravel-ide-helper)** (`barryvdh/laravel-ide-helper`) generates helper files so your IDE understands Laravel's magic — Facade methods, model attributes, relation return types. It turns "I think this method exists" into autocompletion and type-checking. Essential for PhpStorm and useful for VS Code.

### Data and Structure

**[Laravel Data](https://spatie.be/docs/laravel-data)** (`spatie/laravel-data`) is a powerful package that unifies data objects, validation, and API transformation into a single class. If you have read the [DTOs](/books/clean-code-in-laravel/data-transfer-objects) chapter, you know the value of typed data structures. Laravel Data takes that idea further — a single data class can serve as a DTO, a Form Request replacement, and an API Resource, with built-in casting, validation rules, and TypeScript generation.

```php
use Spatie\LaravelData\Data;

class OrderData extends Data
{
    public function __construct(
        public int $user_id,
        public int $amount_in_cents,
        public string $currency = 'USD',
    ) {}
}

// Use it as a DTO
$data = OrderData::from($request);

// Use it as an API resource
return OrderData::from($order);

// Validate incoming requests automatically
OrderData::validate($request->all());
```

### Authorization and Access Control

**[Laravel Permission](https://github.com/spatie/laravel-permission)** (`spatie/laravel-permission`) is the standard for role and permission management. It stores roles and permissions in the database, provides middleware for route protection, Blade directives for view-level checks, and Eloquent scopes for query-level filtering. Building your own role system is a mistake you will regret when edge cases start appearing — permission inheritance, multi-guard support, cache invalidation. This package handles all of it.

### File Management

**[Laravel Medialibrary](https://github.com/spatie/laravel-medialibrary)** (`spatie/laravel-medialibrary`) associates files with Eloquent models. It handles uploads, generates thumbnails and responsive images, stores files across multiple disks, and cleans up orphaned files when models are deleted. File management looks simple until you need image conversions, multiple storage disks, and proper cleanup. This package does it right.

### Backup and Recovery

**[Laravel Backup](https://github.com/spatie/laravel-backup)** (`spatie/laravel-backup`) automates backups of your database and files to local or cloud storage. It supports multiple backup destinations, monitors backup health, and sends notifications when backups fail. No production application should run without automated backups, and building your own backup system is one of those tasks that seems simple until your database is gone and you discover your "backup script" had a silent error for three months.

### Audit Logging

**[Laravel Activitylog](https://github.com/spatie/laravel-activitylog)** (`spatie/laravel-activitylog`) records changes to your Eloquent models — who changed what, when, and what the old values were. This is not optional for most business applications. Audit trails are a regulatory requirement in many industries, and retroactively adding logging to an application is painful. Start with it from day one.

### Queue Monitoring

**[Laravel Horizon](https://laravel.com/docs/horizon)** (`laravel/horizon`) is Laravel's official dashboard for Redis-powered queues. It gives you real-time visibility into job throughput, failure rates, wait times, and worker status. If you are using Redis queues in production — and the [Queue Workers](/books/clean-code-in-laravel/queue-workers) chapter covered why you should — Horizon is not optional. It is how you know your queues are healthy.

### Full-Text Search

**[Laravel Scout](https://laravel.com/docs/scout)** (`laravel/scout`) adds full-text search to your Eloquent models with a single trait. It supports multiple drivers — a database driver for small applications, [Meilisearch](https://www.meilisearch.com/) or [Algolia](https://www.algolia.com/) for production-scale search. Building your own search with `LIKE` queries or raw SQL full-text indexes works until it does not. Scout gives you a clean API with room to scale.

### Excel Import and Export

**[Laravel Excel](https://github.com/SpartnerNL/Laravel-Excel)** (`maatwebsite/excel`) handles Excel and CSV import/export. It supports chunked reading for large files, queued exports, multiple sheets, and formatting. Nearly every business application needs to export data to a spreadsheet at some point. This package handles the complexity of the Excel format so you do not have to.

### Feature Flags

**[Laravel Pennant](https://laravel.com/docs/pennant)** (`laravel/pennant`) is Laravel's official feature flag system. It lets you roll out features gradually, target specific users or segments, and toggle functionality without deploying new code. Feature flags are a modern deployment best practice covered in the [Deployments](/books/clean-code-in-laravel/deployments) chapter. Pennant gives you a first-party solution that integrates with Laravel's authentication and caching.

## Evaluating a Package

When you find a package that solves your problem, evaluate it before installing:

**Check the maintenance pulse.** Look at the commit history, the release cadence, and the issue tracker. A healthy package has regular commits, prompt responses to issues, and tagged releases. A package with 10,000 stars but no commits in eighteen months is a risk.

**Read the source code.** You do not need to read every line, but skim the main classes. Is the code clean? Does it follow Laravel conventions? Does it use modern PHP features? The quality of the source code tells you a lot about the quality of the maintainer's work.

**Check the test suite.** A well-tested package is a package you can trust. If there are no tests, the maintainer is shipping code they cannot verify. That should give you pause.

**Look at the upgrade path.** Read the changelog. Does the package follow semantic versioning? Are major version upgrades well-documented with migration guides? A package that breaks its API without warning will break your application without warning.

**Consider the bus factor.** Is this package maintained by one person or a team? If the sole maintainer disappears, what happens? Packages maintained by organizations like [Spatie](https://spatie.be/), or official Laravel packages, have a lower risk of abandonment.

## When to Build Your Own

Sometimes the right answer is to not install a package. Build your own when:

- **The integration is simple.** A single API call, a slug generator, a string formatter. If you can write it in under a hundred lines and it has no tricky edge cases, own the code.
- **The package does too much.** If you need one feature from a package that provides fifty, you are carrying dead weight. Extract the pattern you need and implement it yourself.
- **The package forces its architecture on you.** Some packages require you to extend their base classes, use their traits everywhere, or restructure your application around their conventions. If a package fights your architecture, it is the wrong package.
- **You need full control over the behavior.** When the feature is core to your business logic — your pricing engine, your matching algorithm, your scoring system — you should own every line. A package update should never be able to change how your business works.

The line between "use a package" and "build it yourself" is not always clear. But the default should not be "find a package." The default should be "understand the problem, evaluate the options, and make a deliberate choice."

## Keeping Dependencies Healthy

Installing a package is not a one-time decision. Dependencies need ongoing care:

**Run `composer outdated` regularly.** This shows you which packages have new versions available. Small, frequent updates are safer than infrequent large jumps across multiple major versions.

**Update dependencies in isolation.** When you update a package, update only that package and run your test suite. If something breaks, you know exactly which update caused it. Do not update ten packages at once and then spend a day figuring out which one broke your tests.

**Remove packages you no longer use.** Dead dependencies still consume upgrade budget. If you installed a package six months ago for a feature that was later removed, run `composer remove` and clean it out.

**Pin to stable versions.** Avoid `dev-main` or `*` version constraints in production. Use caret (`^`) constraints that allow patch and minor updates but not major breaking changes.

## Summary

- Every package is a dependency. Every dependency is a bet on someone else's code, maintenance, and priorities. Make those bets deliberately.
- Before installing a package, ask: does Laravel already solve this? Can I build it in a reasonable amount of time? Is this package well-maintained?
- Simple integrations — API calls, basic string operations, straightforward data transformations — often do not need a package. Laravel's HTTP client, collection methods, and helper functions cover more ground than you think.
- Some packages are worth the dependency. [Debugbar](https://github.com/fruitcake/laravel-debugbar) for development visibility, [Larastan](https://github.com/larastan/larastan) for static analysis, [Laravel Data](https://spatie.be/docs/laravel-data) for typed data objects, [Laravel Permission](https://github.com/spatie/laravel-permission) for access control, [Laravel Backup](https://github.com/spatie/laravel-backup) for disaster recovery, and [Horizon](https://laravel.com/docs/horizon) for queue monitoring have all earned their place.
- Evaluate packages rigorously: check maintenance activity, read the source code, verify the test suite exists, and consider the bus factor.
- Build your own when the integration is simple, the package does too much, or the feature is core to your business logic.
- Maintain your dependencies actively. Run `composer outdated` regularly, update in isolation, and remove packages you no longer use.

## References

- [Composer Documentation](https://getcomposer.org/doc/) — Composer
- [Laravel HTTP Client](https://laravel.com/docs/http-client) — Laravel Documentation
- [Spatie Open Source](https://spatie.be/open-source) — Spatie
- [Laravel Debugbar](https://github.com/fruitcake/laravel-debugbar) — Fruitcake
- [Larastan](https://github.com/larastan/larastan) — Larastan
- [IDE Helper](https://github.com/barryvdh/laravel-ide-helper) — Barry vd. Heuvel
- [Laravel Data](https://spatie.be/docs/laravel-data) — Spatie
- [Laravel Permission](https://spatie.be/docs/laravel-permission) — Spatie
- [Laravel Medialibrary](https://spatie.be/docs/laravel-medialibrary) — Spatie
- [Laravel Backup](https://spatie.be/docs/laravel-backup) — Spatie
- [Laravel Activitylog](https://spatie.be/docs/laravel-activitylog) — Spatie
- [Laravel Horizon](https://laravel.com/docs/horizon) — Laravel Documentation
- [Laravel Scout](https://laravel.com/docs/scout) — Laravel Documentation
- [Laravel Excel](https://docs.laravel-excel.com/) — Maatwebsite
- [Laravel Pennant](https://laravel.com/docs/pennant) — Laravel Documentation
- [The Dependency Inversion Principle](https://web.archive.org/web/20150905081103/http://www.objectmentor.com/resources/articles/dip.pdf) — Robert C. Martin
