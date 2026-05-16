# Typer CLI

CLI command patterns for the Interfaces and Implementation Detail sections.

---

## CLI entry point with syncify

```python
# package/cli.py
import asyncio
from functools import wraps
from typing import Any

import typer

app = typer.Typer()


def syncify(f: Any) -> Any:
    """Sync wrapper for async Typer commands."""
    @wraps(f)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        return asyncio.run(f(*args, **kwargs))
    return wrapper
```

## Commands

```python
@app.command(help="Trigger a fetch of all sources.")
@syncify
async def fetch() -> None:
    from sqlalchemy import select
    from ..models import Source
    from ..services.db import get_session
    from ..services.feed import FeedService, FeedFetchError

    service = FeedService()
    async with get_session() as session:
        result = await session.execute(select(Source))
        sources = result.scalars().all()
        for source in sources:
            try:
                count = await service.process_feed(session, source)
                typer.echo(f"Fetched {count} articles from {source.name}")
            except FeedFetchError:
                typer.echo(f"Failed to fetch {source.name}")

    typer.echo("Fetch complete.")


@app.command(help="Seed default sources into the database.")
@syncify
async def seed() -> None:
    from ..services.db import get_session
    from ..services.seed import seed_sources

    async with get_session() as session:
        await seed_sources(session)

    typer.echo("Default sources seeded.")
```

## Command with arguments and options

```python
from typing import Annotated

@app.command(help="Validate content manifests.")
@syncify
async def validate(
    path: Annotated[str, typer.Argument(help="Path to content directory.")],
    strict: Annotated[bool, typer.Option("--strict", help="Fail on warnings.")] = False,
) -> None:
    from ..services.content import validate_content

    errors, warnings = await validate_content(path)
    if errors:
        for e in errors:
            typer.echo(f"ERROR: {e}", err=True)
        raise typer.Exit(1)
    if warnings and strict:
        for w in warnings:
            typer.echo(f"WARN: {w}", err=True)
        raise typer.Exit(1)
    elif warnings:
        for w in warnings:
            typer.echo(f"WARN: {w}")

    typer.echo(f"Validation passed ({len(warnings)} warnings).")
```

## Key conventions

- **`@syncify` decorator** wraps all async commands — never use `asyncio.run()` directly
- **`@app.command(help=...)`** — every command has a help string
- **`Annotated[T, typer.Argument(...)]`** / **`Annotated[T, typer.Option(...)]`** for all params
- **Error output**: `typer.echo(..., err=True)` + `raise typer.Exit(1)` for failures
- **Success output**: `typer.echo(...)` for informational messages
