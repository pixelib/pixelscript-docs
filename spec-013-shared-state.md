---
project: pixelscript
slug: api-shared-state
title: Shared state and messaging
description: StaticStorage, GlobalMap, GlobalNotification and DataFile, for state that has to outlive a reload.
status: stable
tags: [api, state, storage, messaging, config]
updated: 2026-07-28
parent: API reference
nav_order: 13
permalink: /api-shared-state/
---
# Shared state and messaging

Module-level variables are the right default for state. They are simple, they are scoped, and a reload
clears them, which is usually exactly what you want.

Sometimes it is not. Four globals cover the cases where state has to cross a boundary:

| Global | Scope | Survives reload | Use for |
|--------|-------|-----------------|---------|
| [`StaticStorage`](#staticstorage) | Whole runtime, one flat namespace | Yes | A handful of values that must persist across reloads |
| [`GlobalMap`](#globalmap) | Whole runtime, named maps | Yes | The same, but namespaced |
| [`GlobalNotification`](#globalnotification) | Whole runtime, named channels | Listeners are cleaned up | Decoupled messaging between scripts |
| [`DataFile`](#datafile) | On disk | Yes | Configuration and data a human might edit |

## StaticStorage

A single process-wide `ConcurrentHashMap`, unaffected by script reloads.

```javascript
StaticStorage.set('economy.enabled', true);
const enabled = StaticStorage.get('economy.enabled');

// Get, or write and return the default if the key is absent
const cache = StaticStorage.getOrFillDefault('warp.cache', new HashMap());
```

`getOrFillDefault` is the useful one. It gives you a "create once, reuse across reloads" primitive:

```javascript
// Survives reloads of this script, so players do not lose their cooldowns
const cooldowns = StaticStorage.getOrFillDefault('teleport.cooldowns', new HashMap());
```

Two things to be careful about:

- It is **one flat namespace for the entire server**. Prefix your keys with your feature name.
- Values you leave in it are **never garbage collected**. Do not park player objects or large caches there and forget about them.

## GlobalMap

The same idea with namespacing built in. `GlobalMap.getInstance(name)` returns a named map, creating it on
first use.

```javascript
const warps = GlobalMap.getInstance('warps');

warps.put('spawn', spawnLocation);   // set() is an alias
const spawn = warps.get('spawn');
warps.remove('old-lobby');
```

Setting a key to `null` removes it.

Prefer `GlobalMap` over `StaticStorage` when you have more than a couple of values. The namespace stops two
features from colliding on a key name, and it makes intent obvious at the call site.

Both are backed by `ConcurrentHashMap`, so concurrent reads and writes are safe. Compound operations
("read, decide, write") are not atomic; see [the concurrency model](./spec-008-threading-model.md).

## GlobalNotification

A lightweight publish/subscribe bus, useful for letting features react to each other without importing each
other. Channels are identified by name.

```javascript
const channel = GlobalNotification.getInstance('economy:balance-changed');

channel.on((message) => {
  log(`Balance changed for ${message.uuid}: ${message.newBalance}`);
});
```

```javascript
// A completely different script, no import between the two
const channel = GlobalNotification.getInstance('economy:balance-changed');

channel.send({ uuid: player.getUniqueId().toString(), newBalance: 500 });
```

Method aliases, so you can use whichever vocabulary you prefer:

| Subscribe | Publish |
|-----------|---------|
| `registerListener(fn)` | `send(message)` |
| `subscribe(fn)` | `publish(message)` |
| `on(fn)` | `notify(message)` |

**`getInstance` must be called during initial script evaluation.** After a script finishes loading, the
extension locks and further `getInstance` calls throw. Grab your channel at the top level and keep it in a
`const`.

Listeners are unregistered automatically when the script unloads, so a reload does not leave stale
subscribers behind.

This is the right tool when the alternative would be a circular import, or when several unrelated features
need to hear about the same thing. It is the wrong tool for a direct call between two modules that already
know about each other; just import the function.

## DataFile

A wrapper over Bukkit's YAML configuration API, for data you want on disk in a form a human can edit.

```javascript
const warps = new DataFile('scripts/resources/warps.yml');
```

Paths are relative to `plugins/PixelScript`. The file and its parent directories are created if they do not
exist, and the contents are read into memory when you construct it.

| Method | Purpose |
|--------|---------|
| `getString(path)` / `setString(path, value)` | Strings |
| `getInt(path)` / `setInt(path, value)` | Integers |
| `getBoolean(path)` / `setBoolean(path, value)` | Booleans |
| `getStringList(path)` | A list of strings |
| `getLocation(path)` / `setLocation(path, location)` | A `Location`, stored as world/x/y/z/yaw/pitch |
| `getKeysInSection(path)` | The keys directly under a section, as a JS array |
| `hasSection(path)` | Whether a path is a section |
| `save()` | Write changes back to disk |

Paths are dot-separated: `'warps.spawn.name'`.

Changes live in memory until you call `save()`.

### Worked example: loading a warp list

```yaml
# plugins/PixelScript/scripts/resources/warps.yml
warps:
  spawn:
    name: Spawn
    world: world
    x: 0.5
    y: 64.0
    z: 0.5
    yaw: 90.0
    pitch: 0.0
  hub:
    name: Hub
    world: world
    x: 120.5
    y: 70.0
    z: -34.5
    yaw: 0.0
    pitch: 0.0
```

```javascript
const WARP_FILE = new DataFile('scripts/resources/warps.yml');
const WARP_CACHE = new HashMap();

const warpIds = WARP_FILE.getKeysInSection('warps');

for (let i = 0; i < warpIds.length; i++) {
  const warpId = warpIds[i];
  const name = WARP_FILE.getString(`warps.${warpId}.name`);

  WARP_CACHE.put(warpId, {
    id: warpId,
    name,
    nameLower: name.toLowerCase(),
    location: WARP_FILE.getLocation(`warps.${warpId}`),
  });
}

export function getWarp(id) {
  return WARP_CACHE.get(id);
}
```

Note the shape: read the file once at load time into an in-memory structure, then serve every lookup from
memory. `DataFile` reads are cheap but not free, and this pattern keeps disk access off your hot paths.
Because the file is re-read on every reload, editing the YAML and touching the script picks up the change.

### DataFile versus the Sql module

- **`DataFile`** for configuration, static content, and anything a server admin should be able to open in an editor. Reloading a script re-reads it.
- **[`Sql`](./spec-009-database.md)** for per-player data, anything that grows without bound, and anything that has to be shared between servers.

`DataFile` writes are not concurrency-safe and rewrite the whole file, so do not use it as a database.
