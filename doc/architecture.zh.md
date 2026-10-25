# php-cache-fallback - 架構

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    A[呼叫端] --> B[PD\Cache]
    R[Redis 客戶端 如 pardnchiu\RDB] --> B
    B --> C{isConnected?}
    C -->|是| D[Redis DB 1]
    C -->|否| E[DOCUMENT_ROOT/storage/caches]
    E --> F[md5.json 檔]
    B --> G[clean 過期清理]
    G --> E
```

## Module: Cache

`PD\Cache` 每次操作都先詢問注入的客戶端是否連線，再決定走 Redis 或檔案路徑。

```mermaid
graph TB
    subgraph Cache
        K[md5 鍵值] --> SW{isConnected?}
        SW -->|是| RG[client->get / set 於 DB 1]
        SW -->|否| DIR[確認目錄存在且可寫]
        DIR -->|失敗| NUL[回傳 null]
        DIR -->|成功| FS[讀寫 JSON 檔]
        FS --> EXP{已過期?}
        EXP -->|是| DEL[unlink 並回傳 null]
        EXP -->|否| OUT[回傳 content]
        MIN[HTML 壓縮] --> K
    end
    IN[set 呼叫] --> MIN
    GET[get 呼叫] --> K
```

## Data Flow

```mermaid
sequenceDiagram
    participant 呼叫端
    participant Cache
    participant Redis
    participant 檔案系統
    呼叫端->>Cache: get(page)
    Cache->>Redis: isConnected()
    alt 已連線
        Cache->>Redis: get(1, md5)
        Redis-->>Cache: 內容 / false
    else 未連線
        Cache->>檔案系統: 讀取 md5.json
        檔案系統-->>Cache: content + expire
        Cache->>Cache: 過期則刪檔
    end
    Cache-->>呼叫端: 內容 / null
    呼叫端->>Cache: set(page, html, ttl)
    Cache->>Cache: 壓縮 HTML
    alt 已連線
        Cache->>Redis: set(1, md5, html, ttl)
    else 未連線
        Cache->>檔案系統: 寫入 {content, expire}
    end
    Cache-->>呼叫端: 壓縮後內容 / null
```

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
