---
title: API reference
nav_order: 5
has_children: true
permalink: /api/
---
# API reference

Everything the runtime binds into your scripts. All of it is available as a global, with no import.

## Core

| Article | Covers |
|---------|--------|
| [Script](./spec-001-script.md) | `Script.loadClass`, unload callbacks, runtime libraries, profiling hooks |
| [`$` magic imports](./spec-002-magic-imports.md) | The shorthand Java class resolver |
| [Modules: import and export](./spec-003-import-export.md) | Sharing code between scripts, path resolution, reload cascades |
| [Bukkit](./spec-004-bukkit.md) | The `Bukkit` server global and the platform helpers around it |

## Reacting to the server

| Article | Covers |
|---------|--------|
| [Commands](./spec-005-commands.md) | `registerCommand`, tab completion, aliases, middleware |
| [Event listeners](./spec-006-events.md) | `registerListener` and event priorities |
| [Scheduler](./spec-007-scheduler.md) | Sync and async tasks, timers, cancellation |
| [Concurrency model](./spec-008-threading-model.md) | What runs on which thread, and why loading blocks |

## Data and the outside world

| Article | Covers |
|---------|--------|
| [Database](./spec-009-database.md) | The `Sql` module, queries and transactions |
| [HTTP fetch](./spec-010-fetch.md) | Calling external APIs |
| [Console logging](./spec-011-console.md) | `log`, `warn`, `error` |
| [JSON](./spec-012-json.md) | Parsing and serializing |
| [Shared state and messaging](./spec-013-shared-state.md) | `StaticStorage`, `GlobalMap`, `GlobalNotification`, `DataFile` |
| [Implementing Java interfaces](./spec-014-java-implementations.md) | Handing a JavaScript implementation back to a Java plugin |
| [Redis](./spec-015-redis.md) | The `Redis` module, cross-server key/value, hashes, counters and pub/sub |

## The two you should read first

If you only read two articles here, make them
[Modules](./spec-003-import-export.md) and [the concurrency model](./spec-008-threading-model.md).
Between them they account for most of the bugs people hit in their first month.
