# 🎮 Polymorphic RPG Game Engine State Blueprint

An enterprise-grade data configuration blueprint designed for deterministic, state-driven role-playing game engines. This architectural framework demonstrates advanced data modeling patterns using JSON Schema to enforce runtime polymorphism, recursive nesting constraints, and strict mathematical boundary invariants.

---

## 📐 Data Architecture Flow

```
         [ Raw Game State Payload ]
                     │
                     ▼
           [ Game Engine Core ]
                     │
                     ▼
    ┌────────────────────────────────────────┐
    │     RPGGameEngineSchema Validator      │
    │  ────────────────────────────────────  │
    │  • Enforces Regex Character IDs        │
    │  • Validates Attribute Bounds [1-100]  │
    │  • Evaluates Polymorphic Array Items   │
    │  • Resolves Recursive Socketed Gems    │
    └────────────────────────────────────────┘
                     │
            ┌────────┴────────┐
            │                 │
            ▼                 ▼
     [ VALID STATE ]   [ INVALID STATE ]
            │                 │
            ▼                 ▼
     [ State Committed ] [ Action Dropped ]
```
---

## 🧠 Core Engineering Challenges & Architectural Solutions

This schema provides elegant, deterministic answers to three classic structural problems found in gaming data layers:

### 1. The Polymorphic Object Dilemma
* **The Problem:** Game items share core attributes (`item_id`, `name`, `slot`) but possess mutually exclusive properties depending on their type (e.g., a sword has `damage_type`, but a shield has `defense_rating`). Poor schemas allow items to manifest invalid combinations, like an armored shield that shoots magical bullets.
* **The Solution:** Implemented a robust `oneOf` conditional array paired with explicit `not` constraints. This guarantees that if an item establishes its slot as `armor`, the engine strictly rejects any weapon or accessory keys at the gateway.

### 2. The Recursive Nesting Stack Overflow
* **The Problem:** Equipment in RPGs often contains sockets for enhancement items (gems, runes), which are technically items themselves. Hardcoding nested layers locks down flexibility and bloats schemas.
* **The Solution:** Leveraged a **recursive pointer architecture**. By defining `socketed_gems` to reference `#/$defs/game_item` directly within its own definition, the schema evaluates gems using the exact same constraints as primary equipment, capped safely at a maximum of three structural iterations.

### 3. Co-Dependent Property Constraints
* **The Problem:** Certain items require specific minimum player parameters to equip, but entering incomplete requirements (like establishing a level restriction without checking for class limits) introduces game-breaking state bugs.
* **The Solution:** Utilized `dependentRequired` property binds. If a level constraint is specified inside an item's configuration object, the validation engine automatically mandates a corresponding `class_restriction` block.

---

## 📜 Production Schema Engine

The production-ready JSON Schema validating character states, polymorphic inventories, and recursive socket structures is embedded below:

{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "RPGGameEngineSchema",
  "description": "Validates state, attributes, and polymorphic recursive inventory logic for an RPG engine.",
  "type": "object",
  "required": ["character_id", "attributes", "inventory"],
  "additionalProperties": false,
  "properties": {
    "character_id": {
      "type": "string",
      "pattern": "^CHR-[0-9]{6}$"
    },
    "attributes": {
      "type": "object",
      "required": ["strength", "agility", "intelligence", "stamina"],
      "additionalProperties": false,
      "properties": {
        "strength": { "type": "integer", "minimum": 1, "maximum": 100 },
        "agility": { "type": "integer", "minimum": 1, "maximum": 100 },
        "intelligence": { "type": "integer", "minimum": 1, "maximum": 100 },
        "stamina": { "type": "integer", "minimum": 1, "maximum": 100 }
      }
    },
    "inventory": {
      "type": "array",
      "items": { "$ref": "#/$defs/game_item" }
    }
  },
  "$defs": {
    "game_item": {
      "type": "object",
      "required": ["item_id", "name", "slot"],
      "additionalProperties": false,
      "properties": {
        "item_id": { "type": "string", "pattern": "^ITM-[A-Z0-9]{5}$" },
        "name": { "type": "string" },
        "slot": { "type": "string", "enum": ["weapon", "armor", "accessory"] },
        "durability": {
          "type": "object",
          "required": ["current", "max"],
          "additionalProperties": false,
          "properties": {
            "current": { "type": "integer", "minimum": 0 },
            "max": { "type": "integer", "minimum": 1 }
          }
        },
        "requirements": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "level": { "type": "integer", "minimum": 1 },
            "class_restriction": { "type": "string", "enum": ["warrior", "mage", "rogue"] }
          },
          "dependentRequired": {
            "level": ["class_restriction"]
          }
        },
        "socketed_gems": {
          "type": "array",
          "items": { "$ref": "#/$defs/game_item" },
          "maxItems": 3
        }
      },
      "oneOf": [
        {
          "dependentComment": "Variant 1: Weapon Sub-Schema",
          "properties": {
            "slot": { "const": "weapon" },
            "handedness": { "type": "string", "enum": ["one-handed", "two-handed"] },
            "damage_type": { "type": "string", "enum": ["slashing", "piercing", "bludgeoning", "magic"] }
          },
          "required": ["handedness", "damage_type"],
          "not": {
            "required": ["defense_rating"]
          }
        },
        {
          "dependentComment": "Variant 2: Armor Sub-Schema",
          "properties": {
            "slot": { "const": "armor" },
            "defense_rating": { "type": "integer", "minimum": 1 }
          },
          "required": ["defense_rating"],
          "not": {
            "required": ["handedness", "damage_type"]
          }
        },
        {
          "dependentComment": "Variant 3: Accessory Sub-Schema",
          "properties": {
            "slot": { "const": "accessory" }
          },
          "not": {
            "required": ["handedness", "damage_type", "defense_rating"]
          }
        }
      ]
    }
  }
}
