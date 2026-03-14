You have spent this entire book learning how to write clean code — thin [controllers](/books/clean-code-in-laravel/controllers), focused [Actions](/books/clean-code-in-laravel/actions), typed [DTOs](/books/clean-code-in-laravel/data-transfer-objects), idempotent [Jobs](/books/clean-code-in-laravel/jobs), properly managed [queue workers](/books/clean-code-in-laravel/queue-workers). None of it matters if your code never reaches production. Or worse, if it reaches production broken.

Deployment is the moment where your clean code meets reality. A bad deployment turns a perfectly working application into a 500 error page. A missing migration, a forgotten cache clear, a worker running stale code — any of these can take your application down. And unlike a bug in your code, a broken deployment affects every single user at the same time.

This chapter covers how to deploy Laravel applications reliably — from manual deployments where you control every step, to tools like Deployer that automate the process, to managed platforms like Laravel Forge and Laravel Cloud that handle the infrastructure for you. We start with why deployment matters and what actually happens during one, then work our way up to the tools that make it painless.

## What Happens During a Deployment

A deployment is not just "put new code on server." It is a sequence of steps that must happen in a specific order. Skip a step or get the order wrong, and your application breaks.

Here is the full sequence:

### 1. Pull the Latest Code

```bash
git pull origin main
```

This is the obvious step. But even here, things can go wrong. If you have uncommitted changes on the server (you should not), Git will refuse to pull. If you are on the wrong branch, you deploy the wrong code. Always deploy from a clean state.

### 2. Install PHP Dependencies

```bash
composer install --no-dev --optimize-autoloader --no-interaction
```

Three flags matter here:

- **`--no-dev`** — skips packages in `require-dev`. You do not need Pest, Telescope's debug tools, or IDE helpers in production. They waste disk space and can expose debugging endpoints.
- **`--optimize-autoloader`** — converts PSR-4 autoloading to a classmap. Normally, the autoloader checks the filesystem to find class files at runtime. This flag pre-builds a complete map of every class and its file, eliminating filesystem lookups. The difference is measurable on applications with hundreds of classes.
- **`--no-interaction`** — prevents Composer from asking interactive questions. You want your deployment to run unattended.

### 3. Install and Build Frontend Assets

```bash
npm ci && npm run build
```

Use `npm ci`, not `npm install`. `npm ci` does a clean install from `package-lock.json` — it deletes `node_modules` first and installs exact versions. `npm install` can modify the lock file if versions have drifted, which is not what you want in production.

`npm run build` compiles production-optimized assets via Vite — minified, tree-shaken, with content hashes for cache busting. Never run `npm run dev` on a production server — it starts a development server, not a build.

### 4. Run Database Migrations

```bash
php artisan migrate --force
```

The `--force` flag is mandatory. Laravel refuses to run migrations in production without it — a safety mechanism to prevent accidental schema changes. If your deployment script omits `--force`, migrations are silently skipped. Your code expects new columns that do not exist, and your application crashes.

### 5. Cache Configuration, Routes, Views, and Events

```bash
php artisan optimize
```

This single command caches everything at once:

- **`config:cache`** — combines all configuration files into a single cached file. This eliminates dozens of filesystem reads on every request. **Critical side effect:** once configuration is cached, the `.env` file is no longer loaded. All `env()` calls outside of config files return `null`. This is why you should never call `env()` directly in your application code — always use `config()` instead.
- **`route:cache`** — serializes all route registrations into a single method call. Essential for applications with hundreds of routes.
- **`view:cache`** — precompiles all Blade templates. Without this, views compile on demand during the first request that uses them.
- **`event:cache`** — caches auto-discovered event-to-listener mappings, preventing a scan of all listener classes on every request.

### 6. Restart Queue Workers

```bash
php artisan queue:restart
```

Or, if you are using [Horizon](/books/clean-code-in-laravel/queue-workers#laravel-horizon):

```bash
php artisan horizon:terminate
```

[Queue workers](/books/clean-code-in-laravel/queue-workers) boot the Laravel application once and keep it in memory. After you deploy new code, workers are still running the old version. If you skip this step, workers process Jobs with stale logic — charging the wrong amount, sending the wrong email, or crashing on a class that was renamed.

### 7. Reload PHP-FPM

```bash
sudo service php8.4-fpm reload
```

PHP-FPM caches compiled bytecode via OPcache. A reload tells it to invalidate the cache and pick up the new files.

### The Complete Script

Putting it all together:

```bash
cd /var/www/app

git pull origin main

composer install --no-dev --optimize-autoloader --no-interaction

npm ci && npm run build

php artisan migrate --force

php artisan optimize

php artisan queue:restart  # or: php artisan horizon:terminate

sudo service php8.4-fpm reload
```

This is the baseline. Every deployment tool — Deployer, Forge, Cloud — automates some or all of these steps. Understanding them is important because when something goes wrong, you need to know which step failed and why.

## Versioning Your Releases

Before diving into deployment tools, there is a practice that makes deployments safer: tagging your releases with version numbers. A version tag is a snapshot — a named bookmark that says "this exact code was deployed to production at this point in time."

### Why Version Tags Matter

Without tags, your deployment history looks like this:

```
commit a1b2c3d — Fix payment bug
commit e4f5g6h — Add user notifications
commit i7j8k9l — Update order flow
```

Which of these is currently running in production? Which one did you deploy last Tuesday? If the latest deploy broke something, which commit do you roll back to? Without tags, you are guessing based on timestamps and commit messages.

With tags, it looks like this:

```
v1.3.0 — commit a1b2c3d — Fix payment bug
v1.2.1 — commit e4f5g6h — Add user notifications
v1.2.0 — commit i7j8k9l — Update order flow
```

Now every production release has a name. Rolling back means `git checkout v1.2.1`. Debugging means "this bug was introduced in v1.3.0." Your CI/CD pipeline can trigger deployments on tag pushes. Your team has a shared vocabulary: "we deployed v1.3.0" is unambiguous.

### Semantic Versioning

The convention is [Semantic Versioning](https://semver.org/) — `MAJOR.MINOR.PATCH`:

- **PATCH** (1.0.0 → 1.0.1) — bug fixes, trivial updates. Nothing changes for consumers of your application.
- **MINOR** (1.0.1 → 1.1.0) — new features that are backward-compatible. Existing functionality is unchanged.
- **MAJOR** (1.1.0 → 2.0.0) — breaking changes. Something that worked before no longer works the same way.

For most web applications (as opposed to libraries), the distinction between minor and major is less critical. The important thing is that each release has a unique, incrementing version number that you can reference later.

### Creating Tags

Use annotated tags — they store the tagger name, date, and a message:

```bash
# Create a tag for the current commit
git tag -a v1.3.0 -m "Add payment processing and notification improvements"

# Push the tag to the remote
git push origin v1.3.0
```

Annotated tags are full Git objects. They have their own history, and `git show v1.3.0` displays who created the tag, when, and why:

```
$ git show v1.3.0
tag v1.3.0
Tagger: Ahmad Mayahi <ahmad@mayahi.net>
Date:   Fri Mar 14 10:30:00 2026 +0300

Add payment processing and notification improvements

commit a1b2c3d4e5f6...
```

### Useful Tag Commands

```bash
git tag                          # List all tags
git tag -l "v1.3.*"              # List tags matching a pattern
git tag -a v1.0.0 9fceb02 -m "" # Tag a past commit
git push origin --tags           # Push all tags
git tag -d v1.0.0                # Delete a local tag
git push origin --delete v1.0.0  # Delete a remote tag
```

### Tags in CI/CD Workflows

Most CI/CD platforms can trigger deployments when a tag matching a pattern is pushed:

```yaml
# GitHub Actions — deploy on version tags
on:
  push:
    tags:
      - 'v*'
```

This separates "merging code" from "deploying to production." You merge features into `main` throughout the day. When you are ready to deploy, you create a tag. The pipeline picks it up and deploys. This gives you a deliberate, auditable release process.

## Zero-Downtime Deployment

The deployment script above has a problem: while it runs, your application is in a broken state. `git pull` updates some files but not others. `composer install` might take 30 seconds during which autoloading is inconsistent. A user who hits your application mid-deployment sees errors.

For a personal project, this is fine. For a production application with users, it is not.

Zero-downtime deployment solves this by building the new release in a separate directory and only switching to it after everything is ready. The switch itself is an atomic filesystem operation — a symlink swap — that takes microseconds. There is no window where the application is partially updated.

### How It Works

Instead of deploying into a single directory, a zero-downtime deployment uses this structure:

```
/var/www/app/
├── current -> releases/20260314_103000   ← Symlink (web server points here)
├── releases/
│   ├── 20260312_090000/                   ← Old release
│   ├── 20260313_140000/                   ← Previous release
│   └── 20260314_103000/                   ← Current active release
│       ├── .env -> ../../shared/.env
│       └── storage -> ../../shared/storage
└── shared/
    ├── .env
    └── storage/
```

The web server's document root points to `current/public`. Each deployment creates a new release directory, builds everything inside it, and only switches the `current` symlink after all steps complete. If the deployment fails at any point — Composer errors, a failed migration, a broken build — the previous release remains active. Users never see a thing.

The `shared` directory holds files that persist across releases: the `.env` file and the `storage` directory (logs, uploaded files, cache). These are symlinked into each release so they are never lost during deployment.

## Deployer

[Deployer](https://deployer.org/) is an open-source PHP deployment tool that implements zero-downtime deployments out of the box. It connects to your server via SSH, creates the release directory structure, runs all deployment tasks, and swaps the symlink. It comes with a built-in Laravel recipe that knows exactly which Artisan commands to run and in what order.

### Why Use Deployer

Deployer gives you everything the manual script does, plus:

- **Zero-downtime** — symlink-based releases, no user-facing interruption
- **Rollback** — `dep rollback` instantly switches back to the previous release
- **Lock mechanism** — prevents two people from deploying at the same time
- **Cleanup** — automatically removes old releases (keeps the last 5 by default)
- **Multi-server** — deploy to staging and production from the same config

### Installing Deployer

```bash
composer require deployer/deployer --dev
dep init
```

Select the Laravel recipe when prompted. This creates a `deploy.php` configuration file.

### Configuring Deployer

```php
<?php

namespace Deployer;

require 'recipe/laravel.php';

set('application', 'my-app');
set('repository', 'git@github.com:user/my-app.git');
set('keep_releases', 5);

// Files and directories shared across all releases
set('shared_files', ['.env']);
set('shared_dirs', ['storage']);

// Directories that must be writable
set('writable_dirs', [
    'bootstrap/cache',
    'storage',
    'storage/app',
    'storage/app/public',
    'storage/framework',
    'storage/framework/cache',
    'storage/framework/sessions',
    'storage/framework/views',
    'storage/logs',
]);

// Server configuration
host('production')
    ->set('remote_user', 'forge')
    ->set('hostname', 'example.com')
    ->set('deploy_path', '/var/www/example.com');

host('staging')
    ->set('remote_user', 'forge')
    ->set('hostname', 'staging.example.com')
    ->set('deploy_path', '/var/www/staging.example.com');

// Build frontend assets after code is pulled
task('build', function (): void {
    cd('{{release_path}}');
    run('npm ci');
    run('npm run build');
});

after('deploy:update_code', 'build');

// Unlock deployment on failure so the next deploy is not blocked
after('deploy:failed', 'deploy:unlock');
```

### The Deployment Pipeline

When you run `dep deploy production`, Deployer executes these steps automatically:

1. **Lock** — creates a lock file to prevent concurrent deployments
2. **Create release directory** — e.g., `releases/20260314_103000/`
3. **Clone repository** — pulls your code into the new release
4. **Symlink shared files** — links `.env` and `storage` from the shared directory
5. **Set permissions** — makes `storage` and `bootstrap/cache` writable
6. **Install dependencies** — `composer install --no-dev --optimize-autoloader`
7. **Run custom tasks** — your `build` task for frontend assets
8. **Run Artisan commands** — `storage:link`, `config:cache`, `route:cache`, `view:cache`, `event:cache`, `migrate`
9. **Swap symlink** — atomically switches `current` to the new release
10. **Cleanup** — removes old releases beyond `keep_releases`
11. **Unlock** — removes the lock file

Steps 1 through 8 happen in the new release directory. The old release is still serving traffic. Only step 9 — the symlink swap — makes the new code live. This is what makes it zero-downtime.

### Deploying and Rolling Back

```bash
dep deploy production      # Deploy to production
dep deploy staging         # Deploy to staging
dep rollback               # Revert to the previous release
dep releases               # List all releases on the server
dep ssh production         # SSH into the production server
```

Rollback is instantaneous — it switches the `current` symlink to the previous release directory. No rebuild, no reinstall, no migrations (you handle migration rollbacks separately if needed). This is why keeping multiple releases on disk is valuable — any of them can become the active release in microseconds.

### When to Use Deployer

Deployer is the right choice when:

- You manage your own servers (VPS, dedicated, bare metal)
- You want zero-downtime deployments without paying for a managed service
- You need multi-server deployment (Deployer can deploy to multiple hosts in parallel)
- Your team has the DevOps knowledge to maintain servers

Deployer does not provision servers. You need to set up PHP, Nginx, MySQL, Redis, Supervisor, and SSL yourself. It automates the deployment step only — everything between `git push` and "code is live."

## Laravel Forge

Setting up a server from scratch is tedious. You need to install PHP with the right extensions, configure Nginx with the right rewrite rules, set up MySQL or PostgreSQL, install Redis, configure Supervisor for queue workers, obtain SSL certificates, set up firewall rules, configure log rotation, and keep everything updated with security patches. Deployer automates the deployment, but it assumes you already have a working server.

[Laravel Forge](https://forge.laravel.com/) solves this. It provisions and manages servers for you — one click and you have a production-ready server with PHP, Nginx, a database, Redis, Supervisor, SSL, and automatic security updates. It handles everything between "I need a server" and "my code is deployed."

### What Forge Does

Forge is a server provisioning and management platform. When you create a server through Forge, it:

- Provisions a VPS on your preferred provider (DigitalOcean, Hetzner, AWS, Vultr, Linode, or Forge's own VPS)
- Installs and configures PHP (with all required extensions), Nginx, MySQL/PostgreSQL/MariaDB, Redis, Node.js, and Supervisor
- Sets up automatic security updates
- Configures a firewall (UFW)
- Sets up log rotation
- Installs Composer

You maintain full SSH access. Forge does not lock you in — it provisions standard Ubuntu servers that you can manage manually if needed.

### Deployments on Forge

Forge provides a deploy script that runs when you push to your configured branch (GitHub, GitLab, or Bitbucket). The default script looks like this:

```bash
cd $FORGE_SITE_PATH

git pull origin $FORGE_SITE_BRANCH

$FORGE_COMPOSER install --no-interaction --prefer-dist --optimize-autoloader --no-dev

( flock -w 10 9 || exit 1
    echo 'Restarting FPM...'
    sudo -S service $FORGE_PHP_FPM reload ) 9>/tmp/fpmlock

$FORGE_PHP artisan migrate --force
```

You can customize this script directly in the Forge dashboard. Forge provides environment variables (`$FORGE_SITE_PATH`, `$FORGE_PHP`, `$FORGE_COMPOSER`, etc.) so your script works regardless of the PHP version or server layout.

### Zero-Downtime on Forge

Forge now supports native zero-downtime deployments using the same symlink-based approach as Deployer. Enable it when creating a new site, and Forge provides three macros for your deploy script:

```bash
$CREATE_RELEASE()   # Clone code into a new release directory
# Forge automatically cd's into the release directory after this

$FORGE_COMPOSER install --no-interaction --prefer-dist --optimize-autoloader --no-dev

npm ci && npm run build

$FORGE_PHP artisan migrate --force
$FORGE_PHP artisan optimize

$ACTIVATE_RELEASE()   # Swap the symlink to the new release
$RESTART_QUEUES()     # Restart queue workers / Horizon
```

### Queue Workers and Horizon

Forge manages Supervisor for you. You configure queue workers through the dashboard — specifying the connection, queue, number of processes, and options. Forge generates the Supervisor config, starts the workers, and restarts them automatically if they crash.

For Horizon, Forge detects that Horizon is installed and automatically appends `horizon:terminate` to your deployment script.

### SSL Certificates

Forge integrates with Let's Encrypt. One click and your site has a free SSL certificate that auto-renews every week. No manual renewal cron, no certificate management.

### Push to Deploy

By default, pushing to your configured branch triggers an automatic deployment. Forge sets up a webhook with your Git provider. You can also trigger deployments manually from the dashboard, via the Forge CLI, or through a custom webhook URL.

### Pricing

Forge starts at $12/month for a single server (Hobby plan) and $19/month for unlimited servers (Growth plan). The server itself is billed separately by your cloud provider. With Forge's own VPS offering, you can get a production-ready server starting at $6/month.

### When to Use Forge

Forge is the right choice when:

- You want server provisioning without manual setup
- You need full server access (SSH, root, custom software)
- You want to pick your own hosting provider and control costs
- You are managing multiple sites or client projects
- You want predictable, fixed monthly costs (Forge fee + server cost)
- You need to run non-Laravel applications on the same server

## Laravel Cloud

Forge gives you a managed server. You still think about servers — how much RAM, how many CPU cores, which region, how many workers to run. You still SSH in occasionally to debug something, check logs, or update a package.

[Laravel Cloud](https://cloud.laravel.com/) removes the server entirely. You push code, and Cloud handles everything: provisioning, scaling, databases, caching, queue workers, SSL, backups. There is no server to SSH into. There is no Nginx config to tweak. There is no Supervisor to manage. You write Laravel code and Cloud runs it.

### How Cloud Works

Cloud is a Platform-as-a-Service built exclusively for Laravel by Laravel's creators. When you connect your repository and deploy:

1. Cloud builds a Docker image from your code (running `composer install`, `npm run build`, `php artisan optimize`)
2. Cloud deploys the image to its infrastructure
3. Cloud runs your migrations (`php artisan migrate --force`)
4. Cloud routes traffic to the new version
5. Cloud automatically restarts queue workers with the new code — no `queue:restart` needed

The filesystem is ephemeral — files do not persist across deployments or requests. You use Redis (Valkey) for caching and object storage for uploaded files, both provisioned with one click through Cloud's dashboard.

### Auto-Scaling

This is Cloud's most compelling feature. Instead of guessing how many workers or server resources you need, you set boundaries and Cloud scales within them:

- **App clusters** scale horizontally — more replicas handle more HTTP traffic
- **Worker clusters** scale independently — more replicas process more Jobs
- **Queue clusters** (developer preview) auto-scale based on Job latency and queue depth

You configure minimum and maximum replicas, and Cloud adds or removes instances based on CPU and memory thresholds. During a traffic spike, Cloud spins up more replicas. At 3 AM when nobody is using your application, Cloud can scale down to a single instance — or even hibernate to zero on Flex compute, eliminating charges entirely.

### Hibernation

On Flex compute instances, Cloud can scale your application to zero when it is idle. No requests, no compute charges. When a request arrives, Cloud wakes your application in 5–20 seconds and starts serving traffic. This makes Cloud extremely cost-effective for applications with variable traffic — a side project that gets 100 visitors a day does not cost the same as a SaaS application handling 10,000 requests per minute.

The trade-off is that first request after hibernation is slow. For production applications with consistent traffic, use Pro or Dedicated compute classes that do not hibernate.

### Managed Services

Cloud provides one-click provisioning for everything your Laravel application needs:

- **Databases** — managed MySQL (from $5.50/month) or serverless PostgreSQL via Neon (pay-per-use, hibernates when idle)
- **Cache** — Valkey (Redis-compatible), from $5/month
- **Object storage** — S3-compatible, for uploaded files
- **Queue workers** — configured as background processes, automatically restarted on deployment

Environment variables for these services are automatically injected. You do not configure database credentials manually — Cloud provisions the database and wires it up.

### Pricing

Cloud uses usage-based pricing on top of a plan fee:

- **Starter** — $0/month base + usage. Single instance, no auto-scaling. Good for side projects.
- **Growth** — $20/month + usage. Up to 10 replicas, auto-scaling. Good for production applications.
- **Business** — $200/month + usage. Up to 20 replicas, team features (RBAC, SAML). Good for organizations.

The usage cost depends on compute class, instance count, and uptime. A minimal Flex instance starts at $4/month. A production Pro instance starts at $20/month. Database and cache costs are separate.

### When to Use Cloud

Cloud is the right choice when:

- You want zero infrastructure management
- Your traffic is variable or spiky (auto-scaling saves money)
- You are building a SaaS, startup, or MVP where time-to-market matters more than infrastructure control
- Your team does not have DevOps expertise
- You want managed databases, caching, and backups without configuration
- You want automatic queue worker management

## Forge vs. Cloud

Both are built by the Laravel team. Both deploy Laravel applications. They serve different needs:

| | Forge | Cloud |
|---|---|---|
| **What you manage** | Servers (with Forge's help) | Nothing — Cloud is the server |
| **SSH access** | Full root access | No SSH to infrastructure |
| **Server provider** | You choose (DigitalOcean, Hetzner, AWS, etc.) | Cloud IS the provider |
| **Scaling** | Manual — resize VPS or add servers | Automatic — horizontal auto-scaling |
| **Pricing model** | Fixed — Forge fee + server cost | Usage-based — pay for what you use |
| **Queue workers** | You configure Supervisor | Automatically managed |
| **Databases** | You manage (on server or external) | One-click managed databases |
| **Non-Laravel apps** | Supported (any PHP, Node.js, etc.) | Laravel only |
| **Multiple sites per server** | Yes | No — each app gets its own environment |
| **Ideal for** | Agencies, predictable workloads, full control | Startups, variable traffic, no-ops teams |

If you want control and predictability, use Forge. If you want simplicity and auto-scaling, use Cloud. Both are excellent — the choice depends on your team, your traffic patterns, and how much infrastructure you want to think about.

## Common Deployment Mistakes

- **Forgetting to restart queue workers.** This is the most common deployment mistake. Workers boot the application once and keep it in memory — they do not see new code until restarted. Always include `queue:restart` or `horizon:terminate` in your deployment script. See the [Queue Workers](/books/clean-code-in-laravel/queue-workers) chapter for details.
- **Running `npm run dev` instead of `npm run build`.** `dev` starts a development server. `build` outputs optimized production assets. Using `dev` in production means serving unminified, unoptimized JavaScript — or worse, trying to proxy requests to a Vite server that does not exist.
- **Deploying with `APP_DEBUG=true`.** With debug mode enabled, Laravel exposes full stack traces, environment variables, and database credentials to anyone who triggers an error. Always set `APP_DEBUG=false` in production.
- **Forgetting `--force` on `migrate`.** Laravel refuses to run migrations in production without `--force`. Your deployment script silently skips migrations, and your code expects database columns that do not exist.
- **Forgetting `--no-dev` on `composer install`.** Installs testing frameworks, debug bars, and development tools in production. Wastes disk space and can expose debugging endpoints.
- **Calling `env()` outside of config files.** Once you run `config:cache`, the `.env` file is no longer loaded. Any `env()` call outside of a config file returns `null`. Use `config()` everywhere in your application code.
- **Running migrations that break running workers.** If a migration renames or drops a column, workers still running old code will crash when they try to use the old column name. For destructive migrations, pause workers before migrating and restart them after.
- **Not caching config and routes.** Without `php artisan optimize`, every request reads and parses all config files and re-registers all routes. This is measurably slower — especially on applications with many routes.
- **Using `npm install` instead of `npm ci`.** `npm install` can modify `package-lock.json` if versions have drifted. `npm ci` does a clean install from the lock file — deterministic and reproducible.
- **Missing storage permissions.** Laravel requires write access to `storage/` and `bootstrap/cache/`. If permissions are wrong, logging, caching, and session storage fail silently or throw 500 errors.

## The Deployment Checklist

1. **Tag your release** — `git tag -a v1.3.0 -m "Release description"` and push the tag
2. **Pull code** — `git pull origin main`
3. **Install PHP dependencies** — `composer install --no-dev --optimize-autoloader --no-interaction`
4. **Build frontend assets** — `npm ci && npm run build`
5. **Run migrations** — `php artisan migrate --force`
6. **Cache everything** — `php artisan optimize`
7. **Restart queue workers** — `php artisan queue:restart` or `php artisan horizon:terminate`
8. **Reload PHP-FPM** — `sudo service php8.4-fpm reload`
9. **Verify** — check the application in a browser, check the logs, confirm workers are processing Jobs

For zero-downtime deployments, use Deployer or Forge's native zero-downtime feature. For managed infrastructure, use Forge or Cloud. For full automation, connect your CI/CD pipeline to deploy on tag pushes.

## Summary

- Deployment is a specific sequence of steps that must happen in order: pull code, install dependencies, build assets, run migrations, cache config/routes/views, restart workers, reload PHP-FPM.
- Tag every release with a semantic version (`v1.3.0`). Tags give you rollback targets, an audit trail, and CI/CD triggers. Use annotated tags (`git tag -a`) for a complete history.
- Zero-downtime deployment builds the new release in a separate directory and only switches to it after everything is ready. The switch is an atomic symlink swap — users never see a broken state.
- [Deployer](https://deployer.org/) automates zero-downtime deployments for self-managed servers. It creates release directories, symlinks shared files, runs all Artisan commands, and swaps the symlink. `dep rollback` reverts instantly.
- [Laravel Forge](https://forge.laravel.com/) provisions and manages servers — PHP, Nginx, databases, Redis, Supervisor, SSL, and security updates. It handles deployments, queue workers, and SSL certificates. You maintain full SSH access.
- [Laravel Cloud](https://cloud.laravel.com/) removes the server entirely. You push code and Cloud handles everything — building, deploying, scaling, databases, caching, and queue workers. Auto-scaling adjusts resources based on traffic. Hibernation scales to zero when idle.
- Choose Forge when you want control, predictable costs, and the ability to run multiple sites on one server. Choose Cloud when you want zero infrastructure management, auto-scaling, and usage-based pricing.
- Never skip `queue:restart` or `horizon:terminate` in your deployment script. Workers run old code until restarted — this is the single most common deployment mistake.
- Never call `env()` outside of config files. Once `config:cache` runs, `.env` is not loaded and `env()` returns `null`.
- Use `--no-dev` and `--optimize-autoloader` with Composer. Use `npm ci` instead of `npm install`. Use `--force` with `migrate`. These are not optional in production.

## References

- [Deployment](https://laravel.com/docs/deployment) — Laravel Documentation
- [Forge](https://forge.laravel.com/) — Laravel Forge
- [Cloud](https://cloud.laravel.com/) — Laravel Cloud
- [Deployer](https://deployer.org/) — Deployer
- [Deployer Laravel Recipe](https://deployer.org/docs/7.x/recipe/laravel) — Deployer Documentation
- [Semantic Versioning](https://semver.org/) — SemVer
- [Git Basics — Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging) — Git Documentation
- [Deploy Your Laravel App From Scratch](https://lorisleiva.com/deploy-your-laravel-app-from-scratch) — Loris Leiva
- [How to Deploy Laravel: Zero-Downtime, Build Pipelines, and Best Practices](https://www.deployhq.com/blog/how-to-deploy-laravel-zero-downtime-build-pipelines-and-best-practices) — DeployHQ
