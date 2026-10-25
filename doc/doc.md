# php-cache-fallback - Documentation

Last updated: 2026-10-25

> Back to [README](../README.md)

## Prerequisites

- PHP 8.0 or higher
- Composer
- A Redis client object exposing `isConnected()`, `get($db, $key)`, and `set($db, $key, $value, $expire)`, such as `pardnchiu\RDB` from [`pardnchiu/redis-cli`](https://github.com/pardnio/php-redis-cli)
- `$_SERVER["DOCUMENT_ROOT"]/storage/caches` creatable and writable (used by the file fallback)

## Installation

### Using Composer

```bash
composer require pardnchiu/cache
composer require pardnchiu/redis-cli
```

### From Source

```bash
git clone https://github.com/pardnio/php-cache-fallback.git
cd php-cache-fallback
composer install
```

The package maps `PD\` to `src/` via PSR-4.

## Configuration

### Storage Locations

| Mode | Location | Notes |
|------|----------|-------|
| Redis | DB `1` | Key is `md5($page)`; expiry handled by Redis `EXPIRE` |
| File | `$_SERVER["DOCUMENT_ROOT"]/storage/caches/<md5>.json` | Directory created with `0777` when missing; content is `{"content": ..., "expire": <unix time>}` |

The injected client reads its own Redis connection settings; with `pardnchiu\RDB` see `REDIS_HOST` / `REDIS_PORT` / `REDIS_PASSWORD`.

## Usage

### Basic

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use PD\Cache;
use pardnchiu\RDB;

$cache = new Cache(new RDB());

$key = $_SERVER["REQUEST_URI"];
$html = $cache->get($key);

if ($html === null || $html === false) {
    ob_start();
    include __DIR__ . '/views/products.php';
    $html = $cache->set($key, ob_get_clean(), 3600);
}

echo $html;
```

When Redis is down, `RDB::isConnected()` returns `false` and the same code switches to the file cache.

### Advanced: Write Failures and Scheduled Cleanup

In file mode, `get()` and `set()` both return `null` when the directory cannot be created or written; keep the original content as output.

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use PD\Cache;
use pardnchiu\RDB;

$cache = new Cache(new RDB());

$rendered = '<div>  <p>Hello</p>  </div>';
$stored = $cache->set('/hello', $rendered, 600);

if ($stored === null) {
    error_log('[Unavailable] cache: storage/caches not writable');
    $stored = $rendered;
}

echo $stored;
```

Scheduled cleanup of expired files (set `DOCUMENT_ROOT` yourself when cron runs it via CLI):

```php
<?php

require __DIR__ . '/vendor/autoload.php';

$_SERVER["DOCUMENT_ROOT"] = '/var/www/html';

(new PD\Cache(new pardnchiu\RDB()))->clean();
```

## API Reference

### Cache

| Method | Signature | Description |
|--------|-----------|-------------|
| Constructor | `__construct($REDIS_CLIENT = null)` | Injects the Redis client; every read and write calls its `isConnected()`, so an object must be passed |
| `get` | `get(string $page): mixed` | Redis mode returns the client's `get(1, md5($page))`; file mode returns the content, or `null` when missing, expired, or the directory is unusable |
| `set` | `set(string $page, string $content, int $expire): ?string` | Minifies and stores the content for `$expire` seconds; returns the minified content, or `null` when the file directory is unusable |
| `clean` | `clean(): void` | Deletes every expired JSON file in the cache directory; Redis entries are untouched |

### Minification Rules

| Rule | Pattern |
|------|---------|
| Strip line breaks and following indentation | `/\n[ ]*/` → `""` |
| Strip whitespace between tags | `/>[ ]*</` → `"><"` |

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
