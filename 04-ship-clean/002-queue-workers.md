The [Jobs](/books/clean-code-in-laravel/jobs) chapter covered how to write clean, reliable Job classes — structuring them, handling failures, designing for idempotency, and using middleware. But a Job sitting on a queue does nothing. Something has to pick it up and run it. That something is a queue worker.

A queue worker is a long-running PHP process that polls a queue, picks up Jobs one at a time, and executes their `handle()` method. Without a worker, your Jobs are just serialized objects sitting in Redis or a database table, waiting for something that never comes. In development, you might not notice — the `sync` driver runs Jobs inline. But in production, if your workers are misconfigured, down, or overwhelmed, your Jobs pile up, your users wait, and your application silently falls behind.

This chapter covers the operational side of queues: how workers actually process Jobs, how to run them reliably in production, and how to scale them as your application grows. If the [Jobs](/books/clean-code-in-laravel/jobs) chapter was about writing the code, this chapter is about making sure it actually runs.

## Queues vs. Workers

Before diving in, let us clarify two terms that are often used interchangeably but mean different things.

A **queue** is a data structure — a waiting line. It is a list of Jobs stored somewhere (Redis, a database table, Amazon SQS) waiting to be processed. When you call `ProcessOrderPaymentJob::dispatch($order)`, you are adding a Job to a queue. The queue itself does nothing — it is just storage. Think of it like a to-do list pinned to a wall. Writing items on the list does not get them done.

A **worker** is the process that gets things done. It is a PHP script that runs continuously, checks the queue for new Jobs, picks them up one at a time, and executes them. Without a worker, Jobs pile up in the queue indefinitely. The queue is the list; the worker is the person reading the list and doing the work.

In Laravel terms: `dispatch()` adds to the queue. `php artisan queue:work` starts a worker that reads from the queue.

## How a Worker Picks Up Jobs

When you dispatch a Job, Laravel serializes it and pushes it onto a queue — a list in Redis, a row in a database table, or a message in Amazon SQS. The Job sits there until a worker comes along.

### What Is Serialization?

Serialization is the process of converting a PHP object into a string that can be stored or transmitted. PHP's built-in [`serialize()`](https://www.php.net/manual/en/function.serialize.php) function takes an object — its class name, property values, and their types — and encodes it into a flat string representation. [`unserialize()`](https://www.php.net/manual/en/function.unserialize.php) does the reverse: it takes that string and reconstructs the original object. This is how a Job moves between processes. When you call `ProcessOrderPaymentJob::dispatch($order)`, Laravel serializes the Job (including the order's ID, thanks to `SerializesModels`) into a string and stores it in the queue backend. Minutes later, a completely different PHP process — the worker — reads that string, unserializes it back into a `ProcessOrderPaymentJob` instance, and calls `handle()`.

This is why the [Jobs](/books/clean-code-in-laravel/jobs) chapter emphasizes passing data through the constructor and services through `handle()`. Data (IDs, strings, numbers) serializes cleanly. Services (database connections, HTTP clients, logger instances) do not — they hold open connections and internal state that cannot survive serialization. A serialized `StripePaymentService` is a broken `StripePaymentService`.

### The Worker Loop

Under the hood, a worker's `daemon()` method is a `while (true)` loop. Each iteration follows these steps:

1. **Check if the worker should run** — if the application is in maintenance mode (and `--force` is not set), or if the worker has been paused, it sleeps and loops again without processing anything.
2. **Reset application scope** — clear request-scoped state from the previous Job (scoped container bindings, resolved instances). This prevents state from leaking between Jobs.
3. **Get the next Job** — iterate through the comma-separated queue names from `--queue`, calling `pop()` on each until a Job is found or all queues are empty.
4. **Register timeout handler** — if the `pcntl` extension is loaded, schedule a `SIGALRM` signal after `--timeout` seconds. If the alarm fires, the worker force-kills itself. This prevents runaway Jobs from blocking the worker forever.
5. **Process the Job** — fire the `JobProcessing` event, call `$job->fire()` (which ultimately calls `handle()`), fire `JobProcessed`. If the Job throws an exception, check whether it has retries left — if yes, release it back to the queue with a backoff delay; if no, move it to the failed jobs table and call the Job's `failed()` method.
6. **Check stop conditions** — should the worker exit? It checks: did someone send a `SIGTERM`/`SIGQUIT`/`SIGINT`? Has memory exceeded `--memory`? Has the `queue:restart` cache signal changed? Has `--max-time` elapsed? Has `--max-jobs` been reached? If any condition is met, exit gracefully.
7. **Sleep or continue** — if no Job was found, sleep for `--sleep` seconds. If a Job was processed and `--rest` is set, pause for that duration. Then loop back to step 1.

This loop runs indefinitely. The worker does not exit between Jobs — it keeps the Laravel application booted in memory and reuses it for every Job. This is why workers are fast, but it is also why they need special care during deployments and why static state can leak between Jobs.

### A Worker in Plain PHP

To make this concrete, here is a simplified queue worker in plain PHP — no Laravel, no framework. This is the essence of what `queue:work` does under all the abstractions:

```php
// A queue worker is just an infinite loop that reads from a list

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

while (true) {
    // 1. Check the queue for the next job
    $payload = $redis->lPop('queues:default');

    if ($payload === false) {
        // Nothing to do — sleep and try again
        sleep(3);
        continue;
    }

    // 2. Deserialize the job
    $job = unserialize($payload);

    // 3. Execute it
    try {
        $job->handle();
        echo "Processed: " . get_class($job) . "\n";
    } catch (Throwable $e) {
        // Something went wrong — push it back for a retry
        $redis->rPush('queues:default', $payload);
        echo "Failed: " . $e->getMessage() . "\n";
    }

    // 4. Loop back and check for the next job
}
```

That is all a worker is: pop from a list, do the work, repeat. Laravel adds serialization safety, dependency injection, retry logic, timeout handling, signal management, and graceful shutdown — but the core idea is this loop. Every worker you have ever run is a variation of this pattern.

### Signal Handling

If the `pcntl` extension is available, the worker registers signal handlers:

- **SIGTERM / SIGQUIT / SIGINT** — graceful shutdown. The worker finishes the current Job, then exits. This is what `queue:restart` triggers.
- **SIGUSR2** — pause. The worker stops picking up new Jobs but stays alive.
- **SIGCONT** — resume. The worker starts processing again.

This is why workers respond gracefully to `Ctrl+C` — it sends `SIGINT`, and the worker finishes the current Job before exiting. It does not abandon the Job mid-execution.

## Queue Connections and Drivers

Laravel supports multiple queue backends. I am going to be blunt: **use Redis**. It is my favorite driver, and for good reason. But let me explain the alternatives so you understand why.

### The `sync` Driver

The `sync` driver does not use a queue at all — it runs the Job inline, during the current request. A payment Job that takes five seconds blocks the user for five seconds. This is useful for testing, where you want Jobs to execute immediately so you can assert on their side effects. It has no place in production.

```env
QUEUE_CONNECTION=sync
```

### The `database` Driver

The `database` driver stores Jobs in a database table. It requires no additional infrastructure — if you have a database, you have a queue:

```bash
php artisan queue:table
php artisan migrate
```

```env
QUEUE_CONNECTION=database
```

The worker polls the database by running a `SELECT ... FOR UPDATE SKIP LOCKED` query every `--sleep` seconds. `SKIP LOCKED` is the critical piece — it tells the database to skip rows that are already locked by another worker's transaction, effectively giving each worker a different Job. Without it (on older database versions), workers would serialize on a single lock, processing one Job at a time regardless of how many workers you run.

The database driver works for low-to-medium throughput. But it has structural limitations:

- **Polling overhead** — every `--sleep` seconds, the worker runs a SELECT query, even when the queue is empty. Three workers with `--sleep=3` means three queries per second against your database doing nothing.
- **Shared database load** — queue polling competes with your application's regular queries for database connections, CPU, and I/O.
- **No blocking wait** — the database has no equivalent of Redis's `BLPOP`. The worker must poll at fixed intervals, so there is always latency between a Job being dispatched and a worker picking it up (up to `--sleep` seconds).

For a side project or a low-traffic application, the database driver is fine. For anything with meaningful throughput, use Redis.

### The `redis` Driver

Redis is purpose-built for this kind of work. It is an in-memory data store with atomic operations, blocking pops, and sub-millisecond latency. This is the driver you should use in production, and it is what I use in every application I build.

```env
QUEUE_CONNECTION=redis
```

Here is why Redis is superior to every other option:

**In-memory speed.** All Redis operations happen in memory. Pushing a Job (`RPUSH`) is O(1). Popping a Job (`LPOP`) is O(1). There is no disk I/O, no query planner, no transaction log to sync. A single Redis instance handles hundreds of thousands of operations per second.

**Blocking wait with `BLPOP`.** When a queue is empty, the worker does not poll — it calls `BLPOP`, which blocks the Redis connection server-side with zero CPU usage. The instant a producer pushes a Job, Redis wakes the blocked worker. No wasted queries, no polling latency, no unnecessary load.

**Atomic Lua scripts.** Laravel's Redis queue uses Lua scripts for critical operations like popping a Job and reserving it. Redis executes Lua scripts atomically — no other command can interleave during execution. This means `LPOP` + `ZADD` (move from queue list to reserved sorted set) happens as a single indivisible operation. Two workers can never pop the same Job, and a Job can never be popped but lost because the worker crashed before reserving it. The database driver achieves similar atomicity with `FOR UPDATE SKIP LOCKED`, but with significantly more overhead.

Redis is also required for [Laravel Horizon](#laravel-horizon), which is the best way to manage workers in production. If you are using the database driver and wondering whether to switch to Redis, the answer is yes.

#### How Redis Queues Work Under the Hood

Laravel uses three Redis data structures per queue:

- **`queues:{name}`** — a Redis LIST for ready Jobs. New Jobs are pushed with `RPUSH`, workers pop with `LPOP`. FIFO order.
- **`queues:{name}:delayed`** — a Redis SORTED SET for delayed Jobs. The score is the Unix timestamp when the Job becomes available. Laravel periodically migrates Jobs whose scores are ≤ `now()` back to the main list.
- **`queues:{name}:reserved`** — a Redis SORTED SET for in-flight Jobs. When a worker pops a Job, it immediately adds it here with a score of `now() + retry_after`. If the worker crashes or the Job times out, the Job's score eventually falls below `now()`, and the migration script moves it back to the main list for another worker to pick up.

There is also a **`queues:{name}:notify`** list used as a signaling channel. When a Job is pushed, a `1` is also pushed to `:notify` via the same Lua script. Workers `BLPOP` on `:notify` rather than the main list — this separation allows the actual pop to use Lua scripts for atomicity while still benefiting from blocking behavior.

When a worker calls `pop()`, two things happen:

1. **Migrate expired Jobs** — a Lua script scans both the `delayed` and `reserved` sorted sets for Jobs with scores ≤ `now()` and moves them back to the main list. This handles both delayed Jobs that are ready and reserved Jobs whose workers crashed.
2. **Pop the next Job** — a Lua script atomically `LPOP`s from the main list, increments the Job's `attempts` counter, and `ZADD`s it to the `:reserved` sorted set with a score of `now() + retry_after`. If the list is empty and `block_for` is configured, the worker falls back to `BLPOP` on the `:notify` list.

All of this happens in Redis's single-threaded event loop, with Lua scripts guaranteeing atomicity. It is elegant, fast, and reliable.

#### The `block_for` Configuration

By default, the Redis worker uses sleep-based polling (like the database driver). But you can enable blocking by setting `block_for` in your Redis queue connection config:

```php
// config/queue.php
'redis' => [
    'driver' => 'redis',
    'connection' => env('REDIS_QUEUE_CONNECTION', 'default'),
    'queue' => env('REDIS_QUEUE', 'default'),
    'retry_after' => env('REDIS_QUEUE_RETRY_AFTER', 90),
    'block_for' => 5,
],
```

With `block_for => 5`, the worker blocks on `BLPOP` for up to 5 seconds. If a Job arrives during that time, the worker wakes up immediately. If not, it returns and loops again (checking for signals, memory limits, etc.).

Do not set `block_for` to `0` (block forever) — during a `BLPOP`, PHP cannot process POSIX signals. The worker will not respond to `queue:restart`, `SIGTERM`, or graceful shutdown until a Job arrives or the connection times out. A value of `5` is a good compromise.

### Amazon SQS

If your application runs on AWS, SQS is a fully managed queue service that handles scaling and availability for you. You never worry about Redis memory or database polling — AWS manages the infrastructure:

```env
QUEUE_CONNECTION=sqs
```

SQS is a reasonable choice if you are already in the AWS ecosystem. The trade-offs are higher latency than Redis (SQS polls over HTTP), no support for queue priorities within a single worker, and no compatibility with Horizon.

### Which Driver Should You Use?

| Driver | Use When |
|---|---|
| `sync` | Testing only. Never in production |
| `database` | Low-traffic apps where simplicity matters more than performance |
| `redis` | Everything else. This is the production default |
| `sqs` | You are on AWS and want fully managed infrastructure |

If you are starting a new project, start with Redis. If you are migrating from the database driver, switch to Redis. If you are using SQS but want Horizon, switch to Redis. The answer is almost always Redis.

## Running Workers Locally

Start a worker with the `queue:work` Artisan command:

```bash
php artisan queue:work
```

This starts a single worker process that polls the default queue on the default connection. It runs until you stop it with `Ctrl+C` or close the terminal.

### `queue:work` vs. `queue:listen`

Laravel provides two commands for processing Jobs. They look similar but work very differently.

`queue:work` boots the Laravel application once and keeps it in memory. Every Job runs in the same process, using the same application instance. This is fast — no boot overhead per Job — but the worker will not notice code changes. If you update a Job class while the worker is running, it keeps using the old code.

`queue:listen` spawns a fresh `queue:work --once` process for each Job. The framework boots and shuts down for every single Job. It picks up code changes automatically, but it is significantly slower.

Use `queue:work` for production. Use `queue:listen` only during local development when you want automatic code reloading and do not want to restart the worker manually after every change.

### Worker Options

The `queue:work` command accepts several options that control how the worker behaves:

```bash
php artisan queue:work redis --queue=payments,emails --tries=3 --timeout=90 --max-jobs=1000 --max-time=3600 --memory=128 --sleep=3 --rest=0
```

| Option | Default | What It Does |
|---|---|---|
| `--queue` | `default` | Which queues to process, in priority order |
| `--tries` | `1` | Maximum attempts per Job before marking it as failed |
| `--timeout` | `60` | Maximum seconds a Job can run before the worker kills it |
| `--max-jobs` | `0` (unlimited) | Exit after processing this many Jobs |
| `--max-time` | `0` (unlimited) | Exit after running for this many seconds |
| `--memory` | `128` | Exit if memory usage exceeds this many MB |
| `--sleep` | `3` | Seconds to wait when no Jobs are available |
| `--rest` | `0` | Seconds to pause between processing Jobs |
| `--backoff` | `0` | Seconds to wait before retrying a failed Job |
| `--force` | — | Process Jobs even when the application is in maintenance mode |

Three of these options deserve special attention.

**`--max-jobs` and `--max-time`** are your insurance against memory leaks. PHP was designed for short-lived request cycles, not long-running daemons. Over hours or days, a worker can accumulate memory that the garbage collector cannot reclaim — event listeners that pile up across Jobs, static properties that grow unbounded, cached data in singleton services, Eloquent model relationships loaded across thousands of Jobs, and third-party libraries that maintain internal state. Setting `--max-jobs=1000` or `--max-time=3600` tells the worker to exit gracefully after reaching the limit. Your process manager (Supervisor) restarts it immediately with a clean slate. Always set at least one of these.

**`--sleep`** controls how aggressively the worker polls an empty queue. With `--sleep=3`, the worker waits three seconds before checking again when there are no Jobs. This prevents unnecessary load on your queue backend. If you are using Redis with `block_for`, this is less relevant since `BLPOP` handles the waiting. For the database driver, it directly controls polling frequency.

**`--timeout`** must always be shorter than `retry_after` in your queue connection config. If `--timeout=60` and `retry_after=90`, a Job that times out at 60 seconds gets killed, released back to the queue, and picked up by another worker after 90 seconds. If `retry_after` were shorter than `--timeout`, a still-running Job could be picked up by another worker simultaneously — causing duplicate processing. Always keep a margin: if `--timeout=60`, set `retry_after` to at least `90`.

## Queue Priorities

When you pass multiple queue names to `--queue`, the worker processes them in order — it drains the first queue before moving to the second:

```bash
php artisan queue:work --queue=payments,notifications,emails
```

This worker always processes `payments` Jobs first. Only when the `payments` queue is empty does it check `notifications`. Only when both are empty does it check `emails`. This ensures that a flood of marketing emails never delays payment processing.

Dispatch Jobs to specific queues using the `onQueue()` method:

```php
ProcessOrderPaymentJob::dispatch($order)->onQueue('payments');
SendMarketingEmailJob::dispatch($user)->onQueue('emails');
```

For strict separation — where a flood on one queue should never affect another — run dedicated workers per queue:

```bash
php artisan queue:work --queue=payments
php artisan queue:work --queue=emails
```

Now each queue has its own worker. If the `emails` queue backs up with 50,000 Jobs, the `payments` worker is completely unaffected.

## Supervisor: Keeping Workers Running in Production

A worker is just a PHP process. It can crash, run out of memory, or get killed during a deployment. In production, you need something that watches your workers and restarts them automatically. That something is [Supervisor](http://supervisord.org/) — a process monitor for Linux.

### Installing Supervisor

On Ubuntu:

```bash
sudo apt-get install supervisor
```

### Configuring Supervisor

Create a configuration file at `/etc/supervisor/conf.d/laravel-worker.conf`:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /home/forge/app.com/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=forge
numprocs=4
redirect_stderr=true
stdout_logfile=/home/forge/app.com/storage/logs/worker.log
stopwaitsecs=3600
```

Each directive matters:

| Directive | Purpose |
|---|---|
| `process_name` | Names each process with a number suffix (`laravel-worker_00`, `laravel-worker_01`, etc.) |
| `command` | The Artisan command to run. Include all the options you need |
| `autostart=true` | Start the workers when Supervisor starts (e.g., after a server reboot) |
| `autorestart=true` | Restart the worker if it exits for any reason — whether it crashed, hit `--max-time`, or was told to exit by `queue:restart` |
| `stopasgroup=true` | Send the stop signal to the entire process group, not just the parent. Without this, child processes can outlive the worker |
| `killasgroup=true` | Kill the entire process group if the stop signal is ignored |
| `user` | The system user to run the worker as. Should match your application's file ownership |
| `numprocs` | How many independent worker processes to run. Each is a separate PHP process |
| `stopwaitsecs` | How long to wait for a worker to finish its current Job before killing it |

The `stopwaitsecs` value is critical. Set it higher than your longest-running Job's timeout. If your slowest Job takes up to 300 seconds, set `stopwaitsecs=330`. Otherwise, Supervisor kills the worker before the Job finishes — the Job is left in a half-completed state, and the data it was modifying may be corrupt.

### Starting and Managing Workers

After creating the configuration file:

```bash
# Reload Supervisor's configuration
sudo supervisorctl reread
sudo supervisorctl update

# Start the workers
sudo supervisorctl start "laravel-worker:*"

# Check status
sudo supervisorctl status

# Stop all workers
sudo supervisorctl stop "laravel-worker:*"

# Restart all workers
sudo supervisorctl restart "laravel-worker:*"
```

### Multiple Worker Groups

For applications with distinct queue priorities, define separate Supervisor programs:

```ini
[program:worker-payments]
process_name=%(program_name)s_%(process_num)02d
command=php /home/forge/app.com/artisan queue:work redis --queue=payments --tries=3 --max-time=3600
numprocs=4
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=forge
redirect_stderr=true
stdout_logfile=/home/forge/app.com/storage/logs/worker-payments.log
stopwaitsecs=3600

[program:worker-emails]
process_name=%(program_name)s_%(process_num)02d
command=php /home/forge/app.com/artisan queue:work redis --queue=emails --tries=3 --max-time=3600
numprocs=2
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=forge
redirect_stderr=true
stdout_logfile=/home/forge/app.com/storage/logs/worker-emails.log
stopwaitsecs=3600
```

This gives payments four dedicated workers and emails two. They are completely independent — you can stop the email workers for maintenance without affecting payments.

## Laravel Horizon

Supervisor gets the job done, but managing it means SSH-ing into servers, editing `.conf` files, and manually adjusting process counts. There is no dashboard, no metrics, and no way to dynamically rebalance workers across queues based on load. You are flying blind.

[Laravel Horizon](https://laravel.com/docs/horizon) changes everything. It is a code-driven configuration system, process manager, auto-scaler, and real-time dashboard — all in one package. It is the best way to manage queue workers, and if you are using Redis (which you should be), there is no reason not to use it.

### Horizon and `queue:work`: How They Relate

A common misconception is that Horizon replaces the queue worker. It does not. Horizon is a layer *on top of* `queue:work`. When you run `php artisan horizon`, Horizon reads your configuration and starts multiple `queue:work` processes behind the scenes — deciding how many to run, which queues they serve, and how to distribute them. The underlying worker loop is identical. Horizon adds orchestration, auto-scaling, monitoring, and a dashboard.

Think of it this way:

- **Without Horizon:** you run `php artisan queue:work` yourself and use Supervisor to keep it alive. You manage process counts by editing `.conf` files on the server.
- **With Horizon:** you run `php artisan horizon` and use Supervisor to keep *that* alive. Horizon manages the workers for you based on `config/horizon.php`, which lives in your repository.

You still need Supervisor in both cases — something has to restart `horizon` if the master process dies. But Supervisor's job is simpler: run one `horizon` process. Horizon handles everything else.

| Aspect | `queue:work` + Supervisor | Horizon + Supervisor |
|---|---|---|
| Worker management | Manual — edit `.conf` files on the server | Code-driven — `config/horizon.php` in your repository |
| Process count | Static `numprocs` in Supervisor config | Dynamic auto-scaling based on queue load |
| Dashboard | None — SSH in and check `supervisorctl status` | Real-time web dashboard at `/horizon` |
| Metrics | None — build your own | Built-in throughput, runtime, wait times, failure tracking |
| Balancing | Fixed — each worker has a static queue assignment | Dynamic — workers rebalance across queues as load changes |
| Notifications | Build your own | Built-in Slack, email, and SMS alerts for long wait times |
| Queue driver | Any (database, Redis, SQS) | Redis only |

### Installing Horizon

```bash
composer require laravel/horizon
php artisan horizon:install
```

This publishes `config/horizon.php` (the main configuration) and `app/Providers/HorizonServiceProvider.php` (dashboard authorization).

### Understanding `config/horizon.php`

Horizon's configuration introduces two concepts that often confuse newcomers: **environments** and **supervisors**. Neither has anything to do with Supervisor (the Linux tool). They are Horizon-specific concepts.

#### Environments

An environment maps to your `APP_ENV` value. It lets you define different worker configurations for production, staging, and local:

```php
'environments' => [
    'production' => [
        // High process counts, auto-scaling
    ],
    'local' => [
        // Low process counts for development
    ],
],
```

When Horizon starts, it reads `APP_ENV` and uses the matching environment configuration. If no match is found and you have defined a `*` wildcard environment, Horizon uses that as a fallback. This means your worker configuration is version-controlled alongside your application code — no more SSH-ing into servers to adjust process counts. You change the config, deploy, and Horizon picks up the new settings.

#### Supervisors (Horizon's Kind)

A Horizon supervisor is a **logical group of worker processes**. It is not the Linux Supervisor process monitor — the naming is unfortunate but the concepts are separate.

Each Horizon supervisor:
- Manages a pool of worker processes (PHP processes running `queue:work`)
- Has its own queue assignments
- Has its own balancing strategy and process limits
- Can be independently paused, continued, and scaled
- Runs as part of the `php artisan horizon` master process

When Horizon starts, it reads the supervisor definitions and spawns worker processes accordingly. The Horizon master process monitors its supervisors in a `while (true)` loop, checking every second whether workers need to be added, removed, or rebalanced.

Here is a complete, production-ready configuration:

```php
'environments' => [
    'production' => [
        'supervisor-payments' => [
            'connection' => 'redis',
            'queue' => ['payments'],
            'balance' => 'auto',
            'autoScalingStrategy' => 'time',
            'minProcesses' => 2,
            'maxProcesses' => 5,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
            'tries' => 3,
            'timeout' => 90,
            'maxTime' => 3600,
        ],
        'supervisor-general' => [
            'connection' => 'redis',
            'queue' => ['default', 'notifications', 'emails'],
            'balance' => 'auto',
            'autoScalingStrategy' => 'time',
            'minProcesses' => 1,
            'maxProcesses' => 10,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
            'tries' => 3,
            'timeout' => 60,
            'maxTime' => 3600,
        ],
    ],

    'local' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue' => ['payments', 'default', 'notifications', 'emails'],
            'balance' => 'auto',
            'minProcesses' => 1,
            'maxProcesses' => 3,
            'tries' => 3,
        ],
    ],
],
```

This defines two supervisors in production: `supervisor-payments` with 2–5 workers dedicated to the `payments` queue, and `supervisor-general` with 1–10 workers shared across `default`, `notifications`, and `emails`. In the local environment, a single supervisor with up to 3 workers handles everything.

Why two production supervisors instead of one? Because payments must never be delayed by a backlog of marketing emails. With a single supervisor using `auto` balancing, Horizon might allocate all 10 workers to `emails` if that queue is swamped — leaving `payments` with zero workers. With separate supervisors, `payments` always has at least 2 dedicated workers. Their resources are isolated.

### Balancing Strategies

Horizon offers three strategies for distributing workers across queues within a supervisor:

**`auto` (recommended)** — dynamically adjusts the number of worker processes per queue based on the current workload. If the `notifications` queue has 1,000 pending Jobs while `default` is empty, Horizon allocates more workers to `notifications` until the backlog clears.

The auto-scaler runs every `balanceCooldown` seconds (default 3). For each queue, it collects the current Job count and average runtime, calculates each queue's share of the total workload, and allocates workers proportionally. The `balanceMaxShift` option (default 1) limits how many workers can be added or removed per cycle — preventing aggressive scaling oscillations.

The `autoScalingStrategy` option controls what "workload" means:
- **`time`** — allocates based on estimated time to clear the queue (`job count × average runtime`). A queue with 100 Jobs that each take 10 seconds gets more workers than a queue with 100 Jobs that each take 100 milliseconds. This is the smarter choice when Job durations vary.
- **`size`** — allocates based on the number of pending Jobs, ignoring duration. Simpler, but can over-allocate workers to queues with many fast Jobs.

**`simple`** — distributes workers evenly across queues with a fixed process count. If you have 10 processes and 2 queues, each queue gets 5 workers, regardless of load. Use this when you want predictable, static resource allocation:

```php
'supervisor-1' => [
    'queue' => ['default', 'notifications'],
    'balance' => 'simple',
    'processes' => 10,
],
```

**`false`** — processes queues in the order they are listed, like passing `--queue=payments,emails` to `queue:work`. The first queue is drained before the second is checked. Horizon still scales the number of workers between `minProcesses` and `maxProcesses`, but strict priority is enforced between queues. Use this when queue ordering matters more than throughput.

**Important:** when using `auto` balancing, queue ordering within a supervisor does *not* define priority. The order of `['payments', 'default']` in the config has no effect — Horizon allocates workers purely by load. If you need priority, use the `false` strategy or separate supervisors.

### The Dashboard

Horizon includes a web dashboard at `/horizon` that shows:

- **Current status** — active, paused, or inactive
- **Worker processes** — how many are running, per supervisor and per queue
- **Recent Jobs** — completed, failed, and pending, with runtime and payload details
- **Failed Jobs** — stack traces, retry buttons, and the ability to delete
- **Metrics** — throughput, runtime averages, and wait times over time
- **Tags** — Horizon automatically tags Jobs with associated Eloquent model IDs (e.g., `App\Models\Order:42`), making it easy to find all Jobs related to a specific record

By default, the dashboard is only accessible in the `local` environment. For production, configure the authorization gate in `app/Providers/HorizonServiceProvider.php`:

```php
protected function gate(): void
{
    Gate::define('viewHorizon', function (User $user): bool {
        return $user->isAdmin();
    });
}
```

### Horizon Metrics

Horizon tracks Job throughput, average runtime, and queue wait times. To populate the metrics dashboard with historical data, schedule the snapshot command:

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('horizon:snapshot')->everyFiveMinutes();
```

### Horizon Notifications

Horizon can alert you when a queue's wait time exceeds a threshold:

```php
// config/horizon.php
'waits' => [
    'redis:payments' => 30,      // Alert if payments wait > 30 seconds
    'redis:default' => 60,       // Alert if default wait > 60 seconds
    'redis:emails' => 300,       // Emails can wait up to 5 minutes
],
```

Configure notification channels in `HorizonServiceProvider`:

```php
public function boot(): void
{
    parent::boot();

    Horizon::routeSlackNotificationsTo('https://hooks.slack.com/...', '#alerts');
    Horizon::routeMailNotificationsTo('ops@example.com');
}
```

### Running Horizon in Production

Horizon is a long-running process, just like `queue:work`. Use Supervisor to keep it running:

```ini
[program:horizon]
process_name=%(program_name)s
command=php /home/forge/app.com/artisan horizon
autostart=true
autorestart=true
user=forge
redirect_stderr=true
stdout_logfile=/home/forge/app.com/storage/logs/horizon.log
stopwaitsecs=3600
```

Notice `numprocs` is not set (defaults to 1). You only run one Horizon process — Horizon manages all the workers internally. Setting `numprocs=4` would start four Horizon master processes, each spawning its own full set of workers — quadrupling your resource usage for no benefit.

### Horizon During Local Development

During development, use `horizon:listen` instead of `horizon` for automatic code reloading:

```bash
php artisan horizon:listen
```

This watches your project files for changes and restarts workers automatically. It requires Node.js and the [Chokidar](https://github.com/paulmillr/chokidar) file-watching library:

```bash
npm install --save-dev chokidar
```

### Silencing Noisy Jobs

Some Jobs run frequently but are not interesting — scheduled cleanup tasks, heartbeat checks, internal sync Jobs. They clutter the "Completed Jobs" list on the dashboard. Silence them:

```php
// config/horizon.php
'silenced' => [
    App\Jobs\PruneStaleRecordsJob::class,
],
```

Or implement `Laravel\Horizon\Contracts\Silenced` on the Job class itself.

## Deployment

Workers cache the application in memory. When you deploy new code, existing workers are still running the old version. You must restart them.

### How the Restart Signal Works

When you run `queue:restart`, Laravel sets a cache key (`illuminate:queue:restart`) with the current timestamp. After each Job, every worker compares the cached timestamp against the one it stored at startup. If they differ, the worker exits gracefully. Supervisor (or Horizon) restarts it with the new code.

This mechanism has a few gotchas:

- **You need a real cache driver.** The restart signal is stored in the cache. If your cache driver is `array` (in-memory, per-process), workers will never see it. Use `redis`, `database`, or `file`.
- **All processes must share the same cache store.** If your web server writes to one Redis instance but your workers read from another, the signal never arrives.
- **Cache prefix changes break restarts.** If you change the cache prefix in config, existing workers check the old prefixed key while the restart command writes to the new one. The workers never see the signal.

### Without Horizon

```bash
php artisan queue:restart
```

Workers finish their current Job, then exit. Supervisor restarts them with the new code.

### With Horizon

```bash
php artisan horizon:terminate
```

Horizon finishes all currently-processing Jobs, then exits. Supervisor restarts it, and Horizon boots fresh workers.

### Deployment Script Example

A typical deployment script:

```bash
# Pull latest code
git pull origin main

# Install dependencies
composer install --no-dev --optimize-autoloader

# Run migrations
php artisan migrate --force

# Clear caches
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Restart workers (pick one)
php artisan queue:restart      # Without Horizon
php artisan horizon:terminate  # With Horizon
```

## Pausing Workers

Sometimes you need to stop processing Jobs temporarily — during a database migration that changes a table your Jobs depend on, or during a maintenance window for an external service your Jobs call.

```bash
# Pause a specific queue
php artisan queue:pause redis:payments

# Resume it
php artisan queue:continue redis:payments
```

The format is `connection:queue`. Paused workers stay running but stop picking up new Jobs. They resume immediately when you run `queue:continue`.

With Horizon, you can pause and resume entire supervisors:

```bash
php artisan horizon:pause-supervisor supervisor-payments
php artisan horizon:continue-supervisor supervisor-payments
```

## Scaling Workers

### Why Scaling Matters

A single worker processes Jobs one at a time, sequentially. If each Job takes one second and you dispatch 10 Jobs per second, a single worker falls behind by 9 Jobs every second. After one minute, 540 Jobs are waiting. After ten minutes, 5,400. Your users submitted orders that are "processing" but nothing is happening — payments are not charging, emails are not sending, reports are not generating. The queue grows faster than the worker can drain it.

This is the scaling problem. A single worker is a bottleneck. When your application dispatches more Jobs than one worker can handle, you need more workers — more PHP processes reading from the same queue in parallel.

### What Scaling Means

Scaling queue workers means running multiple worker processes simultaneously. Each process is an independent PHP process running its own `while (true)` loop, polling the same queue. When a Job arrives, whichever worker is free picks it up. The queue backend handles the coordination — Redis uses atomic Lua scripts, the database uses `FOR UPDATE SKIP LOCKED` — so two workers never grab the same Job.

If one worker handles 1 Job per second, four workers handle 4 per second. Ten workers handle 10 per second. The math is straightforward:

```
workers needed = dispatch rate × average Job duration
```

If you dispatch 10 Jobs per second and each takes 500 milliseconds, you need 5 workers to keep up. If each takes 2 seconds, you need 20.

But more workers is not always better. Each worker is a PHP process consuming 50–100 MB of memory. Ten workers on a 1 GB server leaves almost nothing for your web application, your database, or the operating system itself. Workers also compete for CPU — if your Jobs are CPU-intensive (image processing, PDF generation), running more workers than you have CPU cores causes context-switching overhead that actually slows things down. If your Jobs are I/O-bound (API calls, file uploads), you can safely exceed the core count because workers spend most of their time waiting for external responses.

In practice, start small and monitor. Two to four workers is a reasonable starting point for most applications. Watch your queue wait times — if Jobs are waiting more than a few seconds, add workers. If workers are idle most of the time, reduce them.

### Scaling with Horizon

Horizon's `auto` balancing strategy handles this dynamically — adding workers when the queue is busy and removing them when it is not. You set the boundaries, and Horizon makes the decisions:

```php
'supervisor-1' => [
    'balance' => 'auto',
    'minProcesses' => 2,
    'maxProcesses' => 15,
    'balanceMaxShift' => 1,
    'balanceCooldown' => 3,
],
```

During quiet periods, Horizon runs 2 workers (`minProcesses`). When a batch import dispatches 10,000 Jobs, Horizon notices the backlog and starts adding workers — one every 3 seconds (`balanceCooldown`), at most one at a time (`balanceMaxShift`) — up to 15 (`maxProcesses`). When the backlog clears, it scales back down to 2.

This is the single strongest reason to use Horizon. Without it, you either over-provision (wasting memory on idle workers) or under-provision (Jobs back up during spikes). Horizon finds the balance automatically.

### Scaling Without Horizon

Without Horizon, scaling is manual. You change the `numprocs` value in your Supervisor configuration and reload:

```bash
sudo supervisorctl reread
sudo supervisorctl update
```

This requires SSH access and a judgment call about how many workers to run. You pick a number, deploy, and hope it is right. If traffic spikes at 2 AM, nobody is there to add workers. If traffic drops to zero on weekends, those workers sit idle consuming memory. For dynamic scaling without Horizon, you would need to build your own tooling — which is why most production Laravel applications use Horizon with Redis.

## Monitoring Workers

A queue system without monitoring is a queue system where Jobs silently fail and backlogs grow unnoticed.

### Queue Events

Laravel dispatches events at every stage of Job processing. Hook into them for custom monitoring:

```php
// AppServiceProvider::boot()

use Illuminate\Support\Facades\Queue;
use Illuminate\Queue\Events\JobProcessing;
use Illuminate\Queue\Events\JobProcessed;
use Illuminate\Queue\Events\JobFailed;

Queue::before(function (JobProcessing $event): void {
    Log::info("Processing: {$event->job->resolveName()}", [
        'connection' => $event->connectionName,
        'queue' => $event->job->getQueue(),
    ]);
});

Queue::after(function (JobProcessed $event): void {
    // Job completed successfully
});

Queue::failing(function (JobFailed $event): void {
    // Job permanently failed — all retries exhausted
    Log::error("Failed: {$event->job->resolveName()}", [
        'exception' => $event->exception->getMessage(),
    ]);
});
```

The `Queue::looping()` hook runs before each poll cycle — before the worker attempts to fetch the next Job. It is useful for cleanup work like rolling back dangling database transactions left by a crashed Job:

```php
Queue::looping(function (): void {
    while (DB::transactionLevel() > 0) {
        DB::rollBack();
    }
});
```

### Laravel Pulse

[Laravel Pulse](https://laravel.com/docs/pulse) provides real-time application performance monitoring, including queue metrics. It shows slow Jobs, queue throughput, and worker health alongside your application's other performance data. If you need monitoring beyond what Horizon's dashboard provides, Pulse is a natural companion.

## Common Pitfalls

- **Forgetting to restart workers after deployment.** This is the single most common queue issue in production. You deploy new code, but the workers are still running the old version. Jobs fail with "class not found" errors, or worse, they succeed but do the wrong thing because they are using stale logic. Always include `queue:restart` or `horizon:terminate` in your deployment script.
- **Not using a process manager.** Running `queue:work` in a terminal or a `screen` session is not production-grade. The worker will eventually die — out of memory, an unhandled signal, a server restart — and nobody will restart it. Use Supervisor.
- **Setting `stopwaitsecs` too low.** If Supervisor's `stopwaitsecs` is lower than your longest Job's timeout, Supervisor kills the worker before the Job finishes. The Job is left in a half-completed state — a partially processed import, a charge with no matching order update. Set `stopwaitsecs` higher than your longest Job's timeout.
- **Running too many workers on a small server.** Each worker is a PHP process consuming 50–100 MB of memory. Eight workers on a 1 GB server leaves almost nothing for your web application. Start with 2–4 workers and scale based on actual load, not hope.
- **Using the `database` driver at high throughput.** The database driver polls with `SELECT ... FOR UPDATE` queries. At hundreds of Jobs per second, this creates lock contention and slows down both your queue and your application's regular queries. Switch to Redis.
- **Not separating queues by importance.** Running payments and marketing emails on the same queue means a batch import of 100,000 email Jobs blocks payment processing for hours. Always separate critical and non-critical work into different queues with dedicated workers or Horizon supervisors.
- **Ignoring memory limits.** Workers are long-running processes. Memory leaks that are invisible in a single HTTP request compound over thousands of Jobs. Always set `--max-jobs` or `--max-time` to force periodic restarts, and set `--memory` as a safety net.
- **`retry_after` shorter than `--timeout`.** If a Job is still running when `retry_after` expires, the queue thinks the worker crashed and hands the Job to another worker. Now two workers are processing the same Job simultaneously. Always set `retry_after` longer than `--timeout`.
- **Lost database connections.** Workers hold database connections open for hours or days. Databases may close idle connections. When this happens, the next Job fails with a "server has gone away" error. Laravel detects this with the `DetectsLostConnections` trait and exits the worker so Supervisor can restart it with a fresh connection. But if you see these errors frequently, check your database's `wait_timeout` setting.
- **Deleted models between dispatch and execution.** When a Job dispatches with an Eloquent model via `SerializesModels`, only the model's ID is stored. When the worker deserializes the Job, it re-fetches the model from the database. If the model was deleted between dispatch and execution, the Job throws a `ModelNotFoundException`. Set `public $deleteWhenMissingModels = true` on the Job (or use the `#[DeleteWhenMissingModels]` attribute) to silently delete it instead of failing.

## The Queue Workers Checklist

1. **Use Redis** — it is the fastest driver, supports `BLPOP` blocking, uses atomic Lua scripts, and is required for Horizon
2. **Never run `sync` in production** — it defeats the purpose of queues
3. **Use Supervisor** to keep workers (or Horizon) running
4. **Set `--max-jobs` or `--max-time`** — force periodic restarts to prevent memory leaks
5. **Set `--tries` and `--timeout`** — always plan for failure and runaway Jobs
6. **Set `retry_after` longer than `--timeout`** — prevent duplicate Job processing
7. **Separate queues by priority** — critical work (payments) and bulk work (emails) should never compete for the same workers
8. **Restart workers on every deployment** — `queue:restart` or `horizon:terminate` in your deploy script
9. **Set `stopwaitsecs` higher than your longest Job** — never let Supervisor kill a Job mid-execution
10. **Monitor queue wait times** — if Jobs are waiting too long, you need more workers
11. **Use Horizon** — code-driven configuration, auto-balancing, dashboard, and metrics in one package

## Summary

- A queue worker is a long-running PHP process that runs a `while (true)` loop: poll the queue, reserve a Job, deserialize it, call `handle()`, acknowledge it, and repeat. Without workers, dispatched Jobs sit idle forever.
- Serialization converts a Job object into a string for storage in the queue backend. The worker deserializes it back into an object and calls `handle()`. This is why you pass data through the constructor and services through `handle()` — data serializes cleanly, services do not.
- Workers boot the Laravel application once and reuse it across Jobs. This makes them fast, but they do not see code changes — you must restart them after every deployment.
- **Use Redis.** It is the best queue driver by a wide margin — in-memory speed, atomic Lua scripts, `BLPOP` blocking, and Horizon compatibility. The database driver works for low-traffic apps but does not scale. The `sync` driver has no place in production.
- Redis uses three data structures per queue: a LIST for ready Jobs, a SORTED SET for delayed Jobs, and a SORTED SET for reserved (in-flight) Jobs. A Lua script atomically pops from the list and reserves the Job, preventing race conditions.
- Use `queue:work` in production and `queue:listen` (or `horizon:listen`) only during local development for automatic code reloading.
- Always set `--max-jobs` or `--max-time` to force periodic worker restarts. PHP was not designed for long-running daemons, and memory leaks accumulate silently over thousands of Jobs.
- Always set `retry_after` longer than `--timeout`. If a Job is still running when `retry_after` expires, the queue hands it to another worker — causing duplicate processing.
- [Supervisor](http://supervisord.org/) keeps workers running in production. It restarts crashed workers automatically. Always set `stopwaitsecs` higher than your longest Job's timeout.
- [Laravel Horizon](https://laravel.com/docs/horizon) is a layer on top of `queue:work` that adds code-driven configuration, dynamic auto-scaling, a real-time dashboard, metrics, and notifications. It requires Redis.
- Horizon's config has two key concepts: **environments** (map to `APP_ENV` — different worker configs per deployment) and **supervisors** (logical groups of worker processes with their own queues, balancing strategies, and process limits).
- Horizon's `auto` balancing strategy dynamically allocates workers based on queue load. Use separate supervisors for queues that need guaranteed dedicated resources.
- Always restart workers during deployment. Use `php artisan queue:restart` for plain workers or `php artisan horizon:terminate` for Horizon. This is not optional.

## References

- [Queues](https://laravel.com/docs/queues) — Laravel Documentation
- [Horizon](https://laravel.com/docs/horizon) — Laravel Documentation
- [Pulse](https://laravel.com/docs/pulse) — Laravel Documentation
- [Deployment](https://laravel.com/docs/deployment) — Laravel Documentation
- [Supervisor Documentation](http://supervisord.org/) — Supervisor
- [Laravel Queues in Action](https://learn-laravel-queues.com/) — Mohamed Said
- [A Deep Dive Into Laravel Queues](https://www.honeybadger.io/blog/laravel-queues-deep-dive/) — Honeybadger
- [Laravel Queues Under the Hood](https://wendelladriel.com/blog/laravel-queues-under-the-hood) — Wendell Adriel
- [Queues and Workers in Production](https://martinjoo.dev/laravel-queues-and-workers-in-production) — Martin Joo
- [Laravel Jobs and Queues 101](https://laravel-news.com/laravel-jobs-and-queues-101) — Laravel News
- [Redis Lists](https://redis.io/docs/data-types/lists/) — Redis Documentation
- [Redis Sorted Sets](https://redis.io/docs/data-types/sorted-sets/) — Redis Documentation
- [BLPOP Command](https://redis.io/commands/blpop/) — Redis Documentation
- [PHP `serialize()` Function](https://www.php.net/manual/en/function.serialize.php) — PHP Documentation
- [PHP `unserialize()` Function](https://www.php.net/manual/en/function.unserialize.php) — PHP Documentation
- [Object Serialization](https://www.php.net/manual/en/language.oop5.serialization.php) — PHP Documentation
