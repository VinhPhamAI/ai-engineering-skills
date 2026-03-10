---
name: software-architecture
description: Guide for quality focused software architecture. This skill should be used when users want to write code, design architecture, analyze code, in any case that relates to software development.
---

# Software Architecture Development Skill

This skill provides guidance for quality focused software development and architecture. It is based on Clean Architecture and Domain Driven Design principles.

## Code Style Rules

### General Principles

- **Early return pattern**: Always use early returns when possible, over nested conditions for better readability
- Avoid code duplication through creation of reusable functions and modules
- Decompose long (more than 80 lines of code) components and functions into multiple smaller components and functions. If they cannot be used anywhere else, keep it in the same file. But if file longer than 1000 lines of code, it should be split into multiple files.
- **JavaScript**: Use arrow functions instead of function declarations when possible
- **Python**: Use type hints for function parameters and return values

### Best Practices

#### Library-First Approach

- **ALWAYS search for existing solutions before writing custom code**
  - **JavaScript**: Check npm for existing libraries that solve the problem
  - **Python**: Check PyPI for existing packages that solve the problem
  - Evaluate existing services/SaaS solutions
  - Consider third-party APIs for common functionality
- **JavaScript**: Use libraries instead of writing your own utils or helpers. For example, use `cockatiel` instead of writing your own retry logic.
- **Python**: Use libraries instead of writing your own utils or helpers. For example, use `tenacity` instead of writing your own retry logic, `pydantic` for data validation, `pandas` for data manipulation.
- **When custom code IS justified:**
  - Specific business logic unique to the domain
  - Performance-critical paths with special requirements
  - When external dependencies would be overkill
  - Security-sensitive code requiring full control
  - When existing solutions don't meet requirements after thorough evaluation

#### Architecture and Design

- **Clean Architecture & DDD Principles:**
  - Follow domain-driven design and ubiquitous language
  - Separate domain entities from infrastructure concerns
  - Keep business logic independent of frameworks
  - Define use cases clearly and keep them isolated
- **Naming Conventions:**
  - **AVOID** generic names: `utils`, `helpers`, `common`, `shared`
  - **USE** domain-specific names: `OrderCalculator`, `UserAuthenticator`, `InvoiceGenerator`
  - Follow bounded context naming patterns
  - Each module should have a single, clear purpose
- **Separation of Concerns:**
  - Do NOT mix business logic with UI components
  - Keep database queries out of controllers
  - Maintain clear boundaries between contexts
  - Ensure proper separation of responsibilities

#### Anti-Patterns to Avoid

- **NIH (Not Invented Here) Syndrome:**
  - **JavaScript**: Don't build custom auth when Auth0/Supabase exists
  - **JavaScript**: Don't write custom state management instead of using Redux/Zustand
  - **JavaScript**: Don't create custom form validation instead of using established libraries
  - **Python**: Don't build custom auth when existing solutions exist (e.g., Django auth, FastAPI security)
  - **Python**: Don't write custom data validation instead of using Pydantic
  - **Python**: Don't create custom ORM when SQLAlchemy/Django ORM exists
- **Poor Architectural Choices:**
  - Mixing business logic with UI components
  - Database queries directly in controllers
  - Lack of clear separation of concerns
- **Generic Naming Anti-Patterns:**
  - **JavaScript**: `utils.js` with 50 unrelated functions
  - **JavaScript**: `helpers/misc.js` as a dumping ground
  - **JavaScript**: `common/shared.js` with unclear purpose
  - **Python**: `utils.py` with 50 unrelated functions
  - **Python**: `helpers.py` as a dumping ground
  - **Python**: `common.py` with unclear purpose
- Remember: Every line of custom code is a liability that needs maintenance, testing, and documentation

#### Code Quality

- Proper error handling with typed catch blocks
- Break down complex logic into smaller, reusable functions
- Avoid deep nesting (max 4 levels)
- Keep functions focused and under 300 lines when possible
- Keep files focused and under 1000 lines of code when possible
- Always consider YAGNI + SOLID + KISS + DRY principles when designing or adding new code