# 🎮 Polymorphic RPG Game Engine State Blueprint

An enterprise-grade data configuration blueprint designed for deterministic, state-driven role-playing game engines. This architectural framework demonstrates advanced data modeling patterns using JSON Schema to enforce runtime polymorphism, recursive nesting constraints, and strict mathematical boundary invariants.

---

## 📐 Project Overview & Core Architecture

The core state engine of the RPG system converts complex, dynamic player and inventory states into deterministic inputs. This ensures that in-game actions, item equipment checks, and stat calculations run against uncorrupted data models.

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

This framework provides elegant, deterministic answers to three classic structural problems found in complex gaming data layers:

### 1. The Polymorphic Object Dilemma
* **The Problem:** Game items share core attributes (`item_id`, `name`, `slot`) but possess mutually exclusive properties depending on their type (e.g., a sword has `damage_type`, but a chestplate has `defense_rating`). Weak schemas allow items to manifest invalid combinations, like an armored shield that accidentally shoots magical bullets.
* **The Solution:** Implemented a robust object-level `oneOf` conditional array paired with explicit `not` constraints. All polymorphic variant fields are hoisted to the parent definition to maintain compilation visibility under strict mode, ensuring that if an item establishes its slot as `armor`, the engine strictly rejects weapon configurations.

### 2. The Recursive Nesting Stack Overflow
* **The Problem:** Equipment in RPGs often contains sockets for enhancement items (gems, runes), which are structurally identical to items themselves. Hardcoding nested layers locks down design flexibility and creates massive schema bloat.
* **The Solution:** Leveraged a **recursive pointer architecture**. By defining `socketed_gems` to reference `#/definitions/game_item` directly within its own definition, the validation engine evaluates gems using the exact same constraints as primary equipment, capped safely at a maximum array length of three iterations.

### 3. Co-Dependent Property Constraints
* **The Problem:** Certain items require specific minimum player parameters to equip, but entering incomplete requirements (like establishing a level restriction without specifying a class limitation) introduces catastrophic state bugs.
* **The Solution:** Utilized standard Draft-07 `dependencies` property mapping. If a `level` restriction constraint is specified inside an item's requirements configuration object, the validation engine automatically mandates a corresponding `class_restriction` string array.

---

## 📜 Production Schema Engine

The production-ready JSON Schema validating character states, polymorphic inventories, and recursive socket structures is embedded below:

```json
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
```

---

## 📊 System Validation Matrix

The following test payloads located in the `tests/` directory validate the engine's error-handling and compliance boundaries:

| Test File | Target Field | Payload State / Value | Expected Outcome | Validation Rule Enforced |
| :--- | :--- | :--- | :--- | :--- |
| `valid_character_state.json` | `character_id` | `"CHR-104922"` | **PASS** | Evaluates against regex pattern `^CHR-[0-9]{6}$` |
| `valid_character_state.json` | `attributes` | All integers between `1` and `100` | **PASS** | Enforces `minimum` and `maximum` numerical boundaries |
| `valid_character_state.json` | `inventory[0]` | `slot: "weapon"`, has `handedness` & `damage_type` | **PASS** | Complies with Variant 1 of the polymorphic `oneOf` block |
| `valid_character_state.json` | `socketed_gems[0]` | Re-evaluates generic item schema nested inside weapon | **PASS** | Successfully resolves recursive `#/$defs/game_item` reference |
| `invalid_character_state.json` | `character_id` | `"CHR-002144"` (Only 4 trailing digits instead of 6) | **FAIL** | Blocked by regex constraint pattern |
| `invalid_character_state.json` | `inventory[0]` | Declares `slot: "armor"` but provides `damage_type: "slashing"` | **FAIL** | Blocked by `oneOf` logical exclusion (`not: required: ["damage_type"]` for armor) |
| `invalid_character_state.json` | `requirements` | Declares `"level": 20` but omits `class_restriction` | **FAIL** | Blocked by `dependentRequired` validation rule |
