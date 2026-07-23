# Models

Models are lightweight Query Builder subclasses.

A basic model only needs to define its database table.

```php
namespace App\Models;

use App\Systems\QueryBuilder;

final class User extends QueryBuilder
{
    protected string $defaultTable = 'users';
}
```

## Recommended Query Syntax

Use `query()` to create a fresh model query:

```php
$users = User::query()->get();
```

With conditions:

```php
$user = User::query()
    ->where('email', $email)
    ->first();
```

`query()` creates a fresh model instance:

```php
public static function query(): static
{
    return new static();
}
```

This has effectively the same object-creation cost as:

```php
new User();
```

Each `query()` call starts with fresh query state, preventing conditions, bindings, ordering or limits from being shared between separate query chains.

## Default Connection

Models use the connection configured in `database.default` unless the model defines `$defaultConnection`.

```php
namespace App\Models;

use App\Systems\QueryBuilder;

final class User extends QueryBuilder
{
    protected ?string $defaultConnection = 'sqlite';

    protected string $defaultTable = 'users';

    protected array $defaultSelect = [
        'name',
        'email',
    ];
}
```

The model now uses the `sqlite` connection automatically:

```php
$users = User::query()->get();
```

Pass a connection name to `query()` only when the current query must use another configured connection:

```php
$users = User::query('mysql')->get();
```

The explicit connection replaces `$defaultConnection` for that query.

When `$defaultConnection` is not defined or remains `null`, the model uses the application's default database connection.

## Default Selected Columns

By default, a model selects all columns:

```sql
SELECT * FROM users
```

A model may define `$defaultSelect` when only specific columns should be selected automatically.

```php
namespace App\Models;

use App\Systems\QueryBuilder;

final class User extends QueryBuilder
{
    protected string $defaultTable = 'users';

    protected array $defaultSelect = [
        'id',
        'name',
        'username',
        'email',
    ];
}
```

Now the model automatically selects the configured columns:

```php
$users = User::query()->get();
```

The generated query becomes:

```sql
SELECT id, name, username, email FROM users
```

There is no need to repeat the selected columns in the query chain.

```php
$users = User::query()
    ->where('status', 'active')
    ->order('name ASC')
    ->limit(20)
    ->get();
```

The generated query uses the model's `$defaultSelect` columns:

```sql
SELECT id, name, username, email
FROM users
WHERE status = ?
ORDER BY name ASC
LIMIT 20
```

## Default Database Connection

```php
namespace App\Models;

use App\Systems\QueryBuilder;

final class User extends QueryBuilder
{
    protected string $defaultTable = 'users';

    protected array $defaultConnection = 'sqlite';
}
```

## Method Chaining

Models support the complete Query Builder API.

```php
$users = User::query()
    ->where('status', 'active')
    ->whereNotNull('email_verified_at')
    ->order('created_at DESC')
    ->limit(20)
    ->get();
```

Find the first matching user:

```php
$user = User::query()
    ->where('email', $email)
    ->first();
```

Find a user by ID:

```php
$user = User::query()->find($id);
```

Check whether a user exists:

```php
$exists = User::query()
    ->where('email', $email)
    ->exists();
```

Count matching users:

```php
$total = User::query()
    ->where('status', 'active')
    ->count();
```

## Overriding Default Columns

The `select()` method is optional.

Use it only when the current query requires different columns from `$defaultSelect`.

```php
$users = User::query()
    ->select('id', 'email')
    ->get();
```

The selected columns are replaced for the current query:

```sql
SELECT id, email FROM users
```

They are not merged with `$defaultSelect`.

To explicitly select every column:

```php
$user = User::query()
    ->select('*')
    ->where('id', $id)
    ->first();
```

The next fresh model query uses `$defaultSelect` again:

```php
$users = User::query()->get();
```

## Protecting Sensitive Columns

`$defaultSelect` can prevent sensitive database columns from being returned accidentally.

```php
final class User extends QueryBuilder
{
    protected string $defaultTable = 'users';

    protected array $defaultSelect = [
        'id',
        'name',
        'username',
        'email',
        'created_at',
    ];
}
```

Sensitive columns remain excluded from normal queries:

```text
password
remember_token
verification_token
reset_token
```

Trusted application logic may still select them explicitly when required:

```php
$user = User::query()
    ->select('id', 'password')
    ->where('email', $email)
    ->first();
```

## Instance Syntax

A model may also be instantiated directly:

```php
$model = new User();

$users = $model
    ->where('status', 'active')
    ->get();
```

Although sequential use is supported after a terminal method resets the query state, `User::query()` is recommended because every query starts with a fresh model instance.

Avoid branching multiple unfinished queries from the same model instance:

```php
$model = new User();

$activeQuery = $model->where('status', 'active');
$adminQuery = $model->where('role', 'admin');
```

Both variables reference the same mutable query object.

Use separate model queries instead:

```php
$activeUsers = User::query()
    ->where('status', 'active')
    ->get();

$admins = User::query()
    ->where('role', 'admin')
    ->get();
```

## Adding Model Methods

Models may contain reusable query methods.

```php
namespace App\Models;

use App\Systems\QueryBuilder;

final class User extends QueryBuilder
{
    protected string $defaultTable = 'users';

    protected array $defaultSelect = [
        'id',
        'name',
        'username',
        'email',
    ];

    public function active(): self
    {
        return $this->where('status', 'active');
    }
}
```

Use the model method as part of the query chain:

```php
$users = User::query()
    ->active()
    ->order('name ASC')
    ->get();
```

A model method may accept arguments:

```php
public function registeredAfter(string $date): self
{
    return $this->where('created_at', '>=', $date);
}
```

Usage:

```php
$users = User::query()
    ->registeredAfter('2026-01-01')
    ->where('status', 'active')
    ->order('created_at DESC')
    ->get();
```

Model methods may also be combined:

```php
public function verified(): self
{
    return $this->whereNotNull('email_verified_at');
}
```

```php
$users = User::query()
    ->active()
    ->verified()
    ->order('name ASC')
    ->limit(50)
    ->get();
```

## Model Behavior

Pkathamo models provide:

- a fixed database table
- optional default selected columns
- reusable query methods
- fresh query state through `query()`
- method chaining
- access to the complete Query Builder API

Pkathamo models do not provide:

- ORM entity lifecycle
- dirty attribute tracking
- automatic relationships
- model events
- automatic attribute casting
- hidden database queries

Returned database rows are PDO objects.