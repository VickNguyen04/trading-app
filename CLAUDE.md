# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This repository is currently in the **spec-driven planning phase** — there is no application code, build system, package manifest, or test suite yet. Only documentation and specs exist (`docs/`, `specs/`). Do not assume a tech stack; confirm with the user before scaffolding any implementation.

## Spec-Driven Workflow

Work in this repo follows a fixed pipeline, one module at a time, moving artifacts from raw business knowledge to executable tasks:

1. **`docs/<module>-domain-knowledge.md`** — pure business rules for a module, written as input for the next step. Explicitly excludes diagrams, ERDs, and SQL schema (see `docs/order-management-domain-knowledge.md` header) — those belong to step 2.
2. **`specs/<module>/requirements.md`** — generated from the domain-knowledge doc. User stories in `As a... I want... so that...` form, each with Given-When-Then acceptance criteria. Every story/scenario carries a `Source: §X` reference back to the domain-knowledge doc section it derives from, and edge cases are tagged inline (e.g. `Edge case §8.1`) so nothing from the source doc is dropped silently.
3. **`specs/<module>/design.md`** (not yet created) — diagrams, ERD, SQL schema; the design step referenced by the domain-knowledge doc.
4. **`specs/<module>/tasks.md`** (not yet created) — implementation task breakdown derived from the design.
5. **`docs/adr/`** — Architecture Decision Records for cross-cutting technical decisions (currently empty).

When asked to produce one of these artifacts, follow the existing example under `specs/order-management/` for structure, tone, and traceability conventions rather than inventing a new format.

## Git Workflow

- Always commit after completing each major step (requirements/design/tasks).
- During the implementation phase, commit each task in `tasks.md` separately, using Conventional Commits (`feat`/`docs`/`fix`/`test`).
- Work on the feature branch (`feature/order-management`); never commit directly to `main`.
- Push to remote after every significant commit.
