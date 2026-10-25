# php-cache-fallback - Architecture

Last updated: 2026-10-25

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    A[Caller] --> B[PD\Cache]
    R[Redis client e.g. pardnchiu\RDB] --> B
    B --> C{isConnected?}
    C -->|Yes| D[Redis DB 1]
    C -->|No| E[DOCUMENT_ROOT/storage/caches]
    E --> F[md5.json files]
    B --> G[clean expired]
    G --> E
```

## Module: Cache

`PD\Cache` asks the injected client whether it is connected on every operation, then takes the Redis or file path.

```mermaid
graph TB
    subgraph Cache
        K[md5 key] --> SW{isConnected?}
        SW -->|Yes| RG[client->get / set on DB 1]
        SW -->|No| DIR[Ensure directory exists and is writable]
        DIR -->|Fail| NUL[Return null]
        DIR -->|OK| FS[Read / write JSON file]
        FS --> EXP{Expired?}
        EXP -->|Yes| DEL[unlink and return null]
        EXP -->|No| OUT[Return content]
        MIN[HTML minify] --> K
    end
    IN[set call] --> MIN
    GET[get call] --> K
```

## Data Flow

```mermaid
sequenceDiagram
    participant Caller
    participant Cache
    participant Redis
    participant FileSystem
    Caller->>Cache: get(page)
    Cache->>Redis: isConnected()
    alt connected
        Cache->>Redis: get(1, md5)
        Redis-->>Cache: content / false
    else disconnected
        Cache->>FileSystem: read md5.json
        FileSystem-->>Cache: content + expire
        Cache->>Cache: delete file if expired
    end
    Cache-->>Caller: content / null
    Caller->>Cache: set(page, html, ttl)
    Cache->>Cache: minify HTML
    alt connected
        Cache->>Redis: set(1, md5, html, ttl)
    else disconnected
        Cache->>FileSystem: write {content, expire}
    end
    Cache-->>Caller: minified content / null
```

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
