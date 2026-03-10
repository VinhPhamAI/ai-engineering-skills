---
name: software-architecture
description: >
  Guide for quality-focused software architecture and development. Use this skill whenever the user
  writes code, designs system architecture, reviews or analyzes existing code, asks about project
  structure, naming conventions, design patterns, testing strategy, tech stack decisions, or any
  aspect of software development — even casual coding questions. Trigger on: "how should I structure
  this", "is this good code", "design this feature", "review my architecture", "should I use X or Y
  pattern", "help me build", "refactor this", or any request to write non-trivial code.
---

# Software Architecture Development Skill

Guides high-quality software design using Clean Architecture, Domain-Driven Design (DDD), and
modern engineering best practices. Covers JavaScript/TypeScript and Python ecosystems.

---

## Step 1 — Pick the Right Architecture

Use this decision tree **before** prescribing any approach:

```
Is the domain logic complex with many business rules?
├── YES → Is the team large or the system long-lived?
│         ├── YES → Clean Architecture + DDD (→ read clean-architecture.md, ddd-patterns.md)
│         └── NO  → Modular Monolith with DDD patterns (bounded modules, no full layering)
└── NO  → Is this primarily CRUD / data-in-data-out?
          ├── YES → Simple Layered Architecture (controller → service → repository)
          └── NO  → Evaluate: Hexagonal Architecture if many external integrations
```

**Warning signs of over-engineering:**
- Applying Clean Architecture to a <5 endpoint CRUD API
- Using CQRS without a clear read/write asymmetry problem
- Creating 5+ layers for a script or small tool
- Mapping every table to a domain aggregate

**When complexity grows, migrate incrementally** — start simple, extract layers as the domain clarifies.

---

## Step 2 — Non-Negotiable Code Rules

These apply regardless of architecture chosen:

### Naming
- **NO** generic names: `utils`, `helpers`, `common`, `misc`, `shared`
- **YES** domain-specific names: `OrderCalculator`, `InvoiceGenerator`, `UserAuthenticator`
- Follow bounded-context naming: same concept = same name everywhere in that context
- JS/TS: `camelCase` for variables/functions, `PascalCase` for classes/types/components
- Python: `snake_case` for variables/functions, `PascalCase` for classes

### Code Structure
- **Early return pattern** — always prefer guard clauses over nested conditions
- Max nesting depth: **4 levels**
- Functions: focused, ideally **<80 lines**, hard limit **300 lines**
- Files: ideally **<300 lines**, hard limit **1000 lines** (split if exceeded)
- No code duplication — extract reusable functions; if not reusable elsewhere, keep in same file

### Language-Specific
- **JS/TS**: Arrow functions over `function` declarations; strict TypeScript types, avoid `any`
- **Python**: Type hints on all function params and return values; use `dataclasses` or `pydantic` for data models

### Library-First
Before writing custom code, search for existing solutions:
- **JS/TS**: Check npm — e.g., use `cockatiel` for retry, `zod` for validation, `date-fns` for dates
- **Python**: Check PyPI — e.g., use `tenacity` for retry, `pydantic` for validation, `pandas` for data

Custom code is justified only for: unique business logic, extreme performance requirements, security-sensitive paths, or when no library fits.

---

## Step 3 — Go Deeper (Read Reference Files)

| Topic | When to Read |
|---|---|
| `references/clean-architecture.md` | Designing layers, dependency rules, use cases, ports & adapters |
| `references/ddd-patterns.md` | Modeling domain: entities, aggregates, value objects, bounded contexts |
| `references/design-patterns.md` | CQRS, event-driven, saga, repository, factory patterns |
| `references/code-quality.md` | SOLID, error handling, testing pyramid, layer-specific testing |
| `references/anti-patterns.md` | What to avoid: God class, anemic model, NIH syndrome, leaky abstractions |

Read **only the files relevant** to the task at hand. For a simple refactor, `code-quality.md` alone may suffice. For a full system design, read all five.

---

## Quick Anti-Pattern Checklist

Before finalizing any design, verify:
- [ ] No business logic in controllers or route handlers
- [ ] No database queries directly in UI/presentation layer
- [ ] No `any` type in TypeScript (or untyped Python functions)
- [ ] No functions longer than 300 lines
- [ ] No files longer than 1000 lines  
- [ ] No custom auth/validation/ORM when a library exists
- [ ] Domain layer has zero infrastructure dependencies