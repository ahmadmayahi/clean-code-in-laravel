Your database schema is the foundation of your application. Every controller, every model, every query sits on top of it. A poorly designed schema leads to slow queries, complex workarounds, and painful migrations. A well-designed schema makes everything else easier.

Yet database problems are the hardest to fix after the fact. You can refactor a controller in an afternoon. Restructuring a table with 50 million rows while the application is serving traffic is a different story entirely. The decisions you make in your migrations — column types, indexes, constraints — have consequences that compound over time.

This chapter covers the most common database mistakes in Laravel applications and how to avoid them: the N+1 problem that silently kills performance, indexing strategies that make queries fast, migration practices that keep your schema maintainable, and query patterns that scale.

## The N+1 Problem

The N+1 problem is the most common performance issue in Laravel applications. It happens when you load a collection of models and then access a relationship on each one:

```php
// Bad: N+1 — 1 query for orders + N queries for users
$orders = Order::all();
foreach ($orders as $order) {
    echo $order->user->name; // Each iteration triggers a query
}
```

If you have 100 orders, this executes 101 queries. With 1,000 orders, it is 1,001 queries. The code looks innocent — there are no explicit queries — but Eloquent silently loads the relationship on every iteration. This is called "lazy loading," and it is the default behavior.

The fix is eager loading:

```php
// Good: 2 queries total — 1 for orders + 1 for users
$orders = Order::with('user')->get();
foreach ($orders as $order) {
    echo $order->user->name; // No additional query
}
```

The `with('user')` call tells Eloquent to load all users in a single query upfront, then match them to their orders in memory. Two queries instead of 101.

### Preventing N+1 in Development

Laravel can throw an exception when an N+1 query is detected. Enable this in development:

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    Model::preventLazyLoading(! app()->isProduction());
}
```

This will throw a `LazyLoadingViolationException` whenever a relationship is accessed without being eager-loaded. Fix every violation before deploying. This single line of code prevents more performance problems than any amount of code review.

### Nested Eager Loading

Eager load nested relationships with dot notation:

```php
$orders = Order::with([
    'user',
    'items.product.category',
    'shippingAddress',
    'payments',
])->get();
```

Each dot-separated level triggers one additional query. The example above runs 5 queries regardless of how many orders are returned — far better than the hundreds or thousands that lazy loading would produce.

### Conditional Eager Loading

Load relationships only when needed:

```php
$orders = Order::query()
    ->with(['items.product'])
    ->when($request->has('include_user'), fn (Builder $q) => $q->with('user'))
    ->get();
```

### Eager Loading Counts

When you only need the count, not the full relationship:

```php
$users = User::withCount(['orders', 'reviews'])->get();

// $user->orders_count, $user->reviews_count — no extra queries
```

This adds a subquery to the original query instead of loading the entire relationship. Use it whenever you display counts in lists or dashboards.

### Limiting Eager Loaded Records

Laravel 12 supports limiting eagerly loaded records natively:

```php
$users = User::with([
    'posts' => fn (Builder $query) => $query->latest()->limit(5),
])->get();
```

Each user loads only their 5 most recent posts. Before Laravel 12, this required external packages.

## Indexing Strategy

Indexes make queries fast. Without them, the database scans every row to find matches — a "full table scan." With them, it jumps directly to the relevant rows using a data structure (typically a B-tree) that makes lookups logarithmic instead of linear.

A table with 1 million rows and no index on the `status` column requires scanning all 1 million rows for `WHERE status = 'pending'`. With an index, the same query touches a handful of rows. The difference between a 500ms query and a 2ms query is often a single index.

### When to Add Indexes

The most common question about indexes is *when* to add them. The answer comes from your queries, not your schema. Look at the queries your application actually runs — the `WHERE` clauses, the `ORDER BY` sorts, the `GROUP BY` aggregations — and index the columns they reference:

```php
// Migration
Schema::create('orders', function (Blueprint $table): void {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('status');
    $table->decimal('total', 10, 2);
    $table->timestamp('placed_at')->nullable();
    $table->timestamps();
    $table->softDeletes();

    // Index columns you filter or sort by
    $table->index('status');
    $table->index('placed_at');
    $table->index('total');

    // Composite index for queries that filter on multiple columns
    $table->index(['user_id', 'status']);
    $table->index(['status', 'placed_at']);
});
```

The rule of thumb: if you use a column in a `WHERE`, `ORDER BY`, or `GROUP BY` clause, it probably needs an index. Foreign keys created with `foreignId()->constrained()` are automatically indexed by Laravel.

### Composite Index Order Matters

The order of columns in a composite index matters. The index `['user_id', 'status']` is used for:

- `WHERE user_id = 1` (uses the index)
- `WHERE user_id = 1 AND status = 'pending'` (uses the index)
- `WHERE status = 'pending'` (does NOT use the index)

This is called the "leftmost prefix" rule — the database can only use the index starting from the leftmost column. Put the column you filter on alone first, and the most selective column first.

### Unique Indexes for Data Integrity

Use unique indexes to enforce uniqueness at the database level, not just in validation:

```php
$table->unique('email');
$table->unique(['user_id', 'product_id']); // One review per product per user
```

Validation prevents most duplicates, but race conditions can slip through. A unique index is the last line of defense — the database will reject the duplicate even if two requests arrive simultaneously.

### Do Not Over-Index

Every index speeds up reads but slows down writes. Each `INSERT`, `UPDATE`, or `DELETE` must update every index on the table. For read-heavy tables (product catalogs, blog posts), index liberally. For write-heavy tables (logs, analytics events), index sparingly and only what you query.

## Migration Best Practices

### Use Descriptive Migration Names

Your migration files are a changelog of your database's evolution. When something breaks in production and you need to find which migration added a column, changed a type, or dropped an index, you run `git log --oneline database/migrations/` and scan the filenames. If every migration is named `update_orders` or `changes`, that changelog is useless.

```bash
# Good — tells you exactly what changed
php artisan make:migration add_tracking_number_to_orders_table
php artisan make:migration create_order_items_table
php artisan make:migration add_composite_index_on_status_and_placed_at_to_orders_table

# Bad — tells you nothing
php artisan make:migration update_orders
php artisan make:migration changes
php artisan make:migration fix_stuff
```

Follow the convention: `{action}_{what}_{to/from}_{table}_table`. Laravel parses these names to pre-fill the migration body — `add_tracking_number_to_orders_table` automatically generates a `Schema::table('orders')` call with the right structure. The migration name should tell you what it does without opening the file.

### The Down Migration Debate

Every `up()` can have a corresponding `down()` that reverses the change:

```php
public function up(): void
{
    Schema::table('orders', function (Blueprint $table): void {
        $table->string('tracking_number')->nullable()->after('status');
        $table->timestamp('shipped_at')->nullable()->after('tracking_number');
    });
}

public function down(): void
{
    Schema::table('orders', function (Blueprint $table): void {
        $table->dropColumn(['tracking_number', 'shipped_at']);
    });
}
```

The `down()` method enables `php artisan migrate:rollback`, which is useful during local development when you are iterating on a schema. You run `migrate`, test, realize the column name is wrong, `rollback`, fix the migration, and `migrate` again.

**However, down migrations are controversial in production.** Freek Van der Herten [argues against them](https://freek.dev/2900-why-i-dont-use-down-migrations), and his reasoning is compelling:

1. **Down migrations are untested code.** You write the `down()` once and never run it until an emergency. By then, it might not work — and you have a broken rollback on top of a broken deployment.

2. **New data makes rollbacks destructive.** If you add a `tracking_number` column and users start filling it in, rolling back drops that column and all its data. If you split `name` into `first_name` and `last_name` and users update their names, the original `name` data no longer exists. Rollbacks destroy data that was created after the migration ran.

3. **Code and database get out of sync.** Your application code depends on the new schema. Rolling back the database without rolling back the code (which is often impractical with deployed containers or serverless functions) breaks everything — models reference non-existent columns, queries fail, and validation rules check vanished fields.

The alternative is **forward-only migrations**: instead of rolling back, you write a new migration that moves the database forward to the desired state. Made a mistake? Write a new migration that fixes it. This approach is more predictable and testable because the fix goes through the same pipeline as every other change.

**The practical approach:** use `down()` during local development for fast iteration. In production, treat migrations as forward-only — if something goes wrong, write a new migration to fix it rather than trying to reverse history.

### Preserve All Column Attributes When Modifying

In Laravel 12, when modifying a column, the migration must include all of the attributes that were previously defined on the column. Otherwise, they will be dropped:

```php
// Original column
$table->string('name')->nullable()->default('guest');

// Bad: modifying without preserving attributes — nullable and default are lost
$table->string('name', 100)->change();

// Good: include all original attributes
$table->string('name', 100)->nullable()->default('guest')->change();
```

### Use Foreign Key Constraints

Without foreign key constraints, your database is a bag of disconnected data. Delete a user and their orders remain, pointing at a `user_id` that no longer exists. Delete a category and its products reference a phantom. These orphaned records cause 404 errors, broken reports, and confused customers — and they accumulate silently over months until someone notices the numbers do not add up.

Foreign key constraints make orphaned records impossible at the database level:

```php
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
$table->foreignId('category_id')->constrained()->nullOnDelete();
$table->foreignId('assigned_to')->nullable()->constrained('users')->nullOnDelete();
```

`cascadeOnDelete()` automatically deletes child records when the parent is deleted — delete a user and their orders disappear with them. `nullOnDelete()` sets the foreign key to `null` instead — delete a category and its products become uncategorized rather than orphaned. Choose based on your domain: should child records survive or die with the parent?

## Query Optimization

### Select Only What You Need

Every column you select travels from the database server to PHP. On small tables with a handful of short columns, this is invisible. But imagine a `posts` table where each row has a `body` column containing 50KB of HTML. Loading 200 posts with `Post::all()` transfers 10MB of data when you only need their titles for a sidebar. That transfer consumes memory, saturates your database connection, and slows every request — all for data you never use.

```php
// Bad: selects all columns — transfers entire rows including large text fields
$users = User::all();

// Good: selects only needed columns
$users = User::select(['id', 'name', 'email'])->get();

// Good: when you need a single column as a key-value pair
$names = User::pluck('name', 'id');
```

`select()` tells the database to return only the columns you list. `pluck()` goes further — it returns a flat collection of values (or a key-value pair), skipping model hydration entirely. Use `select()` when you need model instances with a subset of fields. Use `pluck()` when you need raw values for a dropdown, a lookup table, or a simple list.

### Processing Large Datasets Without Running Out of Memory

At some point, every Laravel application needs to process a large dataset — sending emails to all users, recalculating prices for every product, or exporting a year's worth of orders. The naive approach is to load everything at once:

```php
// Dangerous: loads ALL records into memory simultaneously
Order::all()->each(function (Order $order): void {
    $this->processOrder($order);
});
```

`Order::all()` on a table with 100,000 rows creates 100,000 Eloquent model instances in memory at the same time. Each model consumes roughly 2-5KB depending on the number of columns. With 100,000 rows, that is 200-500MB of memory — enough to hit PHP's `memory_limit` and crash your process. Even if it does not crash, holding that much data in memory slows garbage collection and starves other processes.

Laravel provides three strategies for processing large datasets without loading everything at once. Each makes a different trade-off between memory usage, query count, and API convenience.

#### chunk() — Fixed-Size Batches

`chunk()` loads a fixed number of records per query, processes them, then releases them from memory before loading the next batch:

```php
Order::chunk(200, function (Collection $orders): void {
    $orders->each(function (Order $order): void {
        $this->processOrder($order);
    });
});
```

With 100,000 records and a chunk size of 200, this executes 500 queries instead of one. At any given moment, only 200 models exist in memory. The trade-off is more queries but controlled memory usage. `chunk()` uses `OFFSET` and `LIMIT` under the hood, which means each query re-scans from the beginning — this gets slower as the offset grows on very large tables.

Use `chunkById()` instead of `chunk()` when the table has an auto-incrementing primary key. It uses `WHERE id > ?` instead of `OFFSET`, which is significantly faster on large datasets because the database can use the primary key index to jump directly to the right rows.

#### lazy() — LazyCollection with Fluent API

`lazy()` works like `chunk()` internally — it loads records in batches — but returns a `LazyCollection` instead of requiring a callback. This gives you the full collection API (`filter()`, `map()`, `take()`, etc.) while keeping memory usage low:

```php
Order::lazy()->each(function (Order $order): void {
    $this->processOrder($order);
});

// The real power: chaining collection methods on large datasets
$highValueOrders = Order::lazy()
    ->filter(fn (Order $order) => $order->total > 10000)
    ->map(fn (Order $order) => $order->only(['id', 'total', 'user_id']));
```

Internally, `lazy()` uses `chunkById()` — so it benefits from the same index-based pagination. The difference from `chunk()` is purely ergonomic: you get a fluent collection pipeline instead of nested callbacks. Use `lazy()` when you want to chain collection methods. Use `chunk()` when you need explicit control over batch boundaries (for example, wrapping each batch in a database transaction).

#### cursor() — One Model at a Time

`cursor()` uses a PHP generator backed by a database cursor. It executes a single query and yields one model at a time, making it the most memory-efficient option:

```php
Order::cursor()->each(function (Order $order): void {
    ProcessOrderJob::dispatch($order);
});
```

Only one model exists in memory at any moment. The trade-off is that the database connection stays open for the entire iteration — if processing each record takes time, you risk hitting your database's connection timeout. `cursor()` is ideal for dispatching queued jobs (fast per-record work) or simple transformations, but not for long-running operations per record.

#### Which One Should You Use?

| Strategy | Memory | Queries | Best For |
|---|---|---|---|
| `chunk()` / `chunkById()` | Batch size | Many (N / batch size) | Batch operations, transactional batches |
| `lazy()` | Batch size | Many (N / batch size) | Fluent collection pipelines on large datasets |
| `cursor()` | Single model | One | Dispatching jobs, simple transforms, exports |

### Use Database-Level Operations

When you need to update or delete thousands of records, loading each one as an Eloquent model is wasteful. Each model consumes memory, each `save()` fires a separate query, and model events add overhead you may not need. If you are not relying on accessors, mutators, or model events, let the database do the work in a single query:

```php
// Bad: loads all models to update them — 5,001 queries for 5,000 matches
$orders = Order::where('status', 'pending')
    ->where('placed_at', '<', now()->subDays(30))
    ->get();

foreach ($orders as $order) {
    $order->update(['status' => 'cancelled']);
}

// Good: single query, no models loaded
Order::where('status', 'pending')
    ->where('placed_at', '<', now()->subDays(30))
    ->update(['status' => 'cancelled']);
```

The first example loads every matching model into memory, then issues one `UPDATE` query per model. If 5,000 orders match, that is 5,001 queries. The second example runs a single `UPDATE ... WHERE` query directly on the database — no models loaded, no memory consumed, orders of magnitude faster.

The same applies to `delete()`, `increment()`, and `decrement()`:

```php
// Delete without loading models
Order::where('status', 'cancelled')
    ->where('cancelled_at', '<', now()->subYear())
    ->delete();

// Increment without loading models
Product::where('id', $productId)->increment('view_count');
```

**The trade-off:** bulk operations bypass Eloquent model events (`creating`, `updating`, `deleting`) and observers. If your model relies on events to send notifications, update caches, or log activity, those will not fire. In that case, use chunked processing instead.

### Subqueries Instead of Joins

Laravel's subquery support is often cleaner than raw joins:

```php
$users = User::select(['users.*'])
    ->addSelect([
        'last_order_at' => Order::select('placed_at')
            ->whereColumn('user_id', 'users.id')
            ->latest('placed_at')
            ->limit(1),
    ])
    ->addSelect([
        'total_spent' => Order::selectRaw('SUM(total)')
            ->whereColumn('user_id', 'users.id'),
    ])
    ->orderByDesc('total_spent')
    ->get();
```

Subqueries keep the query readable and avoid the column name conflicts that raw joins produce. They also allow you to sort and filter by computed values without loading related models.

## Schema Design Principles

### Use Appropriate Column Types

The column type you choose affects more than storage — it determines what values the database accepts, how it compares them, how much space they consume, and how efficiently indexes work. A `boolean` column takes 1 byte and compares instantly. A `string` column storing `"true"` or `"false"` takes 4-5 bytes and requires string comparison. A `timestamp` can be sorted and filtered efficiently; a `string` storing `"2024-01-15 10:30:00"` cannot. Choosing the right type upfront avoids data quality issues and performance problems that are painful to fix on a table with millions of rows.

```php
$table->string('name');              // VARCHAR(255) — for short text
$table->text('description');         // TEXT — for long text
$table->integer('quantity');         // INT — for whole numbers
$table->decimal('price', 10, 2);    // DECIMAL — for money (never use float!)
$table->boolean('is_active');       // TINYINT(1) — for true/false
$table->json('metadata');           // JSON — for flexible structured data
$table->timestamp('published_at');  // TIMESTAMP — for dates with time
$table->date('birth_date');         // DATE — for dates without time
```

### Never Use Float for Money

```php
// Bad: floating point arithmetic errors
$table->float('price');  // 0.1 + 0.2 = 0.30000000000000004

// Good: store as cents (integer) or use decimal
$table->integer('price_in_cents');
$table->decimal('price', 10, 2);
```

Floating-point arithmetic is approximate by design. `DECIMAL` stores exact values. For money, the difference between `$99.99` and `$99.99000000000001` is a customer complaint and an accounting discrepancy. See the [Money Value Object](/books/clean-code-in-laravel/enums-value-objects-and-type-safety) pattern for handling money in your application layer.

### Use JSON Columns Deliberately

JSON columns are powerful but easy to misuse:

```php
// Good: truly flexible, schema-less data
$table->json('metadata');  // User preferences, feature flags, API responses

// Bad: structured data that should be its own table
$table->json('addresses'); // Should be an addresses table with proper columns
```

JSON columns cannot be efficiently indexed (except with generated columns), cannot have foreign key constraints, and make queries harder to write. Use them for data that is truly schema-less — configuration, preferences, third-party API payloads. If you find yourself writing `->where('metadata->role', 'admin')`, that data belongs in a real column.

## The Database Checklist

1. **Prevent lazy loading** in development — `Model::preventLazyLoading()`
2. **Eager load relationships** — use `with()` for every relationship you access
3. **Index filtered and sorted columns** — check your `WHERE` and `ORDER BY` clauses
4. **Decide on a down migration strategy** — use `down()` for local iteration, or go forward-only in production
5. **Preserve column attributes** — when modifying columns in Laravel 12, include all original attributes
6. **Use foreign key constraints** — data integrity at the database level
7. **Select only needed columns** — do not load entire rows when you need one field
8. **Chunk large datasets** — never load millions of rows into memory
9. **Use decimal for money** — never float
10. **Use database-level operations** — `update()`, `delete()`, `increment()` without loading models
11. **Do not over-index** — every index slows writes; index what you query, not everything

## Summary

- The N+1 problem is the most common performance issue in Laravel. `Model::preventLazyLoading()` in development catches it before production. Use `with()` to eager load every relationship you access.
- Eager loading supports nested relationships (`items.product.category`), conditional loading (`when()`), count-only loading (`withCount()`), and limiting loaded records (`limit()` in Laravel 12).
- Indexes make the difference between a 500ms query and a 2ms query. Index columns used in `WHERE`, `ORDER BY`, and `GROUP BY` clauses. Composite index column order matters — the "leftmost prefix" rule determines which queries can use the index.
- Use unique indexes to enforce data integrity at the database level, not just in validation. Race conditions can bypass validation; they cannot bypass a unique index.
- Down migrations are a trade-off: `down()` enables local rollbacks for fast iteration, but in production it is untested code that can destroy data created after the migration ran. Many teams use forward-only migrations — fixing mistakes by writing a new migration that moves forward rather than reversing history. In Laravel 12, column modifications must include all original attributes or they are silently dropped.
- Foreign key constraints with `cascadeOnDelete()` or `nullOnDelete()` prevent orphaned records. Data integrity belongs at the database level, not the application level.
- Use `select()` to avoid transferring unnecessary data. For large datasets, `chunk()` and `chunkById()` process fixed-size batches, `lazy()` provides a fluent `LazyCollection` API over batches, and `cursor()` yields one model at a time using a PHP generator. Each trades query count for memory efficiency. Use bulk operations (`update()`, `delete()`, `increment()`) to avoid loading models you only need to modify — but remember they bypass model events.
- Never use `float` for money — use `decimal` or store values as integers (cents). Use JSON columns for truly schema-less data, not for structured data that should be its own table.

## References

- [Eloquent: Relationships — Eager Loading](https://laravel.com/docs/eloquent-relationships#eager-loading) — Laravel Documentation
- [Eloquent: Getting Started — Preventing Lazy Loading](https://laravel.com/docs/eloquent#preventing-lazy-loading) — Laravel Documentation
- [Database: Migrations](https://laravel.com/docs/migrations) — Laravel Documentation
- [Database: Query Builder](https://laravel.com/docs/queries) — Laravel Documentation
- [Eloquent: Getting Started — Chunking Results](https://laravel.com/docs/eloquent#chunking-results) — Laravel Documentation
- [Use the Index, Luke](https://use-the-index-luke.com/) — Markus Winand
- [20 Laravel Eloquent Tips and Tricks](https://laravel-news.com/eloquent-tips-tricks) — Laravel News
- [Eloquent Performance Patterns](https://eloquent-course.reinink.ca/) — Jonathan Reinink
- [Why I don't use down migrations](https://freek.dev/2900-why-i-dont-use-down-migrations) — Freek Van der Herten