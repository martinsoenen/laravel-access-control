---
name: laravel-access-control
description: "Centralizes authorization in Laravel via Lomkit Access Control. Activates when building Policies, restricting Eloquent queries by tenant/role/ownership, generating make:control or make:perimeter classes, writing PostControl/UserControl-style classes, defining Perimeters (global, client, overlay), wiring ControlledPolicy, using the controlled()/uncontrolled() query macros, integrating access control with Laravel Scout searches, or whenever the user mentions access control, perimeters, controls, or row-level security in a Laravel project using lomkit/laravel-access-control. Use this skill whenever authorization logic and query-scoping must stay in sync."
license: MIT
metadata:
  author: lomkit
---
# Laravel Access Control

Lomkit Access Control unifies **policy checks** and **query scoping** in a single place. One `Control` class per model declares a list of `Perimeter`s; each perimeter answers three questions for a given user:

- `allowed(user, method)` — does this user have permission for `view`/`create`/`update`/`delete`/... in this perimeter?
- `should(user, model)` — does an existing model instance fall within this perimeter for this user?
- `query(builder, user)` — how should an Eloquent query be restricted so it only returns models inside this perimeter?

A `ControlledPolicy` then exposes the standard Gate methods (`view`, `viewAny`, `create`, `update`, `delete`, `restore`, `forceDelete`), and the `HasControl` trait adds a `controlled()` macro on Eloquent (and Scout) builders so the same perimeters scope queries.

## Activation Check

Before applying this skill, confirm the package is installed:

1. Look for `lomkit/laravel-access-control` in the project's `composer.json` (`require` or `require-dev`).
2. If missing, do **not** invent the APIs below — instead suggest `composer require lomkit/laravel-access-control` and stop.
3. If present, also check whether `config/access-control.php` exists. If not, recommend publishing it:

   ```bash
   php artisan vendor:publish --tag=access-control-config
   ```

The full reference docs live at https://laravel-access-control.lomkit.com — point the user there for anything not covered below.

> **Beta status.** The upstream documentation explicitly warns that Laravel Access Control is in Beta and the API may change. Flag this to the user before recommending the package for a production project.

## When to Apply

Activate this skill when the user:

- Is writing or modifying a `Policy` for a model and wants authorization centralized
- Needs row-level filtering (multi-tenant, per-client, per-team, ownership) on Eloquent queries
- Runs (or asks about) `php artisan make:control` or `php artisan make:perimeter`
- Mentions `Control`, `Perimeter`, `OverlayPerimeter`, `ControlledPolicy`, `HasControl`, `controlled()`, `uncontrolled()`
- Wants `Model::search(...)` (Laravel Scout) to respect the same authorization rules
- Wants to toggle whether access-control scoping is applied by default to all queries

## Core Concepts

### 1. Perimeter

A `Perimeter` is a fluent definition with four optional closures. Defaults: `allowed` returns `true`, `should` returns `true`, `query`/`scoutQuery` return the builder unchanged.

```php
use Lomkit\Access\Perimeters\Perimeter;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

Perimeter::new()
    ->allowed(fn (Model $user, string $method) => $user->can("{$method} client models"))
    ->should(fn (Model $user, Model $model)    => $model->client_id === $user->client_id)
    ->query(fn (Builder $query, Model $user)   => $query->where('client_id', $user->client_id))
    ->scoutQuery(fn (\Laravel\Scout\Builder $q, Model $user) => $q->where('client_id', $user->client_id));
```

### 2. OverlayPerimeter

An ordinary `Perimeter` short-circuits: as soon as one matches (`allowed` true and `should` true), evaluation stops and its `query` wins.

An `OverlayPerimeter` **combines** with other perimeters instead of replacing them — useful when multiple grants should be additive (e.g. a user is both in the `client` perimeter *and* has a `shared with me` perimeter).

```php
use Lomkit\Access\Perimeters\OverlayPerimeter;

class SharedPerimeter extends OverlayPerimeter
{
    // overlays() returns true automatically
}
```

Behavior summary (`Control::applyQueryControl` and `Control::applies`):

- Non-overlay perimeter matches → its query is applied and iteration stops.
- Overlay perimeter matches → its query is OR-combined with later matching perimeters.
- No perimeter matches → the query is forced to return nothing (`whereRaw('0=1')` for Eloquent, an impossible field for Scout).

### 3. Control

One `Control` per model. It declares the target `$model` and returns an ordered array of perimeters. Order matters: the first non-overlay match wins.

```php
namespace App\Access\Controls;

use Lomkit\Access\Controls\Control;
use Lomkit\Access\Perimeters\Perimeter;
use App\Access\Perimeters\GlobalPerimeter;
use App\Access\Perimeters\ClientPerimeter;
use App\Models\Post;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class PostControl extends Control
{
    protected string $model = Post::class;

    protected function perimeters(): array
    {
        return [
            GlobalPerimeter::new()
                ->allowed(fn (Model $u, string $m) => $u->can("{$m} global posts"))
                ->should(fn (Model $u, Model $post) => true)
                ->query(fn (Builder $q, Model $u) => $q),

            ClientPerimeter::new()
                ->allowed(fn (Model $u, string $m) => $u->can("{$m} client posts"))
                ->should(fn (Model $u, Model $post) => $post->client_id === $u->client_id)
                ->query(fn (Builder $q, Model $u) => $q->where('client_id', $u->client_id))
                ->scoutQuery(fn (\Laravel\Scout\Builder $q, Model $u) => $q->where('client_id', $u->client_id)),
        ];
    }
}
```

Controls are auto-discovered at boot in `app/Access/Controls` (override with `Access::$controlDiscoveryPaths` from a service provider if you keep them elsewhere).

### 4. HasControl trait

Adds two query macros to the model and exposes its `Control`:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Lomkit\Access\Controls\HasControl;

class Post extends Model
{
    use HasControl;
}
```

Then:

```php
Post::controlled()->get();    // applies the perimeter queries for Auth::user()
Post::uncontrolled()->get();  // bypasses access control (use sparingly: admin tools, jobs, seeds)
```

If `access-control.queries.enabled_by_default` is `true`, every query is scoped automatically and `uncontrolled()` is the explicit escape hatch.

### 5. ControlledPolicy

A drop-in policy that delegates every Gate method to the Control:

```php
namespace App\Policies;

use App\Access\Controls\PostControl;
use Lomkit\Access\Policies\ControlledPolicy;

class PostPolicy extends ControlledPolicy
{
    protected string $control = PostControl::class;
}
```

Register it like any other policy (auto-discovery works if your namespaces line up). After that:

```php
$user->can('view', $post);   // → PostControl perimeters with method='view'
$user->can('viewAny', Post::class);
Gate::authorize('update', $post);
```

`viewAny` and `create` are evaluated with a fresh (non-existing) model instance, so only `allowed` runs — `should` is skipped because there is no instance to check.

## Installation & Setup

### 1. Require the package

```bash
composer require lomkit/laravel-access-control
```

The `AccessServiceProvider` is auto-registered.

### 2. Publish the config (optional but recommended)

```bash
php artisan vendor:publish --tag=access-control-config
```

This creates `config/access-control.php`:

```php
return [
    'queries' => [
        'enabled_by_default'        => false, // true => every query is auto-scoped
        'isolate_parent_query'      => true,  // wrap perimeter logic in a parent where(...)
        'isolate_perimeter_queries' => true,  // each perimeter wrapped in orWhere(...) so overlays don't collide
    ],
    'methods' => [
        'viewAny'     => 'view',
        'view'        => 'view',
        'create'      => 'create',
        'update'      => 'update',
        'delete'      => 'delete',
        'restore'     => 'restore',
        'forceDelete' => 'forceDelete',
    ],
];
```

The `methods` map lets you alias Gate methods to access-control "verbs". By default `viewAny` is mapped to `view`, so a perimeter that allows `view` automatically allows `viewAny`.

### 3. Generate classes with the Artisan commands

```bash
php artisan make:perimeter GlobalPerimeter            # plain perimeter
php artisan make:perimeter SharedPerimeter --overlay  # overlay perimeter

php artisan make:control PostControl --model=Post --perimeters=GlobalPerimeter --perimeters=ClientPerimeter
```

Without flags, both commands prompt interactively. Generated files land in `app/Access/Perimeters` and `app/Access/Controls`.

### 4. Add the trait + policy

```php
// app/Models/Post.php
class Post extends Model { use \Lomkit\Access\Controls\HasControl; }

// app/Policies/PostPolicy.php
class PostPolicy extends \Lomkit\Access\Policies\ControlledPolicy
{
    protected string $control = \App\Access\Controls\PostControl::class;
}
```

## Laravel Scout Integration

When `laravel/scout` is installed, the service provider registers a `controlled()` macro on `Laravel\Scout\Builder`. It mirrors the Eloquent macro but goes through each perimeter's `scoutQuery` closure:

```php
Post::search('hello')->controlled()->get();
Post::search('hello')->controlled()->paginate(15);
```

Important constraints (these are Scout limitations, not access-control bugs):

- Scout `where`/`whereIn` only support equality-style filters. Translate `query()` logic that uses `whereHas`, `orWhereNotNull`, etc. into something Scout can express — or denormalize the attribute into `toSearchableArray()` so it can be filtered.
- If no perimeter matches, the package adds an impossible filter (`->where('__NOT_A_VALID_FIELD__', 0)`) to force an empty result set, mirroring the Eloquent `0=1` fallback.
- Make sure any field used in `scoutQuery` is exposed by the model's `toSearchableArray()` and indexed.

If Scout is not installed, the macro is simply not registered — the rest of the package works unchanged.

## Query Isolation Behavior

The two `queries.isolate_*` flags exist because Eloquent `where` chains and overlays can interfere with the caller's own `where` clauses. Examples:

- `isolate_parent_query: true` → final SQL looks like `... WHERE (<all perimeter conditions>) AND <caller's wheres>`. This keeps perimeter ORs from leaking out and accidentally widening the result set.
- `isolate_perimeter_queries: true` → each overlay perimeter is wrapped in its own `orWhere(fn ($q) => ...)`. Without this, two perimeters that each add their own `where`s would AND together instead of OR-ing.

Leave both at `true` unless you have a concrete reason to change them.

## Common Patterns

### Multi-tenant by user attribute

```php
ClientPerimeter::new()
    ->allowed(fn (User $u, string $m) => $u->can("{$m} posts"))
    ->should(fn (User $u, Post $p)    => $p->client_id === $u->client_id)
    ->query(fn (Builder $q, User $u)  => $q->where('client_id', $u->client_id));
```

### Ownership (current user only)

```php
OwnerPerimeter::new()
    ->allowed(fn ($u, $m) => true)
    ->should(fn ($u, $post) => $post->user_id === $u->getKey())
    ->query(fn ($q, $u)    => $q->where('user_id', $u->getKey()));
```

### Admin bypass via overlay

```php
class AdminPerimeter extends OverlayPerimeter
{
    // overlays() === true, so admin grants stack with other perimeters
}

AdminPerimeter::new()
    ->allowed(fn (User $u, string $m) => $u->hasRole('admin'))
    ->should(fn ($u, $m) => true)
    ->query(fn ($q, $u)  => $q); // no restriction = admin sees everything
```

Put admin **first** in the array if it's overlay-only; the iteration will OR its empty restriction with the rest.

### Bypassing access control intentionally

```php
Post::uncontrolled()->get();              // skip for one query
Auth::guard('api')->forgetUser();          // no auth user → perimeters typically fail
```

For artisan commands, queued jobs, or seeders that should run unrestricted, always use `->uncontrolled()` explicitly so the intent is visible.

## Common Pitfalls

- **No perimeter matches → empty result.** If a query suddenly returns zero rows after wiring up control, the auth user probably doesn't satisfy any `allowed()`. The Control intentionally appends `0=1` instead of returning everything.
- **Controls may run twice in `index`-style endpoints.** A typical controller action calls `$this->authorize('viewAny', Post::class)` (Policy → `allowed` for `view` due to the `viewAny → view` mapping) *and* `Post::controlled()->get()` (Query → `allowed` for `view` again). This is expected; the docs call it out. If `allowed()` is expensive, cache it on the user instance.
- **`should` runs only for existing models.** `viewAny`/`create` skip `should` because there's no instance. Don't put existence-dependent checks in `should` and expect them to gate creation — use `allowed` for that.
- **Order matters with non-overlay perimeters.** First match wins and short-circuits the rest. Put more permissive grants (e.g. admin) before more restrictive ones, *or* make them overlay perimeters.
- **`scoutQuery` ≠ `query`.** Scout cannot run arbitrary Eloquent closures. If `query()` uses `whereHas`, you must implement `scoutQuery()` separately (often by denormalizing the relation into the indexed payload).
- **`enabled_by_default: true` affects every query.** Including the ones in tests and seeders. Wrap fixtures in `->uncontrolled()` or set the config to `false` in `phpunit.xml`.
- **Custom Control discovery path.** If you keep Controls outside `app/Access/Controls`, set `Access::$controlDiscoveryPaths = [...]` in a service provider's `register()` — controls discovered too late won't be registered.
- **Policy registration.** `ControlledPolicy` still has to be associated with the model the normal way (Laravel's policy auto-discovery, or `Gate::policy(Post::class, PostPolicy::class)`).
- **`viewAny` mapping.** Because `config('access-control.methods.viewAny') === 'view'`, a perimeter that only grants `create` will not satisfy `viewAny`. Adjust the map if you want stricter separation.

## Quick Reference

| Class / API | Purpose |
|---|---|
| `Lomkit\Access\Perimeters\Perimeter` | Fluent builder: `allowed`, `should`, `query`, `scoutQuery`. Non-overlay (short-circuits). |
| `Lomkit\Access\Perimeters\OverlayPerimeter` | Like `Perimeter` but combines with other perimeters via OR. |
| `Lomkit\Access\Controls\Control` | One per model; declares `$model` and `perimeters()`. |
| `Lomkit\Access\Controls\HasControl` (trait) | Adds `controlled()`/`uncontrolled()` macros + `newControl()`. |
| `Lomkit\Access\Policies\ControlledPolicy` | Base policy delegating to the model's Control. |
| `Lomkit\Access\Access` | Registry used internally for control discovery (`controlForModel`, `addControl`, `discoverControls`, `discoverControlsWithin`). |
| `php artisan make:control` | Generate a Control (with `--model`, `--perimeters`). |
| `php artisan make:perimeter` | Generate a Perimeter (with `--overlay`). |
| `Model::controlled()` | Apply perimeter queries to an Eloquent query. |
| `Model::uncontrolled()` | Explicitly bypass perimeter queries. |
| `Model::search(...)->controlled()` | Same, on a Scout Builder (requires `laravel/scout`). |
| `config('access-control.queries.enabled_by_default')` | Apply `controlled()` to every query implicitly. |
| `config('access-control.methods.*')` | Map Gate method names to access-control verbs. |

For anything beyond this overview, read the docs at https://laravel-access-control.lomkit.com.
