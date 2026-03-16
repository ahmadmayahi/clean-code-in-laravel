Most books put code style at the end, as an afterthought. We put it here, in Chapter 3, because code style is not a finishing touch - it is a foundation. If your team cannot agree on how code looks, every pull request becomes a formatting debate instead of a logic review.

Taylor Otwell settled this for the Laravel community. Laravel uses its own opinionated coding style (based on [PER Coding Style](https://www.php-fig.org/per/coding-style/)), and the official tool for enforcing it is [Laravel Pint](https://laravel.com/docs/pint).

## Laravel Pint

[Laravel Pint](https://laravel.com/docs/pint) is a zero-configuration code style fixer built on top of [PHP-CS-Fixer](https://github.com/PHP-CS-Fixer/PHP-CS-Fixer). It ships with every new Laravel application. Out of the box, Pint uses the `laravel` preset, which is Taylor's preferred style. You do not need a configuration file. You do not need to debate tabs versus spaces. You run Pint, and your code looks like Laravel code.

To fix the style of every PHP file in your project, run Pint with no arguments:

```bash
./vendor/bin/pint
```

If you want to see what Pint would change without actually modifying any files, use the `--test` flag. This is useful in CI pipelines where you want to fail a build on style violations rather than auto-fix them:

```bash
./vendor/bin/pint --test
```

You can also target specific files or directories instead of scanning the entire project:

```bash
./vendor/bin/pint app/Models
./vendor/bin/pint app/Http/Controllers/UserController.php
```

If you need to customize rules, create a `pint.json` file in your project root:

```json
{
    "preset": "laravel",
    "rules": {
        "concat_space": {
            "spacing": "one"
        },
        "ordered_imports": {
            "sort_algorithm": "length"
        }
    }
}
```

The best practice is to run Pint automatically. Add it to your CI pipeline, your pre-commit hooks, or your editor's save action. Code style should never be a manual task.

## Laravel Coding Style

Do not spend time debating which coding style to follow. Do not schedule meetings about brace placement, ternary usage, or whether to use string interpolation. These discussions burn hours and produce no business value.

Instead, adopt the [Spatie Laravel & PHP guidelines](https://spatie.be/guidelines/laravel-php) as your non-negotiable standard. Spatie has been building and maintaining Laravel packages for years — their guidelines are battle-tested, comprehensive, and widely respected in the Laravel community. They cover everything from early returns and ternary usage to validation syntax, string formatting, and class structure.

Treat these guidelines the same way you treat your code formatter: configure once, follow always. When a new team member joins, point them to the guidelines. When a pull request debate starts about style, point to the guidelines. The answer is already written — there is nothing left to discuss.

Combine this with Laravel Pint (configured above) to enforce formatting automatically, and your team never has to think about code style again.

## Strict Types Declaration

By default, PHP is [weakly typed](https://stackoverflow.com/a/2690576/5528448). When a function expects an `int` and you pass the string `"3"`, PHP silently converts it and moves on. No error, no warning. This is by design — PHP was built for the web, where everything starts as a string in an HTTP request.

But that silent coercion can hide real bugs. Consider this:

```php
function calculateTotal(int $quantity, float $price): float
{
    return $quantity * $price;
}

// PHP silently converts "3" to 3 — no error, no warning
calculateTotal("3", 19.99); // Returns 59.97
```

It works, but what if someone passes `"three"` instead of `"3"`? You only discover the problem at runtime, and possibly in production.

Adding `declare(strict_types=1)` at the top of a file changes this behavior. PHP will no longer coerce types — if the types do not match, it throws a `TypeError` immediately:

```php
<?php

declare(strict_types=1);

function calculateTotal(int $quantity, float $price): float
{
    return $quantity * $price;
}

calculateTotal("3", 19.99);
// TypeError: Argument #1 ($quantity) must be of type int, string given
```

Now the bug is caught at the call site, not somewhere downstream where the corrupted value causes unexpected behavior.

Laravel itself does not use `declare(strict_types=1)` — the framework is designed to be flexible with type coercion. Many successful Laravel projects ship without it. Whether you adopt strict types is a team decision, not a universal best practice.

If your team does decide to use it, two things matter. First, be consistent — apply it across all your application files rather than mixing strict and non-strict files, which can lead to confusing behavior at the boundaries. Second, if you are adding it to an existing codebase, introduce it gradually — file by file, with tests covering each change — rather than turning it on everywhere at once.

For more context, see the [PHP documentation on strict typing](https://www.php.net/manual/en/language.types.declarations.php#language.types.declarations.strict) and the [Spatie guidelines](https://spatie.be/guidelines/laravel-php#content-typed-properties).

## Static Analysis

Static analysis means checking your code for bugs without running it. Instead of waiting for something to break in production, a static analysis tool reads your files, follows how data moves through your methods, and tells you about problems before they happen. Think of it as a spell-checker for your logic.

[Type hints](https://www.php.net/manual/en/language.types.declarations.php) are good - they tell PHP what types to expect. Static analysis takes that further. [PHPStan](https://phpstan.org/) reads your entire codebase and catches bugs that tests might miss - calling a method that does not exist, passing the wrong type to a function, or using a variable that might be null. [Larastan](https://github.com/larastan/larastan) is a PHPStan extension that understands Laravel's magic - Eloquent models, facades, collections, and more.

Unlike Pint, Laravel does not ship with PHPStan or any configuration for it. You need to install it yourself:

```bash
composer require --dev larastan/larastan
```

Then create a `phpstan.neon.dist` configuration file in your project root:

```neon
includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    paths:
        - app/
    level: 6
    checkMissingIterableValueType: false
```

You will notice PHPStan uses two possible config filenames: `phpstan.neon.dist` and `phpstan.neon`. The `.dist` file is the one you commit to your repository — it holds the shared configuration that everyone on your team uses. The `phpstan.neon` file (without `.dist`) is for local overrides. If PHPStan finds both, it uses `phpstan.neon` and ignores the `.dist` file. This way, a developer can temporarily lower the level or ignore a path on their machine without changing the committed config. Add `phpstan.neon` to your `.gitignore` so local overrides stay local.

PHPStan has levels from 0 (loose) to 9 (strictest). Start at level 5 or 6 and work your way up:

| Level | What It Checks |
|---|---|
| 0 | Basic checks: unknown classes, functions, methods |
| 1 | Possibly undefined variables |
| 2 | Unknown methods on `mixed` types |
| 3 | Return types |
| 4 | Dead code, unreachable statements |
| 5 | Argument types passed to methods |
| 6 | Missing type hints on properties |
| 7 | Union types handled correctly |
| 8 | Nullable types handled correctly |
| 9 | Mixed type is never used |

Run the analysis:

```bash
./vendor/bin/phpstan analyse
```

Larastan understands Laravel-specific patterns that would confuse plain PHPStan:

```php
// Larastan knows User::where() returns Builder<User>
// Larastan knows $user->posts returns HasMany<Post>
// Larastan knows config('app.name') returns string|null
// Larastan knows Route::get() accepts a closure or controller array
```

> A word of caution: Static analysis can be annoying if it is not set up well. If you crank the level too high on a codebase that is not ready, you will get hundreds of errors and your team will just start ignoring them. If you are adding PHPStan to an existing project, start at a low level, fix what you can, and use a [baseline](https://phpstan.org/user-guide/baseline) to track the rest. Do not add it to your CI/CD pipeline until your codebase passes cleanly - a pipeline that always fails is worse than no pipeline at all. It is also worth spending some time reading the [PHPStan documentation](https://phpstan.org/user-guide/getting-started) to understand how rules and levels work before you roll it out.

## Automated Refactoring with Rector

Pint fixes how your code looks. PHPStan tells you what is wrong. [Rector](https://getrector.com/) goes a step further - it rewrites your code for you. Rector reads your PHP files, applies a set of rules, and changes the code automatically. It can rename deprecated method calls, add type declarations, replace old patterns with modern ones, and even upgrade your code from one Laravel version to the next.

Install Rector with the Laravel extension:

```bash
composer require --dev driftingly/rector-laravel
```

Then create a `rector.php` configuration file in your project root. The simplest setup uses Rector's `LaravelSetProvider`, which automatically detects your Laravel version from `composer.json` and applies the matching rules:

```php
<?php

declare(strict_types=1);

use Rector\Config\RectorConfig;
use RectorLaravel\Set\LaravelSetProvider;

return RectorConfig::configure()
    ->withSetProviders(LaravelSetProvider::class)
    ->withComposerBased(laravel: true);
```

Run it in dry-run mode first to preview what would change:

```bash
./vendor/bin/rector --dry-run
```

When you are happy with the proposed changes, run it without the flag to apply them:

```bash
./vendor/bin/rector
```

### Targeted Rule Sets

Instead of applying everything at once, you can pick specific rule sets for the improvements you care about. For example, the `LARAVEL_IF_HELPERS` set replaces verbose conditional patterns with Laravel's concise helpers:

```php
// Before: manual if + abort
if (! $user->isAdmin()) {
    abort(403);
}

// After: Rector rewrites it to
abort_if(! $user->isAdmin(), 403);
```

The `LARAVEL_TYPE_DECLARATIONS` set adds return types and parameter types to your code based on how methods are actually used — a great way to prepare your codebase for higher PHPStan levels.

To use specific sets, reference them in your configuration:

```php
use RectorLaravel\Set\LaravelSetList;

return RectorConfig::configure()
    ->withSets([
        LaravelSetList::LARAVEL_IF_HELPERS,
        LaravelSetList::LARAVEL_TYPE_DECLARATIONS,
    ]);
```

If you are upgrading between Laravel versions, you can target a specific version level. Check the `LaravelLevelSetList` class in the [driftingly/rector-laravel](https://github.com/driftingly/rector-laravel) package for the constants available for your version:

```php
use RectorLaravel\Set\LaravelLevelSetList;

return RectorConfig::configure()
    ->withSets([
        LaravelLevelSetList::UP_TO_LARAVEL_110,
    ]);
```

Rector is powerful, but it is also opinionated — always review its changes before committing. Run your tests after every Rector pass. Treat it as a very fast junior developer: it does the tedious work, but you still review the pull request.

## Laravel Shift

If you want to take automated upgrades even further, [Laravel Shift](https://laravelshift.com/) is a paid service built specifically for upgrading Laravel applications between major versions. While Rector applies individual code transformations, Shift handles the full upgrade - configuration changes, dependency updates, namespace adjustments, removed features, and everything documented in Laravel's upgrade guide.

The workflow is simple: you sign in with your GitHub, Bitbucket, or GitLab account, point Shift at your repository, and it opens a pull request with atomic commits and detailed comments explaining every change it made. You review the PR, run your tests, and merge.

Shift supports upgrades all the way from Laravel 4.2 to the latest release. Individual shifts cost between $9 and $39, or you can subscribe to a [Shifty Plan](https://laravelshift.com/shifty-plans) starting at $99 per year for unlimited runs and automatic PRs whenever Laravel tags a new release.


> Shift works best on projects that follow Laravel conventions closely. If your codebase is heavily customized or has drifted far from the standard Laravel structure, you may need to fix some things by hand after the automated run. That said, even in those cases, Shift handles most of the repetitive work and lets you focus on the parts that actually need your attention.

## CI/CD Pipeline

Running these tools locally is great, but the real power comes when you enforce them automatically on every push and pull request. [GitHub Actions](https://github.com/features/actions) makes this straightforward.

Before setting up CI, consider adding Composer scripts so your team can run the same checks locally with a single command:

```json
{
    "scripts": {
        "lint": "./vendor/bin/pint --test",
        "fix": "./vendor/bin/pint",
        "analyse": "./vendor/bin/phpstan analyse",
        "refactor": "./vendor/bin/rector --dry-run",
        "check": [
            "@lint",
            "@analyse",
            "@refactor"
        ]
    }
}
```

Now `composer check` runs Pint, PHPStan, and Rector in one go — the same checks your CI pipeline will enforce.

### Automated Code Style Fixing with Pint

This workflow runs Pint on every push and pull request that touches PHP files, and automatically commits the fixes back to the branch:

```yaml
name: Fix PHP code style issues

on:
  push:
    branches: [main]
    paths:
      - '**.php'
  pull_request:
    branches: [main]
    paths:
      - '**.php'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: write

jobs:
  php-code-styling:
    runs-on: ubuntu-latest
    timeout-minutes: 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v6
        with:
          ref: ${{ github.head_ref }}

      - name: Fix PHP code style issues
        uses: aglipanci/laravel-pint-action@2.6

      - name: Commit changes
        uses: stefanzweifel/git-auto-commit-action@v7
        with:
          commit_message: Fix styling
```

Let us break down how this works:

- `paths: ['**.php']` - The workflow only triggers when PHP files change. There is no reason to run a PHP style fixer when you edit a README or a JavaScript file.
- `concurrency` - If you push twice in quick succession, the first run is cancelled. This saves runner minutes and prevents two runs from trying to commit style fixes at the same time.
- `permissions: contents: write` - The workflow needs write access to push the auto-fix commit back to the branch.
- `ref: ${{ github.head_ref }}` - On pull requests, this checks out the actual feature branch (not the merge commit), so the style fix commit lands on the correct branch.
- [`aglipanci/laravel-pint-action@2.6`](https://github.com/aglipanci/laravel-pint-action) - A community action that sets up PHP and runs Pint. It handles the PHP setup so you do not have to.
- [`stefanzweifel/git-auto-commit-action@v7`](https://github.com/stefanzweifel/git-auto-commit-action) - After Pint runs, this action checks if any files were modified. If Pint changed something, it commits and pushes the fixes automatically. If nothing changed, it does nothing.

The result: developers never need to argue about code style in reviews. Push your code, and CI formats it for you.

### Static Analysis with PHPStan

This workflow runs PHPStan to catch bugs and type errors:

```yaml
name: PHPStan

on:
  push:
    branches: [main]
    paths:
      - '**.php'
      - 'phpstan.neon.dist'
      - 'phpstan-baseline.neon'
      - '.github/workflows/phpstan.yml'
  pull_request:
    branches: [main]
    paths:
      - '**.php'
      - 'phpstan.neon.dist'
      - 'phpstan-baseline.neon'
      - '.github/workflows/phpstan.yml'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  phpstan:
    name: PHPStan
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v6

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          coverage: none

      - name: Install composer dependencies
        uses: ramsey/composer-install@v3

      - name: Run PHPStan
        run: ./vendor/bin/phpstan --error-format=github
```

Here is what each piece does:

- `paths` - Notice this workflow triggers on more than just PHP files. Changes to `phpstan.neon.dist` (the configuration), `phpstan-baseline.neon` (the baseline of ignored errors), and the workflow file itself all trigger a re-run. If you tighten your PHPStan rules, you want to know immediately if the codebase still passes.
- [`shivammathur/setup-php@v2`](https://github.com/shivammathur/setup-php) - Sets up the exact PHP version your project uses. The `coverage: none` option skips installing Xdebug, which speeds up the job since PHPStan does not need code coverage.
- [`ramsey/composer-install@v3`](https://github.com/ramsey/composer-install) - Installs your Composer dependencies with built-in caching. On subsequent runs, it restores the `vendor/` directory from cache, making the install step near-instant.
- `--error-format=github` - This is the key flag. Instead of printing errors to the console, PHPStan outputs them in GitHub's annotation format. This means errors appear as inline annotations directly on the pull request diff, right next to the line that caused the problem.

Unlike the Pint workflow, PHPStan does not auto-fix anything - it fails the build. This is intentional. A type mismatch might mean the code is wrong, or it might mean the type hint is wrong. Only the developer can decide which one to fix.

### Automated Refactoring with Rector

This workflow runs Rector in dry-run mode. It checks whether there are any refactoring rules that have not been applied yet. If there are, the build fails. The idea is that developers should run Rector locally, review what it changed, and commit on purpose — not have CI silently rewrite their code:

```yaml
name: Rector

on:
  push:
    branches: [main]
    paths:
      - '**.php'
      - 'rector.php'
      - '.github/workflows/rector.yml'
  pull_request:
    branches: [main]
    paths:
      - '**.php'
      - 'rector.php'
      - '.github/workflows/rector.yml'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  rector:
    name: Rector
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v6

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          coverage: none

      - name: Install composer dependencies
        uses: ramsey/composer-install@v3

      - name: Run Rector
        run: ./vendor/bin/rector --dry-run
```

Here is what each piece does:

- **`paths`** — Like the PHPStan workflow, this triggers on PHP files, the Rector configuration (`rector.php`), and the workflow file itself. Changing a rule set should immediately verify the codebase still passes.
- **`--dry-run`** — Rector checks the code and reports what it would change, but does not modify any files. If there are pending changes, the build fails.

The workflow uses the same [`shivammathur/setup-php`](https://github.com/shivammathur/setup-php) and [`ramsey/composer-install`](https://github.com/ramsey/composer-install) actions as the PHPStan workflow, keeping the setup consistent across all your CI jobs.

### Why Separate Workflows?

You might wonder why we do not combine Pint, PHPStan, and Rector into a single workflow. There are a few reasons:

1. **Different behaviors** — Pint auto-fixes and commits. PHPStan and Rector fail the build. Mixing these in one workflow creates confusing logic where part of the job succeeds and part fails.
2. **Independent triggers** — PHPStan triggers on `phpstan.neon.dist` changes, Rector triggers on `rector.php` changes, and Pint only needs PHP files. Keeping them separate means each workflow has clean, focused trigger rules.
3. **Clear feedback** — When a workflow fails, you immediately know what failed: style, analysis, or refactoring. A single combined workflow forces you to dig through logs to figure out which tool broke.

All three workflows use `timeout-minutes: 5` as a safety net. If something hangs, the job stops after five minutes instead of burning through your Actions quota.

With these tools in place, every chapter that follows will produce code that is not only well-structured but also clean, checked, and ready to ship. This is the foundation that makes everything else possible.

## Summary

- **Code style is a foundation, not a finishing touch.** Settle formatting debates once with [Laravel Pint](https://laravel.com/docs/pint), then automate it so nobody thinks about it again.
- **Adopt the [Spatie guidelines](https://spatie.be/guidelines/laravel-php) as your non-negotiable coding style.** Do not waste time in meetings debating style — the answers are already written.
- **Strict types are optional.** Laravel does not use them. If your team adopts them, be consistent and introduce them gradually.
- **Static analysis catches bugs without running code.** [PHPStan](https://phpstan.org/) with [Larastan](https://github.com/larastan/larastan) understands Laravel's magic and finds problems your tests might miss. Start at a low level and work your way up.
- **Automate refactoring with [Rector](https://getrector.com/).** It rewrites your code to use modern PHP and Laravel patterns. Use [Laravel Shift](https://laravelshift.com/) for full version upgrades.
- **Enforce everything in CI.** Separate workflows for Pint, PHPStan, and Rector give clear feedback, independent triggers, and clean separation of concerns.
