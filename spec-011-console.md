---
project: pixelscript
slug: api-console
title: Console logging
description: Write messages, warnings and errors to the server console.
status: stable
tags: [api, console, logging, debug]
updated: 2026-07-28
parent: API reference
nav_order: 11
permalink: /api-console/
---
# Console logging

Every script gets standard logging globals. Output is prefixed with the script's path, so you can always
tell which file a line came from.

| Function | Level | Alias |
|----------|-------|-------|
| `console.log(...)` | INFO | `log(...)` |
| `console.warn(...)` | WARNING | `warn(...)` |
| `console.error(...)` | SEVERE | `error(...)` |

The short forms are the actual bindings; `console.*` is a thin wrapper provided for familiarity. They
behave identically, so pick one style per codebase.

```javascript
log('Economy module loaded');
warn('Database connection is slow');
error('Failed to load configuration file');
```

## Multiple arguments

All of them accept any number of arguments and join them with spaces:

```javascript
log('Player:', playerName, 'has', coins, 'coins');
// [scripts/economy.js] Player: Notch has 150 coins
```

## Logging objects

Arguments are converted with `String(...)`, so a plain object logs as `[object Object]`. Use
[`JSON.stringify`](./spec-012-json.md) when you want to see inside one:

```javascript
const playerData = { name: 'Notch', level: 50, coins: 1000 };

log(playerData);                  // [object Object]
log(JSON.stringify(playerData));  // {"name":"Notch","level":50,"coins":1000}
```

Java objects log using their `toString()`, which for most Bukkit types is genuinely useful:

```javascript
log(player.getLocation());
// Location{world=CraftWorld{name=world},x=100.5,y=64.0,z=-250.3,pitch=0.0,yaw=0.0}
```

## Logging is not free

Console writes are cheap but not free, and they land on whichever thread called them. Do not log per tick
or per player in a hot loop. If you are trying to find out how long something takes, use
[`Script.measure`](./spec-001-script.md) instead, which feeds the profiler rather than the log.

## What already gets logged for you

You do not need to wrap things in try/catch just to see failures. The runtime already logs, with a script
stack trace pointing at your source line:

- Uncaught errors in command executors, event listeners and scheduler tasks
- SQL errors from the [`Sql` API](./spec-009-database.md)
- Network errors from [`fetch`](./spec-010-fetch.md)
- Failures in unload callbacks
- Script load failures, including which line broke

Add your own logging for the things the runtime cannot see: business logic that went a way you did not
expect.

## Debugging patterns

```javascript
registerCommand('teleport', (sender, args) => {
  log('teleport by', sender.getName(), 'args:', args.join(' '));

  if (args.length < 3) {
    warn('teleport called with too few arguments');
    return;
  }
  // ...
});
```

```javascript
log('Economy module loading...');
// setup
log('Economy module loaded');

Script.addUnloadCallback(() => log('Economy module unloading'));
```

Lifecycle logging like this is worth keeping in permanently. When a reload cascade does something
unexpected, the boot order in your console is the fastest way to see what actually happened.
