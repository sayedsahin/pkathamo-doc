# Database

Pkathamo uses PDO and supports multiple named database connections.

The `Database` service is configured as a container singleton. PDO connections are created lazily and cached by connection name, so each configured connection is opened only when it is first requested.

Supported drivers:

- MySQL
- PostgreSQL
- SQLite

## Configuration

Database connections are configured in:

```text
config/database.php
```

```php
return [
    'default' => env('DB_CONNECTION', 'mysql'),

    'connections' => [
        'mysql' => [
            'driver' => 'mysql',
            'host' => env('MYSQL_DB_HOST', env('DB_HOST', '127.0.0.1')),
            'port' => (int) env('MYSQL_DB_PORT', env('DB_PORT', 3306)),
            'database' => env('MYSQL_DB_NAME', env('DB_NAME', '')),
            'username' => env('MYSQL_DB_USERNAME', env('DB_USERNAME', 'root')),
            'password' => env('MYSQL_DB_PASSWORD', env('DB_PASSWORD', '')),
            'charset' => env('MYSQL_DB_CHARSET', 'utf8mb4'),
            'options' => [
                'persistent' => env('MYSQL_DB_PERSISTENT', env('DB_PERSISTENT', false)),
            ],
        ],

        'pgsql' => [
            'driver' => 'pgsql',
            'host' => env('PGSQL_DB_HOST', env('DB_HOST', '127.0.0.1')),
            'port' => (int) env('PGSQL_DB_PORT', env('DB_PORT', 5432)),
            'database' => env('PGSQL_DB_NAME', env('DB_NAME', '')),
            'username' => env('PGSQL_DB_USERNAME', env('DB_USERNAME', 'postgres')),
            'password' => env('PGSQL_DB_PASSWORD', env('DB_PASSWORD', '')),
            'options' => [
                'persistent' => env('PGSQL_DB_PERSISTENT', env('DB_PERSISTENT', false)),
            ],
        ],

        'sqlite' => [
            'driver' => 'sqlite',
            'database' => env('SQLITE_DATABASE', ROOT_PATH . '/database/database.sqlite'),
            'foreign_keys' => env('SQLITE_FOREIGN_KEYS', true),
            'busy_timeout' => (int) env('SQLITE_BUSY_TIMEOUT', 5000),
            'journal_mode' => env('SQLITE_JOURNAL_MODE', null),
            'synchronous' => env('SQLITE_SYNCHRONOUS', null),
        ],
    ],
];
```

## Default Connection

The default connection is selected by:

```php
'default' => env('DB_CONNECTION', 'mysql'),
```

Example:

```dotenv
DB_CONNECTION=mysql
```

The default connection is used when no connection name is passed:

```php
$users = db()
    ->table('users')
    ->get();
```

```php
$users = User::query()->get(); // model
```

## Named Connections

Pass a configured connection name only when a query must use another database:

```php
$users = db('pgsql')
    ->table('users')
    ->get();
```

```php
$users = User::query('pgsql')
    ->where('status', 'active')
    ->get(); // model
```

The connection name identifies an entry under `database.connections`. It does not have to match the driver name.

```php
'connections' => [
    'primary' => [
        'driver' => 'mysql',
        // ...
    ],

    'reporting' => [
        'driver' => 'mysql',
        // ...
    ],

    'analytics' => [
        'driver' => 'pgsql',
        // ...
    ],
],
```

In this example, `primary`, `reporting` and `analytics` are connection names, while `mysql` and `pgsql` are drivers.

## Connection Lifecycle

The `Database` service stores PDO connections by name.

```text
First request for a connection
→ Validate its configuration
→ Create the PDO instance
→ Cache it by connection name

Later request for the same connection
→ Reuse the cached PDO instance
```

Configuring multiple connections does not open all of them during application bootstrap. Only requested connections are created.

The `db()` helper reuses the singleton `Database` service but returns a fresh Query Builder for every call.

## Direct PDO Access

Resolve the `Database` service through constructor injection:

```php
namespace App\Services;

use App\Systems\Database;
use PDO;

final class ImportService
{
    public function __construct(private Database $database)
    {
    }

    public function connection(): PDO
    {
        return $this->database->connection();
    }
}
```

Resolve it directly from the container when necessary:

```php
use App\Systems\Database;

global $container;

$database = $container->make(Database::class);
$pdo = $database->connection();
```

A named PDO connection may be requested explicitly:

```php
$pdo = $database->connection('pgsql');
```

The configured driver is available through:

```php
$driver = $database->driver();
```

Possible values are:

```text
mysql
pgsql
sqlite
```

## Transactions

Pkathamo provides connection-specific transaction methods through the `db()` helper.

The recommended approach is the callback-based `transaction()` method. It starts a transaction, commits when the callback completes successfully, and rolls back when the callback throws an exception.

```php
$userId = db('mysql')->transaction(function () use ($user, $roleId) {
    $userId = db('mysql')
        ->table('users')
        ->insert($user, true);

    db('mysql')
        ->table('user_roles')
        ->insert([
            'user_id' => $userId,
            'role_id' => $roleId,
        ]);

    return $userId;
});
```

The value returned by the callback is returned by `transaction()`.

When no connection name is provided, the default connection is used:

```php
$userId = db()->transaction(function () use ($user) {
    return db()
        ->table('users')
        ->insert($user, true);
});
```

### Named Connection Transactions

A transaction belongs to one configured connection:

```php
db('sqlite')->transaction(function () {
    db('sqlite')
        ->table('logs')
        ->insert([
            'message' => 'Import started',
        ]);

    db('sqlite')
        ->table('settings')
        ->where('key', 'import_status')
        ->update([
            'value' => 'running',
        ]);
});
```

All Query Builders and Models participating in the transaction must use the same connection name.

```php
db('pgsql')->transaction(function () {
    User::query('pgsql')->insert([
        'name' => 'Rahim',
        'email' => 'rahim@example.com',
    ]);

    Profile::query('pgsql')->insert([
        'user_id' => 1,
        'bio' => 'Example profile',
    ]);
});
```

Models participate in the same transaction because the singleton `Database` service reuses the same PDO instance for the selected connection.

### Manual Transactions

Manual transaction control is also available:

```php
$db = db('mysql');

$db->beginTransaction();

try {
    $userId = db('mysql')
        ->table('users')
        ->insert($user, true);

    db('mysql')
        ->table('user_roles')
        ->insert([
            'user_id' => $userId,
            'role_id' => $roleId,
        ]);

    $db->commit();
} catch (Throwable $exception) {
    $db->rollBack();

    throw $exception;
}
```

Available methods:

```php
$db->beginTransaction();
$db->commit();
$db->rollBack();
$db->inTransaction();
$db->transaction($callback);
```

The callback-based method is preferred because it guarantees that rollback handling is not accidentally omitted.


```php
db('mysql')->transaction(function () {
    try {
        // Failed operation
    } catch (Throwable $exception) {
        // Rethrow when rollback is required.
        throw $exception;
    }
});
```

### Nested Transactions

Nested transactions are not supported.

Do not call `transaction()` for the same connection while that connection already has an active transaction:

```php
db('mysql')->transaction(function () {
    db('mysql')->transaction(function () {
        // Not supported
    });
});
```

Keep related operations inside one transaction callback instead.

## MySQL Environment

```dotenv
DB_CONNECTION=mysql

MYSQL_DB_HOST=127.0.0.1
MYSQL_DB_PORT=3306
MYSQL_DB_NAME=pkathamo
MYSQL_DB_USERNAME=root
MYSQL_DB_PASSWORD=
MYSQL_DB_CHARSET=utf8mb4
MYSQL_DB_PERSISTENT=false
```

Generic fallback variables are also supported:

```dotenv
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=pkathamo
DB_USERNAME=root
DB_PASSWORD=
DB_PERSISTENT=false
```

MySQL-specific variables take priority over the generic fallback variables.

## PostgreSQL Environment

```dotenv
DB_CONNECTION=pgsql

PGSQL_DB_HOST=127.0.0.1
PGSQL_DB_PORT=5432
PGSQL_DB_NAME=pkathamo
PGSQL_DB_USERNAME=postgres
PGSQL_DB_PASSWORD=
PGSQL_DB_PERSISTENT=false
```

## SQLite Environment

```dotenv
DB_CONNECTION=sqlite

SQLITE_DATABASE=/absolute/path/to/database.sqlite
SQLITE_FOREIGN_KEYS=true
SQLITE_BUSY_TIMEOUT=5000
SQLITE_JOURNAL_MODE=WAL
SQLITE_SYNCHRONOUS=NORMAL
```

An in-memory SQLite database may be used for isolated tests:

```dotenv
SQLITE_DATABASE=:memory:
```

### Foreign Keys

SQLite foreign-key enforcement is enabled by default.

```dotenv
SQLITE_FOREIGN_KEYS=true
```

### Busy Timeout

The busy timeout is expressed in milliseconds.

```dotenv
SQLITE_BUSY_TIMEOUT=5000
```

### Journal Mode

Supported values:

```text
DELETE
TRUNCATE
PERSIST
MEMORY
WAL
OFF
```

### Synchronous Mode

Supported values:

```text
OFF
NORMAL
FULL
EXTRA
```

## PDO Configuration

Pkathamo configures PDO with exception-based error handling:

```php
PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION
```

The default fetch mode is:

```php
PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
```

MySQL and PostgreSQL also use native prepared statements:

```php
PDO::ATTR_EMULATE_PREPARES => false
```

Persistent connections are configured separately for each MySQL or PostgreSQL connection:

```php
'options' => [
    'persistent' => false,
],
```

Enable persistent connections only after testing them in the target hosting environment.

## Connection Errors

Requesting an unknown connection throws an exception:

```text
Database connection [reporting] is not configured.
```

An empty connection name is invalid:

```text
Database connection name cannot be empty.
```

Unsupported drivers are rejected. Supported driver names are `mysql`, `pgsql` and `sqlite`.

## Schema Constraints

Application validation does not replace database constraints.

Use database-level constraints where appropriate, including:

- unique indexes for unique email addresses and usernames
- composite unique indexes for unique user-role pairs
- foreign keys for relational integrity
- appropriate indexes for frequently filtered or joined columns
