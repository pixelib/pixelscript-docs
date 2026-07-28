---
project: pixelscript
slug: configuration
title: Configuration
description: The three PixelScript configuration files and what each of them controls.
status: stable
tags: [intro, runtime, minecraft, config]
updated: 2026-07-28
parent: Tutorial
nav_order: 2
permalink: /configuration/
---
# Configuration

PixelScript has three configuration files, all in `plugins/PixelScript`. None of them are required reading
to get started, but each one solves a specific problem you will eventually hit.

| File | Controls |
|------|----------|
| `config.yml` | Which plugins must load before scripts start |
| `class-searchcontext.yml` | Which packages the [`$` resolver](./spec-002-magic-imports.md) searches |
| `database.yml` | The [Sql module](./spec-009-database.md) connection |

## config.yml: startup order

Scripts are loaded as soon as the PixelScript plugin is enabled. Sometimes that is too early, because your
scripts depend on another plugin's API being ready.

List those plugins here and PixelScript will wait for them:

```yaml
# Wait until these plugins are loaded before we start to initialize scripts
required-plugin-dependencies:
  #- 'Vault'
  #- 'LuckPerms'

# Force kill the server if the first tick happened while dependencies listed above aren't ready.
terminate-when-started-with-missing-dependencies: true
```

Three things to know:

- There is **no timeout** on the wait itself. A plugin that never enables means scripts never load. Spelling counts.
- `terminate-when-started-with-missing-dependencies` decides what happens in that case. With it on (the default), if the server reaches its first tick with no scripts loaded, PixelScript logs why and shuts the server down rather than leaving it running in an unknown state with none of your logic active. Turn it off only if a scriptless server is genuinely acceptable to you.
- Loading is still blocking. Once all listed plugins are ready, scripts are evaluated on the main thread, which blocks the first tick. That is deliberate; see [the concurrency model](./spec-008-threading-model.md) for why.

A server that shuts itself down right after boot with a warning about missing dependencies is almost always
a typo in this list, not a real dependency problem.

## class-searchcontext.yml: the `$` resolver

The `$` global lets you grab a Java class by simple name instead of a fully qualified one. This file lists
the packages it searches:

```yaml
# These are the packages that will be searched in
# when using the "$." resolver.
# You can add your own top-level packages here.
search-packages:
  - 'org.bukkit'
  - 'com.destroystokyo.paper'
  - 'org.spigotmc'
```

Add your own plugin's package here and `$.UserManager` will resolve to your service. Full details in
[Magic imports](./spec-002-magic-imports.md).

## database.yml: the Sql module

PixelScript ships with a built-in SQL module backed by HikariCP connection pooling. It defaults to a SQLite
file in the plugin folder, which is fine for a single server. Point it at MySQL/MariaDB for a network.

```yaml
# Database type (sqlite or mysql)
type: sqlite

# SQLite Configuration
sqlite:
  # The database file name (located in the plugin folder)
  file: database.db

# MySQL Configuration
mysql:
  host: localhost
  port: 3306
  database: pixelscript
  username: root
  password: ""

# Connection Pool Settings
connection_pool:
  # Number of connections to maintain in the pool.
  # For SQLite this is always 1, regardless of this setting.
  size: 10

  # Connection timeout in milliseconds
  timeout: 30000
```

Scripts reach this through the `Sql` global. See [the database API](./spec-009-database.md) for queries,
updates, and transactions.

## Per-feature configuration

For your own feature configuration, do not add to these files. Use the `DataFile` global instead, which
gives you a YAML file per feature with a small typed accessor API:

```javascript
const warps = new DataFile('scripts/resources/all_warps.yml');
const spawn = warps.getLocation('warps.spawn');
```

Documented in [Shared state and messaging](./spec-013-shared-state.md).

## Next up

[Scripts and script management](./tutorial-003-scripts.md), where we finally write some code.
