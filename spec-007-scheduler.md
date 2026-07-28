---
project: pixelscript
slug: api-scheduler
title: Scheduler
description: Schedule synchronous and asynchronous tasks, with automatic cancellation on reload.
status: stable
tags: [api, scheduler, tasks, timing]
updated: 2026-07-28
parent: API reference
nav_order: 7
permalink: /api-scheduler/
---
# The `Scheduler` API

`Scheduler` wraps the platform scheduler with automatic cleanup and error handling. Use it to run code on
the main server thread, in the background, or repeatedly.

Every method returns a numeric **task id** you can pass to `Scheduler.cancel()`.

## Sync versus async

**Sync tasks** run on the main server thread. Anything touching players, worlds, entities, blocks or
inventories must be sync. That is not a PixelScript rule, it is a Bukkit rule.

**Async tasks** run on a background thread pool. Use them for database queries, HTTP requests, file I/O and
heavy computation.

> **Never touch game state from an async task.** Read a player's UUID, do your work off-thread, then hop
> back with `runSync` to apply the result.

```javascript
Scheduler.runAsync(() => {
  const balance = fetchBalanceFromDatabase(uuid);        // safe: off-thread work

  Scheduler.runSync(() => {
    player.sendRichMessage(`<green>Balance: ${balance}`); // safe: back on the main thread
  });
});
```

## The methods

| Method | Thread | When |
|--------|--------|------|
| `runSync(callback)` | main | Next tick |
| `runAsync(callback)` | background | Immediately |
| `runSyncLater(callback, delayTicks)` | main | After a delay |
| `runAsyncLater(callback, delayTicks)` | background | After a delay |
| `runSyncTimer(callback, delayTicks, periodTicks)` | main | Repeatedly |
| `runAsyncTimer(callback, delayTicks, periodTicks)` | background | Repeatedly |
| `cancel(taskId)` | | Cancel a scheduled task |

```javascript
// Once, next tick
Scheduler.runSync(() => Bukkit.broadcast(mm('<green>Task executed!')));

// Once, five seconds from now
Scheduler.runSyncLater(() => {
  const player = Bukkit.getPlayer('Notch');
  if (player !== null) player.teleport(spawnLocation);
}, 100);

// Every second, starting immediately
Scheduler.runSyncTimer(() => {
  Bukkit.getOnlinePlayers().forEach(updateScoreboard);
}, 0, 20);

// Every five minutes, first run after five minutes
Scheduler.runAsyncTimer(() => saveWorldData(), 20 * 60 * 5, 20 * 60 * 5);
```

## Cancelling

```javascript
const taskId = Scheduler.runSyncTimer(() => {
  console.log('This repeats forever... or does it?');
}, 0, 20);

Scheduler.runSyncLater(() => {
  Scheduler.cancel(taskId);
}, 200);
```

A timer that cancels itself needs the id in scope before the callback runs, which `const` handles fine
because the first invocation happens on a later tick:

```javascript
function startCountdown(seconds) {
  let remaining = seconds;

  const taskId = Scheduler.runSyncTimer(() => {
    if (remaining <= 0) {
      Scheduler.cancel(taskId);
      Bukkit.broadcast(mm('<red><bold>TIME IS UP!'));
      return;
    }

    Bukkit.broadcast(mm(`<yellow>${remaining} seconds remaining...`));
    remaining--;
  }, 0, 20);

  return taskId;
}
```

## Tick timing

Minecraft runs at 20 ticks per second when healthy. Under lag, ticks take longer, so **ticks are not a
reliable clock**. If you need wall-clock accuracy, record timestamps rather than counting ticks.

| Ticks | Time |
|-------|------|
| 1 | 50 ms |
| 20 | 1 second |
| 100 | 5 seconds |
| 1200 | 1 minute |
| 24000 | 20 minutes (one Minecraft day) |

Naming your intervals pays off quickly:

```javascript
const SECOND = 20;
const MINUTE = 20 * 60;

Scheduler.runSyncTimer(() => saveData(), 0, 5 * MINUTE);
```

## Practical patterns

### Delayed teleport that cancels on movement

```javascript
const pendingTeleports = new Map();

function teleportWithDelay(player, location, seconds) {
  const playerId = player.getUniqueId().toString();

  if (pendingTeleports.has(playerId)) {
    Scheduler.cancel(pendingTeleports.get(playerId));
  }

  player.sendRichMessage(`<yellow>Teleporting in ${seconds} seconds. Do not move.`);
  const startLocation = player.getLocation();

  const taskId = Scheduler.runSyncLater(() => {
    pendingTeleports.delete(playerId);

    if (player.getLocation().distance(startLocation) > 0.5) {
      player.sendRichMessage('<red>Teleport cancelled, you moved.</red>');
      return;
    }

    player.teleport(location);
    player.sendRichMessage('<green>Teleported.');
  }, seconds * 20);

  pendingTeleports.set(playerId, taskId);
}
```

### Spreading work over ticks

Doing everything at once on the main thread causes a spike. Batch it:

```javascript
function processBatched(items, batchSize, periodTicks) {
  let index = 0;

  const taskId = Scheduler.runSyncTimer(() => {
    const batch = items.slice(index, index + batchSize);

    if (batch.length === 0) {
      Scheduler.cancel(taskId);
      return;
    }

    batch.forEach(processItem);
    index += batchSize;
  }, 0, periodTicks);
}

processBatched(myLargeArray, 10, 40); // 10 items every 2 seconds
```

## Automatic cleanup

Every task scheduled through `Scheduler` is cancelled when the script unloads or reloads. Orphaned timers
from a previous version of your script are not a thing you have to think about.

This is the single biggest reason to use `Scheduler` instead of `Bukkit.getScheduler()` directly. Tasks you
schedule against the raw Bukkit scheduler survive reloads and will keep running against a dead script.

## Error handling

Exceptions thrown inside a task are caught, logged with a script stack trace, and swallowed. A repeating
task keeps repeating even if one iteration throws, so a listener that fails every tick will fill your log
rather than stop.

## Method aliases

For familiarity with Bukkit's own naming, each method has an alias. They are identical.

| Alias | Canonical |
|-------|-----------|
| `runTask()` | `runSync()` |
| `runTaskLater()` | `runSyncLater()` |
| `runTaskTimer()` | `runSyncTimer()` |
| `runTaskAsynchronously()` | `runAsync()` |
| `runTaskLaterAsynchronously()` | `runAsyncLater()` |
| `runTaskTimerAsynchronously()` | `runAsyncTimer()` |

Pick one convention per codebase.

## Related

- [The concurrency model](./spec-008-threading-model.md) for what runs where and why
- [Database](./spec-009-database.md), whose calls are blocking and belong in `runAsync`
- [HTTP fetch](./spec-010-fetch.md), same story
