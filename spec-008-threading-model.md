---
project: pixelscript
slug: threading-model
title: Concurrency model
description: Which code runs on which thread, why loading blocks the main thread, and how to protect shared state.
status: stable
tags: [api, threading, concurrency]
updated: 2026-07-28
parent: API reference
nav_order: 8
permalink: /threading-model/
---
# Concurrency model

PixelScript's threading model is built around one priority: **safe hot reloading**. Understanding it is the
difference between a server that reloads cleanly and one that develops mysterious race conditions.

## Script evaluation is always blocking and synchronous

The initial evaluation of a script, on boot and on every reload, happens **on the main server thread and
blocks it**. Everything at the top level of your file runs there.

That is a deliberate design choice, and it buys two things.

**Registration is safe.** You can register commands, listeners and tasks at the top level of your script
without wondering whether they are live yet. By the time evaluation returns, they are.

**Reloads are atomic from a gameplay perspective.** Blocking the main thread means players cannot interact
with the server while a script is mid-reload. If you are reloading a script that stops players from
breaking blocks in a protected region, there is no window where the protection is half-installed and
players can slip through.

The same applies at startup: script loading finishes before players can join, so the server is always in a
known-good state.

The cost is that a slow top-level block delays the server. Keep your top level to registration and cheap
setup. Anything expensive belongs in a scheduled task.

```javascript
// Runs on the main thread, blocking, during load. Keep it quick.
const config = new DataFile('scripts/resources/warps.yml');

registerCommand('warp', handleWarp);
registerListener($.PlayerJoinEvent, handleJoin);

// Expensive work: get it off the load path
Scheduler.runAsync(() => warmCaches());
```

## Everything else runs where the platform puts it

After evaluation, your functions are just functions. They run on whichever thread invokes them:

| Trigger | Thread |
|---------|--------|
| Command executor | Main thread |
| Event listener | Whatever thread fires the event (usually main; `AsyncChatEvent` and friends are not) |
| `Scheduler.runSync*` | Main thread |
| `Scheduler.runAsync*` | Background pool |
| `Script.addUnloadCallback` | Main thread, during unload |

Generated code is **thread safe by default** in the sense that the runtime itself will not corrupt, but it
does **not** synchronise your data. Two async tasks touching the same JS object concurrently is your
problem to solve, exactly as it would be in Java.

## The rule that matters

**Never touch game state from an async thread.** Players, worlds, entities, blocks, inventories and
scoreboards are all main-thread only. Bukkit will sometimes let you get away with it and sometimes throw,
and the failure mode when it does not throw is far worse.

The pattern:

```javascript
registerCommand('balance', (sender) => {
  const uuid = sender.getUniqueId().toString();   // main thread: read what you need

  Scheduler.runAsync(() => {
    const balance = queryBalance(uuid);            // async: do the slow thing

    Scheduler.runSync(() => {
      sender.sendRichMessage(`<green>Balance: ${balance}`); // main: apply the result
    });
  });
});
```

Note that the *arguments* are read on the main thread before the hop. Passing a `Player` into an async task
and calling methods on it there is the most common version of this bug.

## Blocking APIs

Two built-in APIs are synchronous and will block whichever thread calls them:

- [`Sql.*`](./spec-009-database.md), including the callbacks, which run inline
- [`fetch()`](./spec-010-fetch.md), which waits for the HTTP response

Both belong inside `Scheduler.runAsync`. Their callbacks fire on that same async thread, which is why the
examples in those articles always hop back with `runSync` before touching a player.

## Protecting shared state

Since PixelScript runs on the JVM, you get Java's `synchronized` semantics through the `Java` global:

```javascript
const lock = {};

Java.synchronized(function () {
  // Only one thread executes this at a time
  sharedCounter++;
}, lock);
```

Pass the object to lock on as the second argument. Every caller must lock on the *same* object for this to
do anything.

In practice you can usually avoid locking entirely:

- Keep mutable state on the main thread and only read immutable snapshots off it.
- Use `ConcurrentHashMap`-backed helpers such as [`GlobalMap` and `StaticStorage`](./spec-013-shared-state.md) when state genuinely has to be shared.
- Prefer "compute async, apply sync" over "mutate from anywhere".

## Reloads and in-flight work

When a script unloads, its scheduler tasks are cancelled and its listeners unregistered, but an async task
that is *already executing* will finish. If that task calls back into something the script owns, it may be
operating against a version of the world that no longer exists.

Guard against it where it matters:

```javascript
Scheduler.runAsyncLater(() => {
  if (!player.isOnline()) return;   // the world moved on while we waited
  applyRankUpgrade(player);
}, 20 * 5);
```
