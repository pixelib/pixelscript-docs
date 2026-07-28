---
project: pixelscript
slug: api-bukkit
title: Bukkit and platform globals
description: Direct access to the Bukkit server instance, plus the small helpers that come with it.
status: stable
tags: [api, bukkit, server, globals]
updated: 2026-07-28
parent: API reference
nav_order: 4
permalink: /api-bukkit/
---
# Bukkit and platform globals

`Bukkit` is the live `org.bukkit.Server` instance, available in every script with no import. Everything on
the Bukkit `Server` interface is available on it, exactly as documented by Paper.

Alongside it, the Paper platform binds a handful of small conveniences.

| Global | Type | Purpose |
|--------|------|---------|
| `Bukkit` | `org.bukkit.Server` | The server instance |
| `__plugin` | `JavaPlugin` | The PixelScript plugin instance, for APIs that demand a `Plugin` |
| `newLocation(world, x, y, z, pitch?, yaw?)` | `org.bukkit.Location` | Build a `Location` from a world *name* |
| `DataFile` | class | YAML config files, see [shared state](./spec-013-shared-state.md) |
| `ArrayList`, `HashMap`, `HashSet`, `UUID`, `Gson`, `Arrays`, `List` | classes | Common Java types, preloaded |

## Working with players

```javascript
// Everyone online. Note this is a Java Collection, so size() not .length
const onlinePlayers = Bukkit.getOnlinePlayers();
console.log(`There are ${onlinePlayers.size()} players online`);

onlinePlayers.forEach(player => {
  player.sendRichMessage(`<green>Hello ${player.getName()}!`);
});

// Find one by name
const player = Bukkit.getPlayer('Notch');
if (player !== null && player.isOnline()) {
  player.sendRichMessage('<green>You are the chosen one!');
}

// Offline lookups by name or UUID
const offline = Bukkit.getOfflinePlayer('Notch');
console.log(offline.getUniqueId().toString());
```

`Bukkit.broadcastMessage(...)` still works but takes a legacy string. On Paper, prefer
`sendRichMessage` per player, or `Bukkit.broadcast(component)` with an Adventure component.

## Worlds and locations

```javascript
const world = Bukkit.getWorld('world');
if (world !== null) {
  console.log(`Time: ${world.getTime()}, players: ${world.getPlayers().size()}`);
  world.setTime(1000);
}

Bukkit.getWorlds().forEach(w => {
  console.log(`${w.getName()} (${w.getEnvironment()})`);
});
```

### `newLocation(worldName, x, y, z, pitch?, yaw?)`

Constructing a `Location` normally means resolving a `World` object first. This global does it for you and
throws a readable error when the world is not loaded.

```javascript
const spawn = newLocation('world', -57.5, 65.5, -1542.5);
const vipSpawn = newLocation('world', -57.5, 71, -1008.5, 0, 180);

player.teleport(spawn);
```

Note the argument order: `pitch` comes before `yaw`, which is the opposite of the `Location` constructor.

## Server information

```javascript
console.log(`Server version: ${Bukkit.getVersion()}`);
console.log(`Bukkit version: ${Bukkit.getBukkitVersion()}`);
console.log(`Max players: ${Bukkit.getMaxPlayers()}`);
console.log(`Online mode: ${Bukkit.getOnlineMode()}`);
```

## Dispatching commands

```javascript
Bukkit.dispatchCommand(Bukkit.getConsoleSender(), 'say Hello from PixelScript!');

// Make a player run a command as themselves
player.chat('/warp spawn');
```

For scheduling, use the [`Scheduler` global](./spec-007-scheduler.md) rather than
`Bukkit.getScheduler()`. It handles cleanup on reload and error reporting for you.

## Plugins and services

```javascript
const pluginManager = Bukkit.getPluginManager();

pluginManager.getPlugins().forEach(plugin => {
  console.log(`${plugin.getName()} v${plugin.getDescription().getVersion()}`);
});

const hasVault = pluginManager.getPlugin('Vault') !== null;
```

If your scripts depend on another plugin's API being present, do not guard on this at runtime. List the
plugin in `config.yml` under `required-plugin-dependencies` and PixelScript will not start scripts until it
has loaded. See [Configuration](./tutorial-002-configuration.md).

## `__plugin`

Some Java APIs insist on a `Plugin` instance, typically for registration. `__plugin` is the PixelScript
plugin, and it is what you hand them.

```javascript
const ProtocolLibrary = Script.loadClass('com.comphenix.protocol.ProtocolLibrary');
const listener = new MyPacketAdapter(__plugin, ListenerPriority.MONITOR, [/* ... */]);
ProtocolLibrary.getProtocolManager().addPacketListener(listener);
```

Anything you register against `__plugin` directly is **not** cleaned up on reload. Pair it with
[`Script.addUnloadCallback`](./spec-001-script.md).

## Preloaded Java collections

`ArrayList`, `HashMap`, `HashSet`, `UUID`, `Gson`, `Arrays` and `List` are bound as globals so you do not
have to load them. They are useful when a Bukkit API wants a real Java collection:

```javascript
const lore = new ArrayList();
lore.add(mm('<gray>A mysterious item'));
meta.lore(lore);
```

For your own bookkeeping, plain JS `Map`, `Set` and arrays are usually the better choice, and they avoid
boxing surprises with numeric keys. Use the Java types when Java is going to receive them.
