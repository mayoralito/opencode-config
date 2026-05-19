# SQLAlchemy Models

Complete relational model examples for the Data Storage section.

---

## Basic model with all conventions

```python
# package/models/source.py
from datetime import datetime, timezone
from uuid import UUID, uuid4

from sqlalchemy import Boolean, DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base


class Source(Base):
    """An RSS/Atom feed source."""

    __tablename__ = "source"

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    url: Mapped[str] = mapped_column(String(2048), unique=True, nullable=False)
    name: Mapped[str | None] = mapped_column(String(256), nullable=True)
    added_by: Mapped[UUID | None] = mapped_column(
        nullable=True,
        comment="FK to User.id — added in Phase 2 when User table exists",
    )
    is_default: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False)
    added_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(tz=timezone.utc),
    )
    last_fetched_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
    )
```

## Model with composite unique constraint

```python
# package/models/article_content.py
from datetime import datetime, timezone
from uuid import UUID, uuid4

from sqlalchemy import DateTime, ForeignKey, String, Text, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base


class ArticleContent(Base):
    """Full body content for an article.

    Structured as a separate table to support future translation.
    UniqueConstraint on (article_id, language) allows multiple translations
    per article without any schema migration when translation is added.
    """

    __tablename__ = "article_content"
    __table_args__ = (
        UniqueConstraint("article_id", "language", name="uq_article_content_article_language"),
    )

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    article_id: Mapped[UUID] = mapped_column(
        ForeignKey("article.id", ondelete="CASCADE"),
        nullable=False,
    )
    content: Mapped[str] = mapped_column(Text, nullable=False)
    language: Mapped[str | None] = mapped_column(String(10), nullable=True)
    is_translated: Mapped[bool] = mapped_column(nullable=False, default=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(tz=timezone.utc),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(tz=timezone.utc),
        onupdate=lambda: datetime.now(tz=timezone.utc),
    )
```

## Model with FK and index

```python
# package/models/article.py
from datetime import datetime, timezone
from uuid import UUID, uuid4

from sqlalchemy import DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base


class Article(Base):
    """A news article fetched from a source.

    Globally deduplicated by URL — if the same URL appears in multiple feeds,
    only one Article row exists (owned by the first source that fetched it).
    """

    __tablename__ = "article"

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    source_id: Mapped[UUID] = mapped_column(
        ForeignKey("source.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    title: Mapped[str] = mapped_column(String(1024), nullable=False)
    url: Mapped[str] = mapped_column(String(2048), unique=True, nullable=False)
    published_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
    )
    fetched_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(tz=timezone.utc),
    )
    language: Mapped[str | None] = mapped_column(String(10), nullable=True)
```

## Junction table (many-to-many)

```python
# package/models/article_author.py
from uuid import UUID, uuid4

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base


class ArticleAuthor(Base):
    """Junction table linking Articles to Authors."""

    __tablename__ = "article_author"

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    article_id: Mapped[UUID] = mapped_column(
        ForeignKey("article.id", ondelete="CASCADE"),
        nullable=False,
    )
    author_id: Mapped[UUID] = mapped_column(
        ForeignKey("author.id", ondelete="CASCADE"),
        nullable=False,
    )
```

## Structural placeholder (no service population)

```python
# package/models/topic.py
from uuid import UUID, uuid4

from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base


class Topic(Base):
    """A topic/tag for categorizing articles.

    Structural placeholder for future AI-generated topics.
    Not populated during foundation — Topic and ArticleTopic tables exist
    but the feed service does not write to them.
    """

    __tablename__ = "topic"

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    name: Mapped[str] = mapped_column(String(128), unique=True, nullable=False)
```

---

## Key conventions

- **Identity**: `id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)`
- **FK with cascade**: `ForeignKey("table.id", ondelete="CASCADE")`
- **Nullable FK to future table**: plain `UUID | None` column with `comment=`, no `ForeignKey`
- **Timestamps**: `DateTime(timezone=True)` with `default=lambda: datetime.now(tz=timezone.utc)`
- **Auto-updating timestamp**: add `onupdate=lambda: datetime.now(tz=timezone.utc)`
- **String columns**: always specify max length (`String(256)`, `String(2048)`)
- **Composite unique**: `UniqueConstraint` in `__table_args__`, not `unique=True` on a single column
- **Index on FK**: `index=True` on the `mapped_column` for the FK column when queried frequently
- **Docstrings**: class-level docstring explaining purpose and future-expansion notes
