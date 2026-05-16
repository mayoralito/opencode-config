# In-Memory Models

Dataclass and in-memory state patterns for designs that don't use a database.

---

## Nested dataclasses for player/entity state

```python
from dataclasses import dataclass, field
from typing import Dict, Set
from uuid import UUID


@dataclass
class AdventurePosition:
    adventure_ref: str
    step_index: int
    step_state: Dict[str, Any]   # mid-step scratch space (e.g., enemy HP)


@dataclass
class PlayerStatistics:
    enemies_defeated: Dict[str, int] = field(default_factory=dict)
    locations_visited: Dict[str, int] = field(default_factory=dict)
    adventures_completed: Dict[str, int] = field(default_factory=dict)


@dataclass
class PlayerState:
    player_id: UUID
    name: str
    character_class: str | None = None
    level: int = 1
    xp: int = 0
    hp: int = 20
    max_hp: int = 20
    prestige_count: int = 0
    current_location: str | None = None
    milestones: Set[str] = field(default_factory=set)
    statistics: PlayerStatistics = field(default_factory=PlayerStatistics)
    inventory: Dict[str, int] = field(default_factory=dict)
    equipment: Dict[str, str] = field(default_factory=dict)
    active_quests: Dict[str, str] = field(default_factory=dict)
    completed_quests: Set[str] = field(default_factory=set)
    active_adventure: AdventurePosition | None = None
    stats: Dict[str, int | float | str | bool | None] = field(default_factory=dict)
```

## Dynamically-typed stat container

```python
@dataclass
class CharacterConfig:
    """Defines the full set of stats that player entities can have."""

    public_stats: List[StatDefinition] = field(default_factory=list)
    hidden_stats: List[StatDefinition] = field(default_factory=list)


@dataclass
class StatDefinition:
    name: str
    type: Literal["int", "float", "str", "bool"]
    default: int | float | str | bool | None = None
    description: str = ""
```

## Content manifest as dataclass

```python
@dataclass
class LocationManifest:
    name: str
    display_name: str
    description: str = ""
    region: str | None = None
    unlock: Condition | None = None
    adventures: List[AdventurePoolEntry] = field(default_factory=list)


@dataclass
class AdventurePoolEntry:
    ref: str
    weight: int = 1
    requires: Condition | None = None
```

## Key conventions

- **Use `@dataclass`** for mutable in-memory state, `@dataclass(frozen=True)` for immutable config
- **`field(default_factory=...)`** for mutable defaults (dicts, sets, lists)
- **Nested dataclasses** for related state (statistics inside player state)
- **Typed dicts** (`Dict[str, int | float | str | bool | None]`) over `Dict[str, Any]` when the value type is known
- **Docstrings** on classes explaining purpose and usage constraints
- **Identity**: `UUID` for entities, `str` (kebab-case name) for content references
