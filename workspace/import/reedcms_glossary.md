# ReedCMS Glossary & Core Concepts

## The Battle Cry

**Hwæt! UCG sceal ðe CMS woruld gewinnan!**  
*Lo! UCG shall conquer the CMS world!*

```
Hwæt! UCG sceal ðe CMS woruld gewinnan!
EPC sceal ðe file chaos bindan!
PostgreSQL sceal flat and swift bēon!
Redis sceal ðe UCG swift beran!
```

*Translation:*
```
Lo! UCG shall conquer the CMS world!
EPC shall bind the file chaos!
PostgreSQL shall be flat and swift!
Redis shall bear the UCG swiftly!
```

---

## Core Architecture Concepts

### UCG (Universal Content Graph)
**Definition:** The foundational pattern that eliminates database explosion by treating everything as entities with associations.

**Innovation:** Replaces traditional CMS table explosion (pages + posts + menus + categories + custom fields + search indexes) with a single unified pattern.

**Performance:** Predictable O(1) operations, no complex JOINs, infinite flexibility.

**Implementation:**
- **Redis Layer:** Live UCG cache for sub-millisecond queries
- **PostgreSQL UCG:** Compressed backup for recovery
- **CSV Layer:** Schema definitions as source of truth

### EPC (Explicit Path Chain)
**Definition:** ReedCMS's branded file resolution system using explicit path hierarchies for theme overrides.

**Innovation:** Self-building file resolution chains based on UCG context rather than hardcoded theme inheritance.

**Pattern:** Most specific file wins, falls back through explicit path chain to default.

**Example:**
```
Theme Chain: ["corporate", "corporate.berlin", "corporate.berlin.christmas"]
File Resolution:
1. themes/corporate.berlin.christmas/hero-banner/hero-banner.css  ← Most specific (wins)
2. themes/corporate.berlin/hero-banner/hero-banner.css           ← Fallback
3. themes/corporate/hero-banner/hero-banner.css                  ← Fallback
4. snippets/hero-banner/hero-banner.css                          ← Default
```

### 4-Layer Architecture
**Philosophy:** Anti-bloat by design through focused layer separation.

**Layers:**
1. **CSV Layer:** Configuration source of truth (Git-versionable)
2. **PostgreSQL Main:** Live content with ACID transactions
3. **PostgreSQL UCG:** Structure backup (compressed, recovery-only)
4. **Redis Layer:** Performance cache and search index

**Principle:** Each layer serves one focused purpose, preventing feature creep.

---

## Development Philosophy

### KISS-Brain Principle
**Definition:** Code should be mentally predictable - developers always know where data lives and how it flows.

**Application:** Every operation should be immediately understandable without complex mental models.

### 4k Demo Scene Philosophy
**Definition:** Every line of code has a clear purpose, inspired by demoscene efficiency.

**Principles:**
- No bloat, no unused features
- Minimal API surface with maximum power
- Elegant architecture without complexity
- Load exactly what you need, when you need it

### Anti-Flatterfat Rule
**Principle:** *"Je flatter die PostgreSQL, um so weniger flatterfat ist ihr Verhalten"*

**Translation:** The flatter PostgreSQL stays, the less chaotic its behavior becomes.

**Implementation:** Keep PostgreSQL brutally simple with flat tables, move complexity to UCG/Redis.

---

## Content Management Concepts

### Snippet
**Definition:** Reusable content component with type-safe field definitions.

**Components:**
- `.tera` template (required)
- `.js` Web Component (required)
- `.css` styling (required)
- `translations.{locale}.csv` (optional)

**Registry-Driven:** All snippets defined in CSV registry with validation rules.

### Meta-Snippet
**Definition:** Registry definition that describes snippet types, their fields, and behavior.

**Properties:**
- `is_routable`: Can have URL slug for routing
- `is_navigable`: Can appear in navigation menus
- `is_searchable`: Include in search index
- `required_fields`: Validation at creation
- `indexable_fields`: Database indexes for performance

### Semantic Name
**Definition:** Human-readable identifier with `$` prefix for CLI usage.

**Pattern:** `$variable_name_style` for existing entity references.

**Usage:** `reed snippet update $welcome_hero` instead of UUID references.

---

## Theme System

### Context Scoping
**Definition:** Multi-dimensional theme variations based on location, season, audience, etc.

**Pattern:** `base.dimension1.dimension2.dimension3`

**Examples:**
- `corporate.berlin` (location-specific)
- `corporate.christmas` (seasonal)
- `corporate.berlin.christmas.b2b` (multi-dimensional)

**Resolution:** EPC resolves files through context chain with intelligent fallbacks.

### Theme Chain
**Definition:** Ordered list of themes from most specific to default, built by UCG.

**UCG-Driven:** Theme hierarchies are content graph entities, not configuration files.

**Dynamic:** New contexts automatically integrated into resolution chain.

---

## Feature Toggle System

### Scope-Based Feature Toggles
**Innovation:** Feature toggles via theme/content scopes instead of conditional code logic.

**Template Features:** EPC file resolution serves different templates based on scope.

**Content Features:** UCG routes to different content IDs based on scope.

**Benefits:**
- Zero conditional code logic
- Clean separation of features
- Zero-downtime feature switching
- Easy rollback via scope assignment

### Parallel Content Versions
**Pattern:** Multiple content versions stored parallel in PostgreSQL with different UUIDs.

**UCG Routing:** Smart content routing based on scope without PostgreSQL complexity.

**Example:**
```sql
-- Multiple versions in PostgreSQL
INSERT INTO snippet_content (id, ...) VALUES ('content-v1', ...);
INSERT INTO snippet_content (id, ...) VALUES ('content-v2', ...);
```

```redis
# UCG routes based on scope
HSET entity:snippet:hero content_id "content-v1"      # Default
HSET entity:snippet:hero.v2 content_id "content-v2"   # Feature version
```

---

## Translation System

### Multi-Layer Translation Resolution
**Priority:** Global > Snippet > Plugin

**Files:**
- `config/translations.{locale}.csv` (global overrides)
- `snippets/{name}/translations.{locale}.csv` (snippet-specific)
- `plugins/{name}/snippets/{snippet}/translations.{locale}.csv` (plugin-specific)

### Template Integration
**Function:** `{{ string('translation.key') }}`

**Implementation:** Tera custom function with positional arguments for clean syntax.

**Resolution:** Automatic locale detection from template context.

---

## Plugin System

### Two-Language Architecture
**LUA Plugins:** Content processing, validation, workflows (5-200ms operations)

**Rust Pro Plugins:** AI/ML integration, high-performance operations (sub-ms to 30s)

### Unix Domain Sockets
**Communication:** Binary protocol over Unix domain sockets for zero overhead.

**Isolation:** Plugin crashes never affect core system stability.

**Performance:** Sub-millisecond command execution via native socket communication.

---

## Performance Targets

### Query Performance
```
Entity Queries:        0.1ms   (Redis direct lookup)
Content Queries:       1-2ms   (PostgreSQL, no JOINs)
Search Queries:        0.5ms   (Redis set operations)
Combined Operations:   1.5ms   (optimized multi-layer)
```

### Memory Usage
```
Small Site (1K snippets):     ~20MB total
Large Site (100K snippets):   ~1.3GB total
Linear scaling with no performance cliffs
```

### Scaling Characteristics
**Pattern:** Linear scaling with predictable performance - no query optimization guesswork needed.

---

## CLI Philosophy

### FreeBSD-Style Commands
**Pattern:** `command "target" / modifier value / modifier value`

**Chainable:** Commands flow like natural language with visual separators.

**Example:**
```bash
reed snippet create "hero-banner" /
  name $welcome_hero /
  with title "Welcome Message" /
  associate with page $homepage at path "home.hero"
```

### String Definitions vs References
**Definitions:** Quoted strings for new entities (`"hero-banner"`)

**References:** `$` prefix for existing entities (`$welcome_hero`)

**Consistency:** Same pattern across all CLI operations.

---

## Technical Acronyms

- **UCG:** Universal Content Graph
- **EPC:** Explicit Path Chain
- **CSV:** Comma-Separated Values (configuration source)
- **PG:** PostgreSQL
- **KISS:** Keep It Simple, Stupid (Brain variant: mentally predictable)
- **CMS:** Content Management System
- **CLI:** Command Line Interface

---

## Easter Eggs & Cultural References

### Altenglisch Elements
**þ (thorn):** Used in error codes and system messages
**ð (eth):** Alternative spelling in documentation
**ƿ (wynn):** Victory symbol in "UCG for ðe ƿin"

### Demo Scene Heritage
**4k Demo Influence:** Every byte counts, perfection through simplicity
**Code Quality:** Stunning code for code reviews
**Performance:** Sub-millisecond operations as art form

---

*"UCG for ðe ƿin - where elegant architecture meets linguistic beauty"* 🏆