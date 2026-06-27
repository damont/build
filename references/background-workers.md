# Reference: Background Workers

Implementation guide for apps that need to run work outside the request/response cycle — sending notifications, processing uploads, syncing data, running long computations. Uses Redis Streams with a consumer group pattern.

**Pull this in when:** your app has long-running operations blocking API responses, fan-out notifications, scheduled jobs, or processing pipelines. Most CRUD apps don't need this.

---

## Design Principles

1. **Events, not commands.** Messages describe **what happened**, not what should happen next. Publish `artifact_created` with the artifact ID — don't publish `send_artifact_notification`. This keeps the publisher decoupled from downstream work.
2. **Durable delivery.** Use Redis Streams with consumer groups. Messages persist until acknowledged. Unprocessed messages survive worker restarts.
3. **One worker container.** A single Python worker process listens on all topics. Don't run a separate container per topic.

---

## Event Publishing

Create `api/services/events.py`:

```python
import json
import logging
from datetime import datetime, timezone
from redis.asyncio import Redis

logger = logging.getLogger(__name__)

_redis: Redis | None = None


async def init_events(redis_url: str):
    """Initialize the Redis connection. Call during app lifespan startup."""
    global _redis
    _redis = Redis.from_url(redis_url, decode_responses=True)
    logger.info("Event bus connected to Redis")


async def close_events():
    """Close the Redis connection. Call during app lifespan shutdown."""
    global _redis
    if _redis:
        await _redis.close()
        _redis = None


async def publish(topic: str, data: dict):
    """
    Publish an event to a Redis Stream.

    topic: Stream name, e.g. "artifact_created", "phase_changed"
    data: Event payload. Must be JSON-serializable.
          Always include enough context to identify what happened
          (e.g. project_id, artifact_id) but NOT what to do about it.
    """
    if not _redis:
        logger.warning("Event bus not initialized, dropping event: %s", topic)
        return

    payload = {
        "data": json.dumps(data),
        "published_at": datetime.now(timezone.utc).isoformat(),
    }
    message_id = await _redis.xadd(topic, payload)
    logger.info("Published event %s to %s", message_id, topic)
```

---

## Worker

Create `worker/main.py`:

```python
import asyncio
import json
import logging
from redis.asyncio import Redis
from redis.exceptions import ConnectionError as RedisConnectionError, TimeoutError as RedisTimeoutError

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(name)s: %(message)s")
logger = logging.getLogger(__name__)

# Map topic names to handler coroutines
HANDLERS: dict[str, list] = {}


def handles(topic: str):
    """Decorator to register a handler for a topic."""
    def decorator(fn):
        HANDLERS.setdefault(topic, []).append(fn)
        return fn
    return decorator


# --- Import handlers here so they register via @handles ---
# from worker.handlers import notifications, processing  # noqa


async def run_worker(redis_url: str, topics: list[str], group: str = "workers", consumer: str = "worker-1"):
    # socket_keepalive matters when Redis is reached through a proxy (e.g. the
    # Azure Container Apps ingress) — see the deploy note below.
    redis = Redis.from_url(redis_url, decode_responses=True, socket_keepalive=True)

    # Ensure consumer groups exist
    for topic in topics:
        try:
            await redis.xgroup_create(topic, group, id="0", mkstream=True)
        except Exception:
            pass  # Group already exists

    logger.info("Worker started, listening on: %s", ", ".join(topics))

    while True:
        # Read from all topics, block up to 5 seconds
        streams = {topic: ">" for topic in topics}
        # When Redis is behind a proxy (Container Apps ingress), a blocking
        # XREADGROUP that returns nothing can surface as a client read timeout —
        # treat it as "no messages" and retry rather than crash. Harmless locally.
        try:
            results = await redis.xreadgroup(group, consumer, streams, count=10, block=5000)
        except (RedisTimeoutError, RedisConnectionError):
            continue

        for topic, messages in results or []:
            for message_id, fields in messages:
                try:
                    data = json.loads(fields["data"])
                    handlers = HANDLERS.get(topic, [])
                    for handler in handlers:
                        await handler(data)
                    await redis.xack(topic, group, message_id)
                except Exception:
                    logger.exception("Failed to process %s message %s", topic, message_id)


if __name__ == "__main__":
    import os
    redis_url = os.environ.get("REDIS_URL", "redis://localhost:6379")
    topics = list(HANDLERS.keys())
    asyncio.run(run_worker(redis_url, topics))
```

---

## Handler Example

Create `worker/handlers/notifications.py`:

```python
import logging
from worker.main import handles

logger = logging.getLogger(__name__)


@handles("artifact_created")
async def on_artifact_created(data: dict):
    """React to an artifact being created. The event just says what happened."""
    project_id = data["project_id"]
    artifact_id = data["artifact_id"]
    logger.info("Artifact %s created in project %s", artifact_id, project_id)
    # ... send notification, update cache, etc.
```

---

## Configuration

Add to `api/config.py` Settings class:

```python
redis_url: str = "redis://localhost:6379"
```

Add to `.env.example`:

```
REDIS_URL=redis://localhost:6379
```

Wire into lifespan in `api/main.py`:

```python
from api.services.events import init_events, close_events

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    await init_events(settings.redis_url)
    # ... existing startup (MongoDB, etc.)
    yield
    # ... existing shutdown
    await close_events()
```

---

## Docker Compose Additions

Add Redis and worker services:

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped

  worker:
    build:
      context: .
      target: backend  # Same image as the API
    container_name: myapp-worker
    command: ["uv", "run", "python", "-m", "worker.main"]
    environment:
      - REDIS_URL=redis://redis:6379
      - MONGODB_URL=mongodb://host.docker.internal:27017
      - MONGODB_DB_NAME=myapp
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      - redis
    restart: unless-stopped

  api:
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    # ... existing api config

volumes:
  redis-data:
```

---

## Dependencies

Add to `pyproject.toml`:

```toml
"redis[hiredis]>=5.0.0",
```

Only add this when the app actually needs background workers.

---

## Deployment Considerations

| Environment | Redis | Notes |
|-------------|-------|-------|
| **Local dev** | `redis:7-alpine` in Docker Compose | Just works, no setup needed |
| **Pi (Path A)** | Same `redis:7-alpine` in Docker Compose | Lightweight, runs fine on Pi. Volume persists data |
| **Azure (Path B)** | `redis:7-alpine` as an internal **Container App** (TCP :6379, always-on) | Classic Azure Cache for Redis is blocked for new creates and Azure Managed Redis is ~$50–75/mo — so self-host the container. The worker reaches it through the env ingress proxy, so the blocking `XREADGROUP` needs the `socket_keepalive` + `TimeoutError`-retry handling shown above, and the worker scales from zero via a KEDA `redis-streams` rule. See Phase 08 → Path B for the full setup. |

---

## Key Rules

- **Events describe facts, not intentions.** `phase_changed`, `entity_created`, `artifact_uploaded` — not `send_notification`, `update_dashboard`, `reindex_search`.
- **Keep payloads minimal.** Include IDs and metadata. The handler fetches the full object from the database if needed.
- **The API never waits for the worker.** Publish the event and return the response.
- **The worker shares the same codebase** (same Docker image, same Beanie models) — it just runs a different entrypoint.
- **Only add a worker when you actually need async processing.** Most CRUD apps don't need one.
