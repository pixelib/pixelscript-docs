---
project: pixelscript
slug: api-event-listener
title: Event listeners
description: Respond to Bukkit events with registerListener, including priorities and automatic cleanup.
status: stable
tags: [api, events, listener]
updated: 2026-07-28
parent: API reference
nav_order: 6
permalink: /api-event-listener/
---
# Event listeners

`registerListener` hooks your script into the Bukkit event system. It is the main way scripts react to
anything happening on the server.

## `registerListener(eventClass, callback, priority?)`

| Parameter | Description |
|-----------|-------------|
| `eventClass` | The Bukkit event class, for example `$.PlayerJoinEvent` |
| `callback` | Called with the event object when it fires |
| `priority` | Optional. Defaults to `$.EventPriority.NORMAL` |

```javascript
registerListener($.PlayerJoinEvent, (event) => {
  const player = event.getPlayer();
  player.sendRichMessage('<green>Welcome to the server!');
});
```

The callback receives the real event object, so everything in the Paper javadocs is available on it.

## Getting the event class

Common Bukkit events resolve through [`$`](./spec-002-magic-imports.md). Anything else, including Paper's
own newer events and events from other plugins, needs
[`Script.loadClass`](./spec-001-script.md):

```javascript
const AsyncChatEvent = Script.loadClass('io.papermc.paper.event.player.AsyncChatEvent');

registerListener(AsyncChatEvent, (event) => {
  event.renderer(myChatRenderer);
});
```

```javascript
const CustomPluginEvent = Script.loadClass('com.example.plugin.events.CustomPluginEvent');

registerListener(CustomPluginEvent, (event) => {
  console.log('Custom event fired!');
});
```

## Event priority

Priorities decide the order listeners run in when several of them handle the same event. Lowest runs first,
so it sees the event before anyone has modified it; monitor runs last and sees the final outcome.

- `$.EventPriority.LOWEST`
- `$.EventPriority.LOW`
- `$.EventPriority.NORMAL` (default)
- `$.EventPriority.HIGH`
- `$.EventPriority.HIGHEST`
- `$.EventPriority.MONITOR`

`MONITOR` is for observing only. Never modify the event or cancel it from a monitor listener.

```javascript
registerListener($.PlayerSpawnLocationEvent, (event) => {
  if (event.getPlayer().hasPermission('admin.spawn')) {
    event.setSpawnLocation(adminSpawn);
  }
}, $.EventPriority.HIGH);
```

## Cancelling events

Many events are cancellable. Cancelling stops the default behaviour from happening.

```javascript
registerListener($.BlockBreakEvent, (event) => {
  const loc = event.getBlock().getLocation();

  if (isInProtectedZone(loc)) {
    event.setCancelled(true);
    event.getPlayer().sendRichMessage('<red>You cannot break blocks here.</red>');
  }
});
```

Only cancellable events have `setCancelled`. Calling it on one that is not will throw.

## Threading

Listeners run on whatever thread fires the event. Most Bukkit events are on the main thread, so you can
touch game state freely. A few are not, notably `AsyncChatEvent` and `AsyncPlayerPreLoginEvent`.

Do not do slow work in a listener on the main thread. Hand it to
[`Scheduler.runAsync`](./spec-007-scheduler.md) and come back with `runSync` when you need to touch the
world again:

```javascript
registerListener($.PlayerJoinEvent, (event) => {
  const player = event.getPlayer();

  Scheduler.runAsync(() => {
    const rank = lookupRankInDatabase(player.getUniqueId().toString());

    Scheduler.runSync(() => {
      player.sendRichMessage(`<gray>Welcome back, <white>${rank}</white>.`);
    });
  });
});
```

See [the concurrency model](./spec-008-threading-model.md) for the full picture.

## Errors

If a listener throws, the error is caught, formatted with a script stack trace and logged. The event
continues to other listeners. A broken script does not take the server with it, but it also means a silent
bug can hide in the console, so read your logs.

## Automatic cleanup

Listeners registered this way are unregistered when the script unloads or reloads. You never need to
unregister them by hand, and you cannot leak a listener across a reload.

Anything you set up *around* the listener is still yours to clean up. Boss bars, spawned entities and
scoreboards need [`Script.addUnloadCallback`](./spec-001-script.md):

```javascript
const bossBar = Bukkit.createBossBar('Welcome!', $.BarColor.BLUE, $.BarStyle.SEGMENTED_6);

registerListener($.PlayerJoinEvent, (event) => bossBar.addPlayer(event.getPlayer()));
registerListener($.PlayerQuitEvent, (event) => bossBar.removePlayer(event.getPlayer()));

Script.addUnloadCallback(() => {
  bossBar.removeAll();
  bossBar.setVisible(false);
});

// Reloading mid-session should not leave current players out
Bukkit.getOnlinePlayers().forEach(player => bossBar.addPlayer(player));
```

That last line is a habit worth forming. A script that only reacts to join events will look broken every
time you reload with players already online.

## Worked example: spawn by permission

```javascript
const spawn = newLocation('world', -57.5, 65.5, -1542.5);
const vipSpawn = newLocation('world', -57.5, 71, -1008.5);

registerListener($.PlayerSpawnLocationEvent, (event) => {
  const player = event.getPlayer();

  if (player.hasPermission('spawn.bypass')) return;

  event.setSpawnLocation(player.hasPermission('spawn.vip') ? vipSpawn : spawn);
});
```
