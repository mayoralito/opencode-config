# Celery Tasks

Background task patterns for the Implementation Detail section.

---

## Celery app setup with syncify and cache wrappers

```python
# package/celery.py
from collections.abc import Callable
from functools import wraps
from typing import Any

import asyncio
from celery import Celery
from logging import getLogger

from .services.cache import configure_caches, get_cache

logger = getLogger(__name__)

celery = Celery("package")


def _syncify(f: Callable[..., Any]) -> Callable[..., Any]:
    """Sync wrapper for async functions — mirrors cli.py @syncify."""
    @wraps(f)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        return asyncio.run(f(*args, **kwargs))
    return wrapper


# --- Sync cache wrappers for use in Celery tasks ---

def _cache_exists(key: str) -> bool:
    cache = get_cache("persistent")
    return asyncio.run(cache.exists(key))


def _cache_set(key: str, value: Any, ttl: int) -> None:
    cache = get_cache("persistent")
    asyncio.run(cache.set(key, value, ttl=ttl))


def _cache_delete(key: str) -> None:
    cache = get_cache("persistent")
    asyncio.run(cache.delete(key))


@celery.on_after_configure.connect
def setup_caches(sender: Any, **kwargs: Any) -> None:
    configure_caches()
```

## Periodic task with overlap prevention

```python
@celery.task
def fetch_all_sources() -> None:
    """Fetch all sources. Prevents overlapping executions."""
    lock_key = "fetch_all_sources:in_progress"

    if _cache_exists(lock_key):
        logger.info("fetch_all_sources already in progress, skipping")
        return

    _cache_set(lock_key, True, ttl=3600)
    try:
        _syncify(_fetch_all_sources_impl)()
    finally:
        _cache_delete(lock_key)


async def _fetch_all_sources_impl() -> None:
    from sqlalchemy import select
    from .models import Source
    from .services.db import get_session
    from .services.feed import FeedService, FeedFetchError

    service = FeedService()
    async with get_session() as session:
        result = await session.execute(select(Source))
        sources = result.scalars().all()
        for source in sources:
            try:
                count = await service.process_feed(session, source)
                logger.info("Fetched %d articles from %s", count, source.name)
            except FeedFetchError:
                logger.exception("Failed to fetch source: %s", source.name)
            except Exception:
                logger.exception("Unexpected error fetching source: %s", source.name)
```

## Manual task

```python
@celery.task
def fetch_source(source_id: str) -> None:
    """Fetch a single source by ID."""
    _syncify(_fetch_source_impl)(source_id)


async def _fetch_source_impl(source_id: str) -> None:
    from sqlalchemy import select
    from .models import Source
    from .services.db import get_session
    from .services.feed import FeedService, FeedFetchError

    service = FeedService()
    async with get_session() as session:
        result = await session.execute(select(Source).where(Source.id == source_id))
        source = result.scalar_one_or_none()
        if not source:
            logger.error("Source not found: %s", source_id)
            return
        try:
            count = await service.process_feed(session, source)
            logger.info("Fetched %d articles from %s", count, source.name)
        except FeedFetchError:
            logger.exception("Failed to fetch source: %s", source.name)
```

## Periodic task scheduling

```python
@celery.on_after_finalize.connect
def setup_periodic_tasks(sender: Any, **kwargs: Any) -> None:
    from .settings import settings

    interval_seconds = settings.default_fetch_interval * 60
    logger.info("Scheduling fetch_all_sources every %d seconds", interval_seconds)
    sender.add_periodic_task(
        interval_seconds,
        fetch_all_sources.s(),
        name="Fetch All Sources",
    )
```

## Key conventions

- **`_syncify` wrapper** bridges async service code from sync Celery workers
- **Thin sync cache wrappers** (`_cache_exists`, `_cache_set`, `_cache_delete`) keep lock logic clean
- **`async def _impl(...)`** functions contain the actual logic
- **`@celery.task`** decorated sync entry points call `_syncify(_impl)`
- **Overlap prevention** uses a Redis lock with try/finally for cleanup
- **Lock TTL** is a safety net (e.g., 1 hour) — if the worker crashes, the lock expires
- **Per-source error handling** — one source failure doesn't stop the cycle
- **`on_after_configure`** for setup (caches), **`on_after_finalize`** for periodic tasks
