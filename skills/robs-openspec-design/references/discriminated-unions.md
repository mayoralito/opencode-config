# Discriminated Unions

Complex discriminated union patterns for Pydantic models — condition trees, step types, event types, and YAML-to-Python mapping.

---

## Condition tree (recursive discriminated union)

```python
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

class ItemCondition(BaseModel):
    type: Literal["item"]
    name: str

class CharacterStatCondition(BaseModel):
    type: Literal["character_stat"]
    name: str
    gt: int | float | None = None
    gte: int | float | None = None
    lt: int | float | None = None
    lte: int | float | None = None
    eq: int | float | None = None
    mod: ModComparison | None = None

    @model_validator(mode="after")
    def require_comparator(self) -> "CharacterStatCondition":
        if all(v is None for v in (self.gt, self.gte, self.lt, self.lte, self.eq, self.mod)):
            raise ValueError("character_stat must specify at least one comparator")
        return self

class ModComparison(BaseModel):
    divisor: int = Field(ge=1)
    remainder: int = Field(default=0, ge=0)

    @model_validator(mode="after")
    def validate_remainder_in_range(self) -> "ModComparison":
        if self.remainder >= self.divisor:
            raise ValueError(f"remainder must be in [0, divisor-1]")
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
        LevelCondition, MilestoneCondition, ItemCondition,
        CharacterStatCondition,
    ],
    Field(discriminator="type"),
]

AllCondition.model_rebuild()
AnyCondition.model_rebuild()
NotCondition.model_rebuild()
```

## Effects and steps (discriminated unions with mutual exclusion)

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
    """Effects fire first, then either `steps` runs or `goto` jumps."""
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

## YAML-to-Python condition normalizer

When YAML uses bare keys (`level: 3`) but Pydantic expects `type`-tagged form (`{"type": "level", "value": 3}`), a normalizer bridges the gap:

```python
def normalise_condition(raw: dict) -> dict:
    """Convert bare YAML condition keys to the type-tagged form Pydantic expects.

    e.g. {"level": 3} → {"type": "level", "value": 3}
         {"all": [...]} → {"type": "all", "conditions": [...]}

    If raw already has a "type" key it is returned as-is (already normalised).

    Raises ValueError if raw contains more than one recognised condition key —
    e.g. {"level": 3, "milestone": "foo"} is always an authoring mistake.
    """
    if "type" in raw:
        return raw

    # Simple leaf mappings: bare key → {"type": key, "value": val}
    SIMPLE_LEAVES = {"level", "milestone", "item", "class"}
    # Branch mappings: bare key → {"type": key, "conditions": val}
    BRANCH_KEYS = {"all": "conditions", "any": "conditions", "not": "condition"}
    # Complex leaf mappings: bare key → {"type": key, **rest}
    COMPLEX_LEAVES = {"character_stat", "prestige_count", "enemies_defeated",
                      "locations_visited", "adventures_completed"}

    matched = [k for k in raw if k in SIMPLE_LEAVES | BRANCH_KEYS | COMPLEX_LEAVES]
    if len(matched) > 1:
        raise ValueError(f"Condition has multiple keys: {matched} — likely an authoring error")
    if not matched:
        raise ValueError(f"Unrecognised condition: {raw}")

    key = matched[0]
    value = raw[key]

    if key in SIMPLE_LEAVES:
        return {"type": key, "value": value}
    elif key in BRANCH_KEYS:
        field = BRANCH_KEYS[key]
        return {"type": key, field: [normalise_condition(v) if isinstance(v, dict) else v for v in (value if key != "not" else [value])]}
    elif key in COMPLEX_LEAVES:
        return {"type": key, **value}

    return raw
```

## Key conventions

- **`type` as discriminator**: every leaf and branch node has a `Literal["type_name"]` field
- **`Field(discriminator="type")`**: enables Pydantic's fast union dispatch
- **`model_rebuild()`**: required on branch nodes that reference the union alias (forward references)
- **`@model_validator(mode="after")`**: enforces cross-field rules (e.g., "goto and steps are mutually exclusive")
- **`Field(ge=1)`, `Field(min_length=1)`, `Field(ne=0)`**: prevents obviously-invalid input at parse time
- **`default_factory=OutcomeBranch`**: provides sensible defaults for optional outcome branches
- **Normalizer**: keeps YAML ergonomic for authors while keeping Python models explicit
