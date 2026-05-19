# FastAPI Routes

REST endpoint patterns for the Interfaces and Implementation Detail sections.

---

## Router structure

Each domain area gets its own router module under `package/routers/`. Mount on the app with prefix and tags:

```python
# package/www.py
from .routers import sources, articles

app.include_router(sources.router, prefix="/sources", tags=["sources"])
app.include_router(articles.router, prefix="/articles", tags=["articles"])
```

## Full router example — sources CRUD

```python
# package/routers/sources.py
from uuid import UUID

from fastapi import APIRouter, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from ..models import Source
from ..schemas.source import SourceCreate, SourceRead
from ..services.db import get_session
from ..services.feed import normalize_url

router = APIRouter()


@router.get("", response_model=list[SourceRead])
async def list_sources() -> list[SourceRead]:
    async with get_session() as session:
        result = await session.execute(select(Source).order_by(Source.added_at))
        return result.scalars().all()


@router.post("", response_model=SourceRead, status_code=201)
async def create_source(body: SourceCreate) -> SourceRead:
    url = normalize_url(body.url)

    async with get_session() as session:
        # Check for duplicate
        result = await session.execute(select(Source).where(Source.url == url))
        if result.scalar_one_or_none():
            raise HTTPException(status_code=409, detail="Source URL already exists")

        source = Source(url=url, name=body.name)
        session.add(source)
        await session.commit()
        return source


@router.get("/{source_id}", response_model=SourceRead)
async def get_source(source_id: UUID) -> SourceRead:
    async with get_session() as session:
        result = await session.execute(select(Source).where(Source.id == source_id))
        source = result.scalar_one_or_none()
        if not source:
            raise HTTPException(status_code=404, detail="Source not found")
        return source


@router.delete("/{source_id}", status_code=204)
async def delete_source(source_id: UUID) -> None:
    async with get_session() as session:
        result = await session.execute(select(Source).where(Source.id == source_id))
        source = result.scalar_one_or_none()
        if not source:
            raise HTTPException(status_code=404, detail="Source not found")
        await session.delete(source)
        await session.commit()
```

## Paginated list endpoint

```python
# package/routers/articles.py
from uuid import UUID

from fastapi import APIRouter, HTTPException
from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from ..models import Article, ArticleContent
from ..schemas.article import ArticleRead
from ..schemas.pagination import PaginatedList, PaginationParams
from ..services.db import get_session

router = APIRouter()


@router.get("", response_model=PaginatedList[ArticleRead])
async def list_articles(params: PaginationParams) -> PaginatedList[ArticleRead]:
    async with get_session() as session:
        # Count total
        count_result = await session.execute(select(func.count()).select_from(Article))
        total = count_result.scalar()

        # Fetch page
        result = await session.execute(
            select(Article)
            .options(joinedload(Article.content))
            .order_by(Article.published_at.desc(), Article.fetched_at.desc())
            .offset(params.offset)
            .limit(params.limit)
        )
        articles = result.scalars().all()

        return PaginatedList(
            items=[ArticleRead.model_validate(a) for a in articles],
            total=total,
            limit=params.limit,
            offset=params.offset,
            has_more=(params.offset + len(articles) < total),
        )
```

## Source-specific articles endpoint

```python
@router.get("/sources/{source_id}/articles", response_model=PaginatedList[ArticleRead])
async def list_source_articles(
    source_id: UUID,
    params: PaginationParams,
) -> PaginatedList[ArticleRead]:
    async with get_session() as session:
        # Verify source exists
        source_result = await session.execute(select(Source).where(Source.id == source_id))
        if not source_result.scalar_one_or_none():
            raise HTTPException(status_code=404, detail="Source not found")

        count_result = await session.execute(
            select(func.count()).select_from(Article).where(Article.source_id == source_id)
        )
        total = count_result.scalar()

        result = await session.execute(
            select(Article)
            .where(Article.source_id == source_id)
            .order_by(Article.published_at.desc(), Article.fetched_at.desc())
            .offset(params.offset)
            .limit(params.limit)
        )
        articles = result.scalars().all()

        return PaginatedList(
            items=[ArticleRead.model_validate(a) for a in articles],
            total=total,
            limit=params.limit,
            offset=params.offset,
            has_more=(params.offset + len(articles) < total),
        )
```

## Async task trigger endpoint

```python
@router.post("/{source_id}/fetch", status_code=202)
async def trigger_fetch(source_id: UUID) -> dict[str, str]:
    async with get_session() as session:
        result = await session.execute(select(Source).where(Source.id == source_id))
        source = result.scalar_one_or_none()
        if not source:
            raise HTTPException(status_code=404, detail="Source not found")

    # Dispatch Celery task
    fetch_source.delay(str(source_id))

    return {"detail": "Fetch triggered"}
```

## Key conventions

- **Router per domain**: `sources.py`, `articles.py` — mounted with `prefix` and `tags`
- **Response models**: `response_model=SourceRead` or `response_model=PaginatedList[ArticleRead]`
- **Status codes**: `201` for creation, `202` for async task triggers, `204` for deletion
- **Error handling**: `HTTPException(status_code=404/409, detail="...")`
- **Session management**: `async with get_session() as session:` — never hold sessions across endpoints
- **Pagination**: `PaginationParams` as a dependency (FastAPI auto-converts query params)
- **Ordering**: `ORDER BY published_at DESC, fetched_at DESC` with `NULLS LAST` for nullable timestamps
