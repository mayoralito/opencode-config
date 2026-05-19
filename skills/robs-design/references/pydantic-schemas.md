# Pydantic Schemas

Input/output schema patterns and discriminated unions for the Data Structures section.

---

## Input vs. output separation

Always define separate models for input (creation/update) and output (reading). Never reuse the same model.

```python
# package/schemas/source.py
from datetime import datetime
from pydantic import BaseModel, Field
from uuid import UUID


class SourceCreate(BaseModel):
    url: str = Field(
        description="Feed URL. Will be normalized before storage.",
        examples=["https://feeds.example.com/rss"],
    )
    name: str | None = Field(
        default=None,
        description="Optional display name. Derived from feed on first fetch if omitted.",
    )


class SourceRead(BaseModel):
    id: UUID = Field(description="Unique identifier.")
    url: str = Field(description="Normalized URL.")
    name: str | None = Field(description="Display name.")
    is_default: bool = Field(description="Whether this is a built-in default source.")
    added_at: datetime = Field(description="When the source was added.")
    last_fetched_at: datetime | None = Field(
        default=None,
        description="When the source was last successfully fetched.",
    )
```

## Nested output with related data

```python
# package/schemas/article.py
from datetime import datetime
from pydantic import BaseModel, Field
from uuid import UUID


class AuthorRead(BaseModel):
    id: UUID
    name: str


class ArticleRead(BaseModel):
    id: UUID = Field(description="Unique identifier.")
    source_id: UUID = Field(description="ID of the source that first fetched this article.")
    title: str = Field(description="Article title.")
    url: str = Field(description="Article URL (globally unique).")
    published_at: datetime | None = Field(description="Publisher-provided publication time.")
    fetched_at: datetime = Field(description="When the article was fetched by this service.")
    language: str | None = Field(description="Article language code.")
    content: str | None = Field(description="Full article body content.")
    authors: list[AuthorRead] = Field(default_factory=list, description="Article authors.")
```

## Pagination schemas

```python
# package/schemas/pagination.py
from typing import Generic, TypeVar
from pydantic import BaseModel, Field

T = TypeVar("T")


class PaginationParams(BaseModel):
    """Query parameters for paginated list endpoints."""
    limit: int = Field(default=50, ge=1, le=200, description="Number of results to return (max 200).")
    offset: int = Field(default=0, ge=0, description="Number of results to skip.")


class PaginatedList(BaseModel, Generic[T]):
    """Paginated response wrapper with total count and pagination metadata."""
    items: list[T] = Field(description="Items in this page.")
    total: int = Field(description="Total number of items matching the query (ignoring pagination).")
    limit: int = Field(description="Number of results requested.")
    offset: int = Field(description="Offset applied.")
    has_more: bool = Field(description="Whether more items exist beyond this page.")
```

## Discriminated union — condition tree

```python
# package/engine/models/conditions.py
from __future__ import annotations
from typing import Annotated, List, Literal, Union
from pydantic import BaseModel, Field, model_validator


# --- Leaf nodes ---

class LevelCondition(BaseModel):
    type: Literal["level"]
    value: int = Field(ge=1, description="Level threshold.")

class MilestoneCondition(BaseModel):
    type: Literal["milestone"]
    name: str

class CharacterStatCondition(BaseModel):
    type: Literal["character_stat"]
    name: str
    gt: int | float | None = None
    gte: int | float | None = None
    lt: int | float | None = None
    lte: int | float | None = None

    @model_validator(mode="after")
    def require_comparator(self) -> "CharacterStatCondition":
        if all(v is None for v in (self.gt, self.gte, self.lt, self.lte)):
            raise ValueError("character_stat must specify at least one comparator")
        return self


# --- Branch nodes ---

class AllCondition(BaseModel):
    type: Literal["all"]
    conditions: List["Condition"] = Field(min_length=1)

class AnyCondition(BaseModel):
    type: Literal["any"]
    conditions: List["Condition"] = Field(min_length=1)

class NotCondition(BaseModel):
    type: Literal["not"]
    condition: "Condition"


Condition = Annotated[
    Union[
        AllCondition, AnyCondition, NotCondition,
        LevelCondition, MilestoneCondition, CharacterStatCondition,
    ],
    Field(discriminator="type"),
]

AllCondition.model_rebuild()
AnyCondition.model_rebuild()
NotCondition.model_rebuild()
```

## Discriminated union — effects and steps

```python
from __future__ import annotations
from typing import Annotated, List, Literal, Union
from pydantic import BaseModel, Field, model_validator


# Effects — silent mechanical outcomes

class XpGrantEffect(BaseModel):
    type: Literal["xp_grant"]
    amount: int = Field(ne=0)

class ItemDropEntry(BaseModel):
    item: str
    weight: int = Field(ge=1)

class ItemDropEffect(BaseModel):
    type: Literal["item_drop"]
    count: int = Field(default=1, ge=1)
    loot: List[ItemDropEntry] = Field(min_length=1)

class MilestoneGrantEffect(BaseModel):
    type: Literal["milestone_grant"]
    milestone: str

class EndAdventureEffect(BaseModel):
    type: Literal["end_adventure"]
    outcome: Literal["completed", "defeated", "fled"] = "completed"

Effect = Annotated[
    Union[XpGrantEffect, ItemDropEffect, MilestoneGrantEffect, EndAdventureEffect],
    Field(discriminator="type"),
]


# OutcomeBranch — effects + sub-steps shared by all branching events

class OutcomeBranch(BaseModel):
    effects: List[Effect] = []
    steps: List["Step"] = []
    goto: str | None = None

    @model_validator(mode="after")
    def goto_and_steps_are_exclusive(self) -> "OutcomeBranch":
        if self.goto is not None and self.steps:
            raise ValueError("Cannot have both 'goto' and 'steps'.")
        return self


# Events — player-facing screens

class NarrativeStep(BaseModel):
    type: Literal["narrative"]
    label: str | None = None
    text: str = Field(min_length=1)
    effects: List[Effect] = []

class CombatStep(BaseModel):
    type: Literal["combat"]
    label: str | None = None
    enemy: str
    on_win: OutcomeBranch = Field(default_factory=OutcomeBranch)
    on_defeat: OutcomeBranch = Field(default_factory=OutcomeBranch)
    on_flee: OutcomeBranch = Field(default_factory=OutcomeBranch)

class ChoiceOption(BaseModel):
    label: str
    requires: "Condition" | None = None
    effects: List[Effect] = []
    steps: List["Step"] = []
    goto: str | None = None

    @model_validator(mode="after")
    def goto_and_steps_are_exclusive(self) -> "ChoiceOption":
        if self.goto is not None and self.steps:
            raise ValueError("Cannot have both 'goto' and 'steps'.")
        return self

class ChoiceStep(BaseModel):
    type: Literal["choice"]
    label: str | None = None
    prompt: str
    options: List[ChoiceOption] = Field(min_length=1)

class StatCheckStep(BaseModel):
    type: Literal["stat_check"]
    label: str | None = None
    condition: "Condition"
    on_pass: OutcomeBranch = Field(default_factory=OutcomeBranch)
    on_fail: OutcomeBranch = Field(default_factory=OutcomeBranch)


Step = Annotated[
    Union[NarrativeStep, CombatStep, ChoiceStep, StatCheckStep],
    Field(discriminator="type"),
]

NarrativeStep.model_rebuild()
CombatStep.model_rebuild()
ChoiceStep.model_rebuild()
StatCheckStep.model_rebuild()
```

## Model with graph validation

```python
# package/engine/models/quest.py
from typing import List, Set
from pydantic import BaseModel, model_validator


class QuestStage(BaseModel):
    name: str
    description: str = ""
    advance_on: List[str] = []
    next_stage: str | None = None
    terminal: bool = False


class QuestSpec(BaseModel):
    displayName: str
    entry_stage: str
    stages: List[QuestStage]

    @model_validator(mode="after")
    def validate_stage_graph(self) -> "QuestSpec":
        stage_names = [s.name for s in self.stages]

        seen: Set[str] = set()
        for name in stage_names:
            if name in seen:
                raise ValueError(f"Duplicate quest stage name: {name!r}")
            seen.add(name)

        if self.entry_stage not in seen:
            raise ValueError(f"entry_stage {self.entry_stage!r} not defined")

        for stage in self.stages:
            if stage.terminal:
                if stage.next_stage is not None:
                    raise ValueError(f"Terminal stage has next_stage")
                if stage.advance_on:
                    raise ValueError(f"Terminal stage has advance_on")
            else:
                if stage.next_stage is None:
                    raise ValueError(f"Non-terminal stage has no next_stage")
                if stage.next_stage not in seen:
                    raise ValueError(f"next_stage {stage.next_stage!r} not defined")

        return self
```

## Key conventions

- **Every field has `Field(description=...)`** — input models include `examples=`
- **Input ≠ output** — `SourceCreate` ≠ `SourceRead`, never reuse
- **Optional fields use `| None`** with `default=None`, never `Optional`
- **Discriminated unions** use `Annotated[Union[...], Field(discriminator="type")]`
- **Branch nodes call `model_rebuild()`** after the union alias to resolve forward references
- **Validators use `@model_validator(mode="after")`** for cross-field validation
- **`Field` constraints** (`ge=1`, `min_length=1`, `ne=0`) prevent invalid input at parse time
