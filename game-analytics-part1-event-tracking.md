# Game Analytics Part 1: Client-Side Event Tracking Architecture

**Building a Robust, Offline-Capable Event Collection System for Mobile Games**

This is Part 1 of a 3-part series on building AI-powered game analytics:
- **Part 1: Client-Side Event Tracking** (this post)
- [Part 2: Streaming Pipeline - Kafka to BigQuery](./game-analytics-part2-data-pipeline)
- [Part 3: ML-Powered Game Analytics](./game-analytics-part3-ml-predictions)

---

## Overview

Every successful mobile game relies on understanding player behavior. This requires a robust event tracking system that can:

- Collect events reliably, even with poor network connectivity
- Handle millions of events per day without impacting game performance
- Ensure no data loss during app crashes or network failures
- Scale across iOS, Android, and web platforms

This post explores the architecture of a production-grade event tracking SDK.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MOBILE GAME CLIENT                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │   Game       │    │   Event      │    │   Tracking   │                  │
│  │   Logic      │───▶│   Builder    │───▶│   SDK        │                  │
│  │              │    │              │    │              │                  │
│  └──────────────┘    └──────────────┘    └──────────────┘                  │
│                                                 │                           │
│                                                 ▼                           │
│                      ┌─────────────────────────────────────────┐           │
│                      │         Event Processing Pipeline        │           │
│                      ├─────────────────────────────────────────┤           │
│                      │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │           │
│                      │  │Validate │─▶│ Filter  │─▶│  Batch  │  │           │
│                      │  └─────────┘  └─────────┘  └─────────┘  │           │
│                      └─────────────────────────────────────────┘           │
│                                                 │                           │
│                                                 ▼                           │
│                      ┌─────────────────────────────────────────┐           │
│                      │         SQLite FIFO Buffer              │           │
│                      │    (Offline Persistence Layer)          │           │
│                      └─────────────────────────────────────────┘           │
│                                                 │                           │
│                                                 ▼                           │
│                      ┌─────────────────────────────────────────┐           │
│                      │       Background Batch Sender           │           │
│                      │   (Retry Logic + Exponential Backoff)   │           │
│                      └─────────────────────────────────────────┘           │
│                                                 │                           │
└─────────────────────────────────────────────────┼───────────────────────────┘
                                                  │
                                                  ▼
                                    ┌─────────────────────────┐
                                    │    Tracking Server      │
                                    │    (HTTP/JSON-RPC)      │
                                    └─────────────────────────┘
```

---

## Event Data Model

Events are structured as JSON documents with metadata for routing and processing:

```json
{
  "ver": 2,
  "id": 1001,
  "params": {
    "level_id": 150,
    "moves_used": 23,
    "boosters_used": ["hammer", "extra_moves"],
    "stars_earned": 3,
    "time_seconds": 45
  },
  "fill": ["user_id", "session_id", "timestamp"],
  "category": ["gameplay", "level_complete"],
  "data_types": {
    "contains_pii": false
  }
}
```

### Key Fields

| Field | Purpose |
|-------|---------|
| `ver` | Schema version for backward compatibility |
| `id` | Unique event type identifier (maps to event catalog) |
| `params` | Event-specific payload |
| `fill` | Fields to auto-populate (user ID, timestamp, etc.) |
| `category` | Tags for filtering and routing |
| `data_types` | Flags for privacy compliance (GDPR, CCPA) |

---

## Core Components

### 1. Event Builder

The Event Builder provides a type-safe API for constructing events:

```cpp
class EventBuilder {
public:
    // Gameplay events
    static std::string BuildLevelStarted(
        int levelId,
        const std::string& difficulty);

    static std::string BuildLevelCompleted(
        int levelId,
        int movesUsed,
        int starsEarned,
        const std::vector<std::string>& boostersUsed);

    static std::string BuildLevelFailed(
        int levelId,
        int movesUsed,
        const std::string& failReason);

    // Economy events
    static std::string BuildPurchaseStarted(
        const std::string& productId,
        const std::string& currency,
        double price);

    static std::string BuildPurchaseCompleted(
        const std::string& transactionId,
        const std::string& productId);

private:
    static std::string SerializeToJson(
        int eventId,
        const std::map<std::string, JsonValue>& params,
        const std::vector<std::string>& categories);
};
```

**Usage Example:**

```cpp
// In game code
void onLevelComplete(int levelId, int moves, int stars) {
    std::string event = EventBuilder::BuildLevelCompleted(
        levelId, moves, stars, usedBoosters);

    tracking->trackEvent(event);  // Non-blocking
}
```

---

### 2. Event Validation & Filtering

Before persistence, events pass through validation and filtering:

```cpp
class EventProcessor {
public:
    enum class Result {
        Ok,
        InvalidJson,
        MissingEventId,
        EventTooLarge,
        EventBlocked,
        SampledOut
    };

    Result process(const std::string& eventJson) {
        // Step 1: Parse JSON
        rapidjson::Document doc;
        if (doc.Parse(eventJson.c_str()).HasParseError()) {
            diagnostics.recordLoss(LossReason::ParseError);
            return Result::InvalidJson;
        }

        // Step 2: Validate required fields
        if (!doc.HasMember("id") || !doc["id"].IsInt()) {
            diagnostics.recordLoss(LossReason::MissingEventId);
            return Result::MissingEventId;
        }

        // Step 3: Check size limits (prevent memory issues)
        if (eventJson.size() > MAX_EVENT_SIZE_BYTES) {
            diagnostics.recordLoss(LossReason::TooLarge);
            return Result::EventTooLarge;
        }

        // Step 4: Apply server-side filtering rules
        int eventId = doc["id"].GetInt();
        if (filterConfig.isBlocked(eventId)) {
            return Result::EventBlocked;
        }

        // Step 5: Apply sampling (for high-volume events)
        if (filterConfig.shouldSample(eventId)) {
            if (!passesSampling(eventId, filterConfig.getSampleRate(eventId))) {
                return Result::SampledOut;
            }
        }

        // Step 6: Queue for batching
        return batchEvent(doc);
    }

private:
    static constexpr size_t MAX_EVENT_SIZE_BYTES = 64 * 1024;  // 64KB
    FilterConfig filterConfig;
    Diagnostics diagnostics;
};
```

---

### 3. SQLite FIFO Buffer

The persistence layer ensures no events are lost during network outages:

```
┌────────────────────────────────────────────────────────────────┐
│                    SQLite FIFO Buffer                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Table: EventBuffer                                            │
│  ┌────────┬──────────┬────────────┬───────────┬─────────────┐ │
│  │   id   │  method  │   params   │ timestamp │  categories │ │
│  ├────────┼──────────┼────────────┼───────────┼─────────────┤ │
│  │   1    │ tracking │ {...}      │ 170912... │ gameplay    │ │
│  │   2    │ tracking │ {...}      │ 170912... │ economy     │ │
│  │   3    │ tracking │ {...}      │ 170912... │ gameplay    │ │
│  │  ...   │   ...    │    ...     │    ...    │    ...      │ │
│  └────────┴──────────┴────────────┴───────────┴─────────────┘ │
│                                                                │
│  Capacity: 10,000 events (configurable)                        │
│  FIFO: Oldest events sent first                                │
│  Overflow: Oldest events dropped when full                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```cpp
class SqliteFifoBuffer {
public:
    struct Config {
        std::string dbPath = "event_buffer.db";
        int32_t maxCapacity = 10000;
        int32_t batchSize = 100;
    };

    SqliteFifoBuffer(const Config& config) : config_(config) {
        initDatabase();
    }

    bool addEvent(const std::string& method,
                  const std::string& params,
                  const std::vector<std::string>& categories) {
        std::lock_guard<std::mutex> lock(mutex_);

        // Check capacity - drop oldest if full
        if (getCount() >= config_.maxCapacity) {
            deleteOldest(1);
            diagnostics_.recordLoss(LossReason::BufferOverflow);
        }

        // Insert new event
        sqlite3_stmt* stmt;
        sqlite3_prepare_v2(db_,
            "INSERT INTO EventBuffer (method, params, timestamp, categories) "
            "VALUES (?, ?, ?, ?)",
            -1, &stmt, nullptr);

        sqlite3_bind_text(stmt, 1, method.c_str(), -1, SQLITE_TRANSIENT);
        sqlite3_bind_text(stmt, 2, params.c_str(), -1, SQLITE_TRANSIENT);
        sqlite3_bind_int64(stmt, 3, getCurrentTimestamp());
        sqlite3_bind_text(stmt, 4, joinCategories(categories).c_str(), -1, SQLITE_TRANSIENT);

        int result = sqlite3_step(stmt);
        sqlite3_finalize(stmt);

        return result == SQLITE_DONE;
    }

    std::vector<BufferItem> getBatch(int32_t maxItems) {
        std::lock_guard<std::mutex> lock(mutex_);
        std::vector<BufferItem> items;

        sqlite3_stmt* stmt;
        sqlite3_prepare_v2(db_,
            "SELECT id, method, params FROM EventBuffer "
            "ORDER BY id ASC LIMIT ?",
            -1, &stmt, nullptr);

        sqlite3_bind_int(stmt, 1, maxItems);

        while (sqlite3_step(stmt) == SQLITE_ROW) {
            BufferItem item;
            item.id = sqlite3_column_int64(stmt, 0);
            item.method = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 1));
            item.params = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 2));
            items.push_back(item);
        }

        sqlite3_finalize(stmt);
        return items;
    }

    void deleteBatch(const std::vector<int64_t>& ids) {
        std::lock_guard<std::mutex> lock(mutex_);

        std::string idList = joinIds(ids);
        std::string sql = "DELETE FROM EventBuffer WHERE id IN (" + idList + ")";

        sqlite3_exec(db_, sql.c_str(), nullptr, nullptr, nullptr);
    }

    float getUsagePercent() const {
        return static_cast<float>(getCount()) / config_.maxCapacity * 100.0f;
    }

private:
    void initDatabase() {
        sqlite3_open(config_.dbPath.c_str(), &db_);
        sqlite3_exec(db_,
            "CREATE TABLE IF NOT EXISTS EventBuffer ("
            "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
            "  method TEXT NOT NULL,"
            "  params TEXT NOT NULL,"
            "  timestamp INTEGER NOT NULL,"
            "  categories TEXT"
            ")",
            nullptr, nullptr, nullptr);
    }

    Config config_;
    sqlite3* db_;
    std::mutex mutex_;
    Diagnostics diagnostics_;
};
```

---

### 4. Background Batch Sender

The sender runs on a background thread with intelligent retry logic:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Batch Sending State Machine                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│    ┌─────────┐                                                          │
│    │  IDLE   │◀─────────────────────────────────────────────┐          │
│    └────┬────┘                                               │          │
│         │ Timer fires                                        │          │
│         ▼                                                    │          │
│    ┌─────────┐                                               │          │
│    │  FETCH  │ Read batch from SQLite                        │          │
│    │  BATCH  │                                               │          │
│    └────┬────┘                                               │          │
│         │                                                    │          │
│         ▼                                                    │          │
│    ┌─────────┐         ┌─────────┐                          │          │
│    │  SEND   │────────▶│ SUCCESS │──────────────────────────┤          │
│    │         │         └─────────┘  Delete batch,           │          │
│    └────┬────┘                      reset retry count        │          │
│         │                                                    │          │
│         │              ┌───────────────┐                     │          │
│         └─────────────▶│ RETRYABLE     │                     │          │
│           Network      │ ERROR         │                     │          │
│           error        └───────┬───────┘                     │          │
│                                │                             │          │
│                                ▼                             │          │
│                        ┌───────────────┐                     │          │
│                        │  BACKOFF      │  Wait = base × 2^n  │          │
│                        │  (exp. delay) │  (max 256× base)    │          │
│                        └───────┬───────┘                     │          │
│                                │                             │          │
│                                └─────────────────────────────┘          │
│                                                                         │
│         │              ┌───────────────┐                                │
│         └─────────────▶│ NON-RETRYABLE │  Delete batch                  │
│           Server       │ ERROR         │  (server rejected)             │
│           rejected     └───────────────┘                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```cpp
class BatchSender {
public:
    struct Config {
        int32_t baseIntervalMs = 1000;      // 1 second
        int32_t maxBackoffMultiplier = 256;  // 2^8
        int32_t batchSize = 100;
        std::string serverUrl;
    };

    void start() {
        running_ = true;
        senderThread_ = std::thread(&BatchSender::senderLoop, this);
    }

    void stop() {
        running_ = false;
        if (senderThread_.joinable()) {
            senderThread_.join();
        }
    }

private:
    void senderLoop() {
        while (running_) {
            int32_t waitMs = calculateNextInterval();
            std::this_thread::sleep_for(std::chrono::milliseconds(waitMs));

            if (!running_) break;

            sendBatch();
        }
    }

    void sendBatch() {
        // Fetch batch from buffer
        auto batch = buffer_.getBatch(config_.batchSize);
        if (batch.empty()) {
            return;
        }

        // Build JSON-RPC request
        std::string payload = buildJsonRpcPayload(batch);

        // Send HTTP request
        HttpResponse response = httpClient_.post(config_.serverUrl, payload);

        // Handle response
        switch (categorizeResponse(response)) {
            case ResponseType::Success:
                handleSuccess(batch);
                break;
            case ResponseType::RetryableError:
                handleRetryableError();
                break;
            case ResponseType::NonRetryableError:
                handleNonRetryableError(batch);
                break;
        }
    }

    void handleSuccess(const std::vector<BufferItem>& batch) {
        // Delete sent events from buffer
        std::vector<int64_t> ids;
        for (const auto& item : batch) {
            ids.push_back(item.id);
        }
        buffer_.deleteBatch(ids);

        // Reset retry counter
        consecutiveErrors_ = 0;

        diagnostics_.recordSent(batch.size());
    }

    void handleRetryableError() {
        // Increment error count for backoff calculation
        consecutiveErrors_++;

        diagnostics_.recordRetry();
    }

    void handleNonRetryableError(const std::vector<BufferItem>& batch) {
        // Server rejected - discard batch
        std::vector<int64_t> ids;
        for (const auto& item : batch) {
            ids.push_back(item.id);
        }
        buffer_.deleteBatch(ids);

        // Reset retry counter
        consecutiveErrors_ = 0;

        diagnostics_.recordLoss(LossReason::ServerRejected, batch.size());
    }

    int32_t calculateNextInterval() {
        if (consecutiveErrors_ == 0) {
            return config_.baseIntervalMs;
        }

        // Exponential backoff: base × 2^errors (capped)
        int32_t multiplier = std::min(
            1 << consecutiveErrors_,
            config_.maxBackoffMultiplier
        );

        return config_.baseIntervalMs * multiplier;
    }

    ResponseType categorizeResponse(const HttpResponse& response) {
        if (response.statusCode == 200) {
            // Check for JSON-RPC errors in response body
            if (containsJsonRpcError(response.body)) {
                return ResponseType::NonRetryableError;
            }
            return ResponseType::Success;
        }

        if (response.statusCode >= 500 || response.isNetworkError) {
            return ResponseType::RetryableError;
        }

        // 4xx errors are non-retryable
        return ResponseType::NonRetryableError;
    }

    std::atomic<bool> running_{false};
    std::thread senderThread_;
    int32_t consecutiveErrors_ = 0;
    Config config_;
    SqliteFifoBuffer& buffer_;
    HttpClient httpClient_;
    Diagnostics diagnostics_;
};
```

---

### 5. JSON-RPC Transport

Events are sent as JSON-RPC batch requests:

```json
[
  {
    "jsonrpc": "2.0",
    "method": "tracking",
    "params": [
      "com.example.game",
      1,
      1709123456789,
      "session-uuid-here",
      42,
      "user-analytics-id",
      {
        "type": 1001,
        "parameters": {
          "level_id": 150,
          "moves_used": 23,
          "stars_earned": 3
        }
      }
    ],
    "id": 1
  },
  {
    "jsonrpc": "2.0",
    "method": "tracking",
    "params": [...],
    "id": 2
  }
]
```

**Batch Structure:**

| Index | Field | Description |
|-------|-------|-------------|
| 0 | App ID | Application identifier |
| 1 | Sign-in Source | Authentication provider (1=guest, 2=social, etc.) |
| 2 | Timestamp | Unix timestamp in milliseconds |
| 3 | Session ID | Unique session identifier |
| 4 | Event Counter | Monotonic counter for ordering |
| 5 | Analytics ID | User analytics identifier |
| 6 | Event Data | The actual event payload |

---

## Diagnostics & Monitoring

Track SDK health with comprehensive metrics:

```cpp
struct TrackingDiagnostics {
    // Event counts
    std::atomic<uint64_t> eventsTracked{0};
    std::atomic<uint64_t> eventsSent{0};
    std::atomic<uint64_t> eventsLost{0};

    // Loss breakdown
    std::atomic<uint64_t> lostToBufferOverflow{0};
    std::atomic<uint64_t> lostToParseError{0};
    std::atomic<uint64_t> lostToSizeLimit{0};
    std::atomic<uint64_t> lostToServerReject{0};

    // Network stats
    std::atomic<uint64_t> batchesSent{0};
    std::atomic<uint64_t> batchRetries{0};
    std::atomic<uint64_t> bytesSent{0};

    // Buffer stats
    float bufferUsagePercent{0.0f};

    std::string toJson() const {
        return fmt::format(R"({{
            "events_tracked": {},
            "events_sent": {},
            "events_lost": {},
            "loss_breakdown": {{
                "buffer_overflow": {},
                "parse_error": {},
                "size_limit": {},
                "server_reject": {}
            }},
            "batches_sent": {},
            "batch_retries": {},
            "bytes_sent": {},
            "buffer_usage_percent": {:.1f}
        }})",
            eventsTracked.load(),
            eventsSent.load(),
            eventsLost.load(),
            lostToBufferOverflow.load(),
            lostToParseError.load(),
            lostToSizeLimit.load(),
            lostToServerReject.load(),
            batchesSent.load(),
            batchRetries.load(),
            bytesSent.load(),
            bufferUsagePercent
        );
    }
};
```

---

## Platform Considerations

### iOS

```cpp
// Handle app lifecycle
void onAppWillResignActive() {
    // Flush pending events before backgrounding
    batchSender.flushSync();
}

void onAppDidBecomeActive() {
    // Resume sending
    batchSender.start();
}

// Use background task for extended flush time
void onAppDidEnterBackground() {
    auto taskId = [[UIApplication sharedApplication]
        beginBackgroundTaskWithExpirationHandler:^{
            // Time's up - stop gracefully
        }];

    batchSender.flushSync();

    [[UIApplication sharedApplication] endBackgroundTask:taskId];
}
```

### Android

```cpp
// Handle process death
void onTrimMemory(int level) {
    if (level >= TRIM_MEMORY_MODERATE) {
        // Flush to disk immediately
        buffer.flush();
    }
}

// WorkManager for reliable background sync
void scheduleBackgroundSync() {
    OneTimeWorkRequest syncWork = new OneTimeWorkRequest.Builder(
        EventSyncWorker.class)
        .setConstraints(new Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build())
        .build();

    WorkManager.getInstance(context).enqueue(syncWork);
}
```

---

## Summary

A production event tracking SDK requires:

| Component | Purpose |
|-----------|---------|
| Event Builder | Type-safe event construction |
| Validator | Schema validation, size limits |
| Filter | Server-controlled sampling/blocking |
| SQLite Buffer | Offline persistence (FIFO) |
| Batch Sender | Background sending with retry |
| Diagnostics | Health monitoring |

**Key Design Principles:**

1. **Never block the game thread** - All I/O happens asynchronously
2. **Never lose events** - SQLite persistence survives crashes
3. **Graceful degradation** - Exponential backoff prevents server overload
4. **Observable** - Comprehensive diagnostics for debugging

---

## Next: Data Pipeline

In [Part 2](./game-analytics-part2-data-pipeline), we'll explore how these events flow through Kafka to BigQuery, including:

- Event ingestion at scale
- Stream processing with Dataflow
- BigQuery schema design
- Airflow orchestration

---

*Part of the [Game Analytics Series](./index#game-analytics)*
