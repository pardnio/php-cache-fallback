# php-cache-fallback - 技術文件

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 前置需求

- PHP 8.0 以上
- Composer
- 提供 `isConnected()`、`get($db, $key)`、`set($db, $key, $value, $expire)` 的 Redis 客戶端物件，例如 [`pardnchiu/redis-cli`](https://github.com/pardnio/php-redis-cli) 的 `pardnchiu\RDB`
- `$_SERVER["DOCUMENT_ROOT"]/storage/caches` 可建立且可寫入（檔案備援使用）

## 安裝

### 使用 Composer

```bash
composer require pardnchiu/cache
composer require pardnchiu/redis-cli
```

### 從原始碼

```bash
git clone https://github.com/pardnio/php-cache-fallback.git
cd php-cache-fallback
composer install
```

套件以 PSR-4 將 `PD\` 對應至 `src/`。

## 設定

### 儲存位置

| 模式 | 位置 | 說明 |
|------|------|------|
| Redis | DB `1` | 鍵為 `md5($page)`，到期交由 Redis `EXPIRE` |
| 檔案 | `$_SERVER["DOCUMENT_ROOT"]/storage/caches/<md5>.json` | 目錄不存在時以 `0777` 建立；內容為 `{"content": ..., "expire": <unix time>}` |

Redis 連線參數由所傳入的客戶端自行讀取；使用 `pardnchiu\RDB` 時見其 `REDIS_HOST`／`REDIS_PORT`／`REDIS_PASSWORD`。

## 使用方式

### 基礎

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

Redis 斷線時 `RDB::isConnected()` 回傳 `false`，同一段程式自動改走檔案快取。

### 進階：處理寫入失敗與排程清理

檔案模式下目錄無法建立或不可寫入時，`get()` 與 `set()` 皆回傳 `null`；呼叫端應保留原始內容作為輸出。

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

排程清理過期檔案（cron 以 CLI 執行時需自行設定 `DOCUMENT_ROOT`）：

```php
<?php

require __DIR__ . '/vendor/autoload.php';

$_SERVER["DOCUMENT_ROOT"] = '/var/www/html';

(new PD\Cache(new pardnchiu\RDB()))->clean();
```

## API 參考

### Cache

| 方法 | 簽章 | 說明 |
|------|------|------|
| 建構子 | `__construct($REDIS_CLIENT = null)` | 注入 Redis 客戶端；每次讀寫都會呼叫其 `isConnected()`，因此必須傳入物件 |
| `get` | `get(string $page): mixed` | Redis 模式回傳客戶端 `get(1, md5($page))` 的結果；檔案模式回傳內容，不存在、過期或目錄不可用時回傳 `null` |
| `set` | `set(string $page, string $content, int $expire): ?string` | 壓縮 HTML 後寫入，`$expire` 為秒數；回傳壓縮後內容，檔案目錄不可用時回傳 `null` |
| `clean` | `clean(): void` | 刪除快取目錄中所有過期的 JSON 檔；Redis 項目不受影響 |

### 寫入前壓縮規則

| 規則 | 正規式 |
|------|--------|
| 移除換行與其後縮排 | `/\n[ ]*/` → `""` |
| 移除標籤間空白 | `/>[ ]*</` → `"><"` |

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
