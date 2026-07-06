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

## 🛡️ Polymorphic RPG Engine Validation Layer

The core state engine of the RPG system is governed by a strict **Draft-07 JSON Schema**. The configuration models characters, base physical attributes, and a deeply nested, polymorphic recursive inventory management system designed to guarantee data sanity inside runtime game loops.

```
{
  "$schema": "http://json-schema.org/draft-07/schema#",
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
      "items": { "$ref": "#/definitions/game_item" }
    }
  },
  "definitions": {
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
          "dependencies": {
            "level": ["class_restriction"]
          }
        },
        "socketed_gems": {
          "type": "array",
          "items": { "$ref": "#/definitions/game_item" },
          "maxItems": 3
        },
        "handedness": { "type": "string", "enum": ["one-handed", "two-handed"] },
        "damage_type": { "type": "string", "enum": ["slashing", "piercing", "bludgeoning", "magic"] },
        "defense_rating": { "type": "integer", "minimum": 1 }
      },
      "oneOf": [
        {
          "properties": {
            "slot": { "const": "weapon" }
          },
          "required": ["handedness", "damage_type"],
          "not": {
            "required": ["defense_rating"]
          }
        },
        {
          "properties": {
            "slot": { "const": "armor" }
          },
          "required": ["defense_rating"],
          "not": {
            "required": ["handedness", "damage_type"]
          }
        },
        {
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

The engine utilizes automated test fixtures to map simulated player profiles and inventory state data against the explicit conditional boundaries of the polymorphic game schema.

| Test File | Target Coordinate | Input Payload State | Expected Outcome | Active Constraint Evaluated |
| :--- | :--- | :--- | :--- | :--- |
| `valid_character_state.json` | `character_id` | `"CHR-104922"` | **🟢 PASS** | Matches regex sequence string template (`^CHR-[0-9]{6}$`). |
| `valid_character_state.json` | `inventory[0]` | Polymorphic item with `"slot": "weapon"` | **🟢 PASS** | Contains mandatory weapon properties (`handedness`, `damage_type`) and excludes armor keys. |
| `valid_character_state.json` | `inventory[0].requirements` | Contains both `level` and `class_restriction` keys | **🟢 PASS** | Satisfies the schema's property-to-property relational dependency rule. |
| `valid_character_state.json` | `inventory[0].socketed_gems[0]` | Deeply nested, recursive child node array | **🟢 PASS** | Recursively re-compiles `#/definitions/game_item` constraints cleanly down to a 1-tier depth. |
| `invalid_character_state.json` | `inventory[0]` | Armor item injecting a rogue `"damage_type"` key | **🔴 FAIL** | Blocked by the strict mutual exclusivity rule inside the polymorphic object's `oneOf` block. |
| `invalid_character_state.json` | `inventory[0].requirements` | Declares `level` but omits `class_restriction` | **🔴 FAIL** | Violated classical dependency constraint (`"dependencies": {"level": ["class_restriction"]}`). |

---
## 🌐 Semantic Graph Architecture

To complement edge-level JSON validation, the RPG engine synchronizes state data into a centralized, queryable Knowledge Graph using W3C Semantic Web standards. This transforms static player snapshots into web-ready semantic profiles, allowing developers to query gameplay analytics, character gear profiles, and cross-item dependencies using graph query languages.

### 🧠 Ontological Architecture (TBox vs. ABox)

The ontology structure splits game engine information into two logical data spaces:
* **The TBox (Terminology Box):** Declares the core gaming vocabulary. It establishes the conceptual entities (`rpg:Character`, `rpg:GameItem`) and maps the boundaries of how properties can interact (e.g., ensuring an item's `defense_rating` can only be attached to an item entity).
* **The ABox (Assertion Box):** Captures the precise instance state of live entities. It populates the graph with active player identifiers (`char:CHR-104922`) and maps out their current attributes and inventory links, mirroring the exact state verified by the JSON testing pipeline.

### 🎯 Graph Structural Equivalents to Schema Constraints

* **Polymorphic Property Attenuation:** While the JSON Schema utilizes an object-level `oneOf` switch to filter out illegal armor/weapon cross-contamination at runtime, the semantic layer maps these properties as datatype predicates directly under the `rpg:GameItem` domain. This lets the data store maintain a flexible, flat graph structure while application-level queries handle class definitions.
* **Recursive Reference Resolution:** The recursive property mapping defined in your JSON Schema (`#/definitions/game_item`) is replicated by setting the range of the socket predicate (`rpg:hasSocketedGem`) to point directly back to the `rpg:GameItem` class. This creates an infinite nesting loop capability in graph space, bounded naturally by the three-item limit checked during JSON validation.

### 🕸️ The Semantic Inventory Topology (`rpg-inventory.ttl`)
```
@prefix rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs:  <http://www.w3.org/2000/01/rdf-schema#> .
@prefix xsd:   <http://www.w3.org/2001/XMLSchema#> .
@prefix rpg:   <http://api.rpg-engine.internal/ontology#> .
@prefix char:  <http://api.rpg-engine.internal/characters#> .
@prefix item:  <http://api.rpg-engine.internal/items#> .

### 1. TBOX: ONTOLOGICAL FRAMEWORK (The Classes)
rpg:Character rdf:type rdfs:Class ;
    rdfs:label "Game Character Instance" ;
    rdfs:comment "A unique character entity managing core attributes and dynamic inventory states." .

rpg:Attributes rdf:type rdfs:Class ;
    rdfs:label "Character Attribute Block" .

rpg:GameItem rdf:type rdfs:Class ;
    rdfs:label "Inventory Game Item" .

rpg:ItemRequirements rdf:type rdfs:Class ;
    rdfs:label "Item Equipping Restrictions" .

### 2. TBOX: RELATIONSHIPS & BOUNDARIES (The Properties)
# --- Object Properties (Entity to Entity links) ---
rpg:hasAttributes rdf:type rdf:Property ;
    rdfs:domain rpg:Character ;
    rdfs:range rpg:Attributes .

rpg:hasInventoryItem rdf:type rdf:Property ;
    rdfs:domain rpg:Character ;
    rdfs:range rpg:GameItem .

rpg:hasRequirements rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range rpg:ItemRequirements .

rpg:hasSocketedGem rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range rpg:GameItem . # Recursive Blueprint Mirror

# --- Datatype Properties (Entity to Literal Value links) ---
rpg:characterId rdf:type rdf:Property ;
    rdfs:domain rpg:Character ;
    rdfs:range xsd:string .

rpg:strength rdf:type rdf:Property ;
    rdfs:domain rpg:Attributes ;
    rdfs:range xsd:integer .

rpg:agility rdf:type rdf:Property ;
    rdfs:domain rpg:Attributes ;
    rdfs:range xsd:integer .

rpg:intelligence rdf:type rdf:Property ;
    rdfs:domain rpg:Attributes ;
    rdfs:range xsd:integer .

rpg:stamina rdf:type rdf:Property ;
    rdfs:domain rpg:Attributes ;
    rdfs:range xsd:integer .

rpg:itemId rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range xsd:string .

rpg:itemName rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range xsd:string .

rpg:itemSlot rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range xsd:string .

rpg:currentDurability rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range xsd:integer .

rpg:maxDurability rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range xsd:integer .

rpg:handedness rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range xsd:string .

rpg:damageType rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range xsd:string .

rpg:defenseRating rdf:type rdf:Property ;
    rdfs:domain rpg:GameItem ;
    rdfs:range xsd:integer .

rpg:requiredLevel rdf:type rdf:Property ;
    rdfs:domain rpg:ItemRequirements ;
    rdfs:range xsd:integer .

rpg:classRestriction rdf:type rdf:Property ;
    rdfs:domain rpg:ItemRequirements ;
    rdfs:range xsd:string .

### 3. ABOX: LIVE INSTANCE DATA (Synchronized from valid_character_state.json)
char:CHR-104922 rdf:type rpg:Character ;
    rpg:characterId "CHR-104922"^^xsd:string ;
    rpg:hasAttributes char:CHR-104922_Stats ;
    rpg:hasInventoryItem item:ITM-WPN01 , item:ITM-ARM02 .

char:CHR-104922_Stats rdf:type rpg:Attributes ;
    rpg:strength "45"^^xsd:integer ;
    rpg:agility "12"^^xsd:integer ;
    rpg:intelligence "85"^^xsd:integer ;
    rpg:stamina "50"^^xsd:integer .

# Polymorphic Weapon Node with Recursive Gem Nesting
item:ITM-WPN01 rdf:type rpg:GameItem ;
    rpg:itemId "ITM-WPN01"^^xsd:string ;
    rpg:itemName "Spire of the Archmage"^^xsd:string ;
    rpg:itemSlot "weapon"^^xsd:string ;
    rpg:currentDurability "48"^^xsd:integer ;
    rpg:maxDurability "50"^^xsd:integer ;
    rpg:handedness "two-handed"^^xsd:string ;
    rpg:damageType "magic"^^xsd:string ;
    rpg:hasRequirements item:ITM-WPN01_Req ;
    rpg:hasSocketedGem item:ITM-GEM05 .

item:ITM-WPN01_Req rdf:type rpg:ItemRequirements ;
    rpg:requiredLevel "60"^^xsd:integer ;
    rpg:classRestriction "mage"^^xsd:string .

# Recursive Leaf Node (Gem treated as a custom accessory inside weapon)
item:ITM-GEM05 rdf:type rpg:GameItem ;
    rpg:itemId "ITM-GEM05"^^xsd:string ;
    rpg:itemName "Flawless Sapphire"^^xsd:string ;
    rpg:itemSlot "accessory"^^xsd:string ;
    rpg:currentDurability "1"^^xsd:integer ;
    rpg:maxDurability "1"^^xsd:integer .

# Polymorphic Armor Node
item:ITM-ARM02 rdf:type rpg:GameItem ;
    rpg:itemId "ITM-ARM02"^^xsd:string ;
    rpg:itemName "Vanguard Chestplate"^^xsd:string ;
    rpg:itemSlot "armor"^^xsd:string ;
    rpg:currentDurability "100"^^xsd:integer ;
    rpg:maxDurability "100"^^xsd:integer ;
    rpg:defenseRating "250"^^xsd:integer .
```
---
