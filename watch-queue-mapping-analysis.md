# Watch-Queue Mapping Analysis Report

## Overview

This report documents how Watch configurations are mapped to queue metadata in the changedetection.io application. The system uses a priority-based queue mechanism to manage watch check scheduling and execution.

---

## Core Components

### 1. PrioritizedItem (`queuedWatchMetaData.py`)

The fundamental data structure for queue items:

```python
@dataclass(order=True)
class PrioritizedItem:
    priority: int
    item: Any=field(compare=False)
```

**Key Points:**
- Uses Python's `dataclass` with `order=True` for priority queue ordering
- Lower priority value = higher precedence (processed first)
- `item` typically contains `{'uuid': watch_uuid}`

---

## Queue Types

### 1. SignalPriorityQueue (`custom_queue.py`)

Synchronous PriorityQueue with signal emission:

```python
class SignalPriorityQueue(queue.PriorityQueue):
    def put(self, item, block=True, timeout=None):
        # Emits 'watch_check_update' signal with watch UUID
        # Emits 'queue_length' signal with current queue size
```

**Signals Emitted:**
- `watch_check_update`: When item with UUID is added
- `queue_length`: Queue size changes (on put/get)

### 2. AsyncSignalPriorityQueue (`custom_queue.py`)

Async version for asyncio event loops - same signal behavior as sync version.

### 3. RecheckPriorityQueue (`queue_handlers.py`)

Thread-safe priority queue supporting multiple async event loops:

```python
class RecheckPriorityQueue:
    # Hybrid sync/async design
    # Sync interface: threading.Queue for ticker thread
    # Async interface: asyncio.Event for workers
```

**Architecture:**
- Multiple async workers with separate event loops
- Pure coroutines (no threads while waiting)
- Polls queue every 50ms

### 4. NotificationQueue (`queue_handlers.py`)

Separate queue for notifications:

```python
class NotificationQueue:
    # Bridges sync (Flask routes) and async (workers) contexts
    # Emits 'notification_event' signal
```

---

## Priority Levels

| Priority | Type | Description | Use Case |
|----------|------|-------------|----------|
| 1 | Immediate | Highest priority | Manual recheck, new watch creation, realtime triggers |
| 5 | Clone | Medium-high | Watch cloning operations |
| > 100 | Scheduled | Time-based (epoch timestamp) | Automatic scheduled rechecks |
| 1000+ | Deferred | Low priority | Re-queued items (already being processed) |

---

## Queue Metadata Structure

### Queue Item Format

```python
PrioritizedItem(priority=<int>, item={'uuid': '<watch_uuid>'})
```

**Metadata Contents:**
- `priority`: Integer determining processing order
- `item`: Dictionary containing:
  - `uuid`: Watch UUID (required) - used to retrieve watch from datastore

### Example Queue Entries

```python
# Immediate recheck
PrioritizedItem(priority=1, item={'uuid': 'abc123-def456'})

# Clone operation
PrioritizedItem(priority=5, item={'uuid': 'new-uuid-789'})

# Scheduled recheck (uses epoch timestamp)
PrioritizedItem(priority=1699123456, item={'uuid': 'scheduled-uuid'})

# Deferred recheck (when already processing)
PrioritizedItem(priority=10000, item={'uuid': 'deferred-uuid'})
```

---

## Watch to Queue Mapping Locations

### 1. Manual Recheck (UI/API)

**File:** `blueprint/ui/__init__.py`
```python
# Single watch recheck - line 276
worker_pool.queue_item_async_safe(update_q,
    PrioritizedItem(priority=1, item={'uuid': uuid}))

# Batch recheck - line 305
worker_pool.queue_item_async_safe(update_q,
    PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
```

### 2. Watch Clone

**File:** `blueprint/ui/__init__.py` line 257
```python
# Clone has lower priority (5) than immediate rechecks
worker_pool.queue_item_async_safe(update_q,
    PrioritizedItem(priority=5, item={'uuid': new_uuid}))
```

### 3. Realtime Mode

**File:** `realtime/events.py` line 44
```python
# Realtime triggers use priority 1
worker_pool.queue_item_async_safe(update_q,
    PrioritizedItem(priority=1, item={'uuid': uuid}))
```

### 4. Scheduled Checks (Ticker Thread)

**File:** `flask_app.py` line 1253
```python
# Uses epoch timestamp as priority for time-based scheduling
priority = int(time.time())
queuedWatchMetaData.PrioritizedItem(priority=priority,
                                    item={'uuid': uuid})
```

### 5. Batch Mode

**File:** `__init__.py` lines 441, 475, 537
```python
# Batch mode additions use priority 1
worker_pool.queue_item_async_safe(update_q,
    queuedWatchMetaData.PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
```

### 6. Tag-Based Operations

**File:** `api/Tags.py` lines 42, 50
```python
# Tag changes trigger rechecks with priority 1
worker_pool.queue_item_async_safe(self.update_q,
    PrioritizedItem(priority=1, item={'uuid': watch_uuid}))
```

### 7. Watch Edit/Import

**File:** `blueprint/ui/edit.py` line 277
```python
# After editing a watch
worker_pool.queue_item_async_safe(update_q,
    PrioritizedItem(priority=1, item={'uuid': uuid}))
```

### 8. Price Data Follower

**File:** `blueprint/price_data_follower/__init__.py` line 24
```python
# Priority 1 for price data triggers
worker_pool.queue_item_async_safe(update_q,
    PrioritizedItem(priority=1, item={'uuid': uuid}))
```

### 9. Worker Deferral

**File:** `worker.py` line 75
```python
# When UUID is already being processed, re-queue with higher priority number
deferred_priority = max(1000, queued_item_data.priority * 10)
deferred_item = PrioritizedItem(priority=deferred_priority,
                                item=queued_item_data.item)
```

---

## Queue Processing Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     WATCH QUEUE FLOW                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────────┐   │
│  │  Watch   │───>│ Prioritized  │───>│ RecheckPriority   │   │
│  │  Event   │    │    Item      │    │     Queue         │   │
│  └──────────┘    └──────────────┘    └─────────┬─────────┘   │
│       │                   │                    │             │
│       │ Priority:         │ Metadata:          │             │
│       │ 1=Immediate       │ {'uuid': ...}     │             │
│       │ 5=Clone           │                   │             │
│       │ >100=Scheduled    │                   │             │
│       │ 1000+=Deferred    │                   │             │
│                                               │             │
│                                               ▼             │
│                                      ┌─────────────────┐    │
│                                      │  Async Workers  │    │
│                                      │  (worker.py)    │    │
│                                      └────────┬────────┘    │
│                                               │              │
│                                               │ get(uuid)    │
│                                               ▼              │
│                                      ┌─────────────────┐    │
│                                      │    Datastore    │    │
│                                      │   (watch data)  │    │
│                                      └─────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Queue Operations

### Queue Item Methods

```python
# Add to queue
worker_pool.queue_item_async_safe(update_q, PrioritizedItem(priority=1, item={'uuid': uuid}))

# Check queue position
update_q.get_uuid_position(target_uuid)

# Get all queued UUIDs
update_q.get_all_queued_uuids(limit=100, offset=0)

# Get queue summary
update_q.get_queue_summary()

# Check if UUID is queued
uuid in update_q.get_queued_uuids()
```

### Queue Summary Structure

```python
{
    'total_items': 150,
    'priority_breakdown': {1: 5, 5: 2, 1699123456: 143},
    'immediate_items': 5,    # priority 1
    'clone_items': 2,         # priority 5
    'scheduled_items': 143,   # priority > 100
    'min_priority': 1,
    'max_priority': 1699123456
}
```

---

## Signal System

The queue uses blinker signals for event propagation:

| Signal | Trigger | Purpose |
|--------|---------|---------|
| `watch_check_update` | Item with UUID added to queue | UI updates, SSE notifications |
| `queue_length` | Queue size changes | Dashboard queue display |
| `notification_event` | Notification added | Notification processing |
| `watch_favicon_bump` | Favicon updated | Favicon cache invalidation |
| `watch_small_status_comment` | Status update | Progress indicators |

---

## Priority Calculation

### Scheduled rechecks use epoch timestamp:
```python
priority = int(time.time())  # Current Unix timestamp
```

### Deferred items calculate new priority:
```python
deferred_priority = max(1000, queued_item_data.priority * 10)
```

### Clone operations use fixed priority:
```python
priority = 5  # Lower than immediate but higher than scheduled
```

---

## Summary

The watch-queue mapping in changedetection.io follows a priority-based scheduling model:

1. **Metadata is Minimal**: Queue items only contain `{'uuid': uuid}` - the watch data itself is retrieved from the datastore on demand.

2. **Priority Determines Order**: Lower numbers = higher priority (1 is highest, scheduled checks use epoch time which is always higher).

3. **Multiple Entry Points**: Watches enter the queue via UI actions, API calls, scheduled ticker, realtime triggers, and batch operations.

4. **Signal-Based Updates**: The system uses blinker signals to propagate queue events to UI components and other subsystems.

5. **Thread-Safe Design**: Hybrid sync/async queue implementation supports multiple workers with separate event loops.

---

## File Reference

| File | Purpose |
|------|---------|
| `queuedWatchMetaData.py` | PrioritizedItem dataclass definition |
| `custom_queue.py` | SignalPriorityQueue, AsyncSignalPriorityQueue |
| `queue_handlers.py` | RecheckPriorityQueue, NotificationQueue |
| `worker.py` | Async worker processing from queue |
| `flask_app.py` | Ticker thread scheduling logic |
| `__init__.py` | Batch mode queue operations |
| `blueprint/ui/__init__.py` | UI-triggered queue operations |
| `realtime/events.py` | Realtime mode queue triggers |
| `api/Watch.py`, `api/Tags.py` | API-triggered queue operations |
