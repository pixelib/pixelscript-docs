---
project: pixelscript
slug: import-export
title: Modules: import and export
description: Share code between scripts with ES6 named imports and exports, and understand how paths resolve.
status: stable
tags: [api, modules, import, export]
updated: 2026-07-28
parent: API reference
nav_order: 3
permalink: /import-export/
---
# Modules: import and export

PixelScript uses ES6 style modules to share code between scripts. Exports make functions, classes and
values available; imports pull them in.

> **Named exports only.** Default exports are not supported.

## Exporting

```javascript
// utils/messages.js
export function success(message) {
  return `<green>✓ <white>${message}`;
}

export function error(message) {
  return `<red>✗ <white>${message}`;
}
```

```javascript
// config/constants.js
export const SERVER_NAME = 'PSCraft';
export const MAX_PLAYERS = 100;
```

```javascript
// utils/economy.js
export class Economy {
  constructor() {
    this.accounts = new Map();
  }

  getBalance(uuid) {
    return this.accounts.get(uuid) || 0;
  }
}
```

You can also collect exports at the bottom of the file, which reads well for modules with a lot of small
helpers:

```javascript
function formatDuration(minutes) { /* ... */ }
function hasExpired(expiryTime) { /* ... */ }

export { formatDuration, hasExpired };
```

## Importing

```javascript
import { success, error } from '@/utils/messages';
import { SERVER_NAME } from '@/config/constants';
import { Economy } from './economy';

registerListener($.PlayerJoinEvent, (event) => {
  event.getPlayer().sendRichMessage(success(`Welcome to ${SERVER_NAME}`));
});
```

Import renaming (`import { func as myFunc }`) and wildcard imports (`import * as module`) are **not**
supported.

## Path resolution

This is where most confusion lives, so here is the complete set of rules.

| Form | Resolves relative to | Example |
|------|----------------------|---------|
| `@/path/to/file` | The `scripts` root | `@/utils/messages` |
| `/path/to/file` | The `scripts` root | `/utils/messages` |
| `./file` | The importing script's directory | `./economy` |
| `../file` | The importing script's directory | `../chat/messages` |
| `path/to/file` | The importing script's directory | `sys/shop` |

Two things worth internalising:

- **The `.js` extension is optional.** If you omit it the runtime tries `.js` first, then `.ts`. Both `'./economy'` and `'./economy.js'` work; pick one style and stick to it.
- **Use `@/` with the slash.** The root alias must be written `@/utils/messages`. Bare `@utils/messages` does not resolve correctly.

`@/` is strongly recommended for anything more than one directory away. It survives moving files around,
and it stops you from writing `'../../../../patches/chat/messages'`, which nobody can read.

The generated `tsconfig.json` already maps `@/*` to the scripts root, so your IDE follows these imports and
autocompletes across them.

## Lifecycle

An imported script is a **module** in the load tree. Its lifecycle differs from a
[watched script](./tutorial-003-scripts.md) in ways that matter.

**One instance, shared.** Two scripts importing the same module share a single evaluated instance,
including its top-level state. That makes an exported instance a genuine runtime singleton:

```javascript
// utils/economy.js
class EconomyManager {
  constructor() {
    this.accounts = new Map();
  }
  /* ... */
}

// Every importer gets this exact object
export const economy = new EconomyManager();
```

**Reloads cascade upward.** Editing a module reloads it, then every script that imports it, then their
children. A change to a module imported by fifty scripts reloads fifty scripts.

**Unloads when the last importer goes.** A module stays loaded as long as anyone depends on it.

### Isolate modules

Name a file `something.isolate.js` and it gets a **fresh instance for every importer** instead of being
shared. Use this when a module holds per-consumer state that must not be shared. It is the exception, not
the default, and it makes `Script.getCaller()` ambiguous, so reach for it deliberately.

## Avoiding reload cascades

The size of your reload tree is a design decision. Keep modules small and focused:

```text
BAD                                     GOOD
utils/common.js                         utils/messages.js
  (100 helpers, imported by 50 files)   utils/permissions.js
  → one edit reloads 50 scripts         utils/formatting.js
                                          → an edit reloads only what uses it
```

The other half of the rule: **features should be watched, not imported**. If a script's job is to register
commands and listeners, and it exports nothing anyone needs, load it with `Watcher.watch()`. Watchers are
barriers, so a reload stops there.

```javascript
// init-common.js

// Watched: reload barrier. Editing init-common.js does not reload this.
Watcher.watch('features/economy/init.js');
```

```javascript
// features/economy/init.js

// Imported: editing constants.js reloads this file too.
import { SERVER_NAME } from '@/config/constants';
```

## Circular imports

Circular imports are detected and rejected at load time with a readable trace
(`a.js > b.js > a.js`). If you hit one, the usual fix is to extract the shared piece into a third module
that both sides import.

## Worked example: a database wrapper

```javascript
// utils/player-data.js
export function loadPlayerData(uuid, callback) {
  Scheduler.runAsync(() => {
    Sql.makeQuery(
      'SELECT data FROM player_data WHERE uuid = ?',
      (stmt) => stmt.setString(1, uuid),
      (rs) => {
        const data = (rs !== null && rs.next()) ? JSON.parse(rs.getString('data')) : null;
        // Hop back to the main thread before touching game state
        Scheduler.runSync(() => callback(data));
      }
    );
  });
}
```

```javascript
// features/stats/join-greeting.js
import { loadPlayerData } from '@/utils/player-data';

registerListener($.PlayerJoinEvent, (event) => {
  const player = event.getPlayer();

  loadPlayerData(player.getUniqueId().toString(), (data) => {
    player.sendMessage(data ? `Welcome back! Level ${data.level}` : 'Welcome, new player!');
  });
});
```

## Best practices

**Do**

- Use named exports for everything
- Group related helpers into small, focused modules
- Use `@/` for anything outside the current directory
- Export singleton instances when you want shared state
- Import only what you need

**Do not**

- Use default exports, they are not supported
- Create circular dependencies
- Put everything in one giant `utils/common.js`
- Import modules from inside frequently called functions
- Forget that every edit cascades to all importers

## Related

- [Scripts and script management](./tutorial-003-scripts.md) for watched versus imported
- [`$` magic imports](./spec-002-magic-imports.md) for Java classes, which are a different mechanism entirely
- [TypeScript and type generation](./tips-003-typescript-and-types.md) for `.ts` modules
