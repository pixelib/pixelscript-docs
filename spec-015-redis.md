---
project: pixelscript
slug: api-redis
title: Redis
description: Cross-server key/value storage, hashes, atomic counters and pub/sub through the built-in Redis module.
status: stable
tags: [api, redis, storage, messaging, network]
updated: 2026-08-08
parent: API reference
nav_order: 15
permalink: /api-redis/
---
# The `Redis` API

The `Redis` global gives your scripts a pooled connection (via Redisson) to a Redis server: string/hash
key-value storage, atomic counters, and pub/sub. Unlike [`GlobalMap`](./spec-013-shared-state.md#globalmap)
and [`GlobalNotification`](./spec-013-shared-state.md#globalnotification), which only work within a single
JVM, `Redis` is the tool for state and messaging that has to cross **multiple servers** in a network.

The module is optional and disabled by default. Configuration lives in `redis.yml`. See
[Configuration](./tutorial-002-configuration.md).

## Checking availability

`Redis.isAvailable()` tells you whether `enabled: true` is set and the connection is up. Guard any script
that has a hard dependency on it:

```javascript
if (!Redis.isAvailable()) {
  throw new Error('Redis is not available');
}
```

Throwing at the top level marks the script as failed and shows up in `/script list`. Do not silently
degrade a feature that needs cross-server state into one that only half-works per-server.

## Key/value operations

### `Redis.set(key, value)` / `Redis.get(key)`

Plain string storage.

```javascript
Redis.set('server:lobby-1:motd', 'Welcome!');
const motd = Redis.get('server:lobby-1:motd');   // null if the key doesn't exist
```

### `Redis.setEx(key, value, seconds)`

Same as `set`, but the key expires itself after `seconds`. The natural fit for cooldowns, temporary bans, or
one-time tokens, without you having to clean anything up.

```javascript
Redis.setEx(`cooldown:${uuid}:teleport`, 'true', 30);
```

### `Redis.exists(key)` / `Redis.delete(key)`

```javascript
if (Redis.exists(`ban:${uuid}`)) {
  player.kickPlayer('You are banned network-wide.');
}

Redis.delete(`cooldown:${uuid}:teleport`);   // true if a key was actually removed
```

## Hash operations

A hash is a map of fields under one key, good for structured per-entity data (a player's stats, an item's
metadata) without exploding into one Redis key per field.

```javascript
Redis.hset(`player:${uuid}:stats`, 'kills', String(kills));
Redis.hset(`player:${uuid}:stats`, 'deaths', String(deaths));

const kills = Redis.hget(`player:${uuid}:stats`, 'kills');
const stats = Redis.hgetAll(`player:${uuid}:stats`);   // { kills: '12', deaths: '4' }
```

Values are strings on the wire, same as `set`/`get`. Parse numbers yourself with `Number(...)`.

## Atomic counters

`Redis.increment(key)` and `Redis.decrement(key)` are atomic across every server connected to the same
Redis instance, which makes them the right tool for currency, cross-server unique IDs, or shared rate
limits, where two servers writing to the same key from `get`/`set` would race.

```javascript
const newBalance = Redis.increment(`balance:${uuid}`);
const remaining = Redis.decrement(`votes-remaining:${uuid}`);
```

## Pub/Sub

`Redis.publish` and `Redis.subscribe` broadcast between every server connected to the same Redis instance,
which is what makes this the cross-server counterpart to
[`GlobalNotification`](./spec-013-shared-state.md#globalnotification) (single JVM only).

### `Redis.publish(channel, message)`

Returns the number of subscribers that received the message, across every connected server.

```javascript
Redis.publish('chat:global', JSON.stringify({ player: sender.getName(), text: message }));
```

### `Redis.subscribe(channel, handler)`

`handler` is `(channel, message) => void`. Returns a listener id, needed to unsubscribe.

```javascript
const listenerId = Redis.subscribe('chat:global', (channel, message) => {
  const { player, text } = JSON.parse(message);
  Bukkit.broadcast(`<${player}> ${text}`);
});
```

Subscriptions registered this way are **automatically cleaned up when the script unloads**, same guarantee
`GlobalNotification` gives you. You do not need an unload callback for this. Unsubscribe manually only if
you need to stop listening before that:

```javascript
Redis.unsubscribe('chat:global', listenerId);
```

### Worked example: cross-server chat

```javascript
// features/chat/relay.js
if (!Redis.isAvailable()) throw new Error('Redis is not available');

const SERVER_NAME = 'lobby-1';

Redis.subscribe('chat:global', (channel, message) => {
  const payload = JSON.parse(message);
  if (payload.origin === SERVER_NAME) return;   // don't echo our own messages back

  Bukkit.broadcast(`<gray>[${payload.origin}]</gray> <${payload.player}> ${payload.text}`);
});

registerListener($.AsyncChatEvent, (event) => {
  Redis.publish('chat:global', JSON.stringify({
    origin: SERVER_NAME,
    player: event.getPlayer().getName(),
    text: event.message(),
  }));
});
```

## Best practices

**Do**

- Guard the module with `Redis.isAvailable()` at load time
- Use `increment`/`decrement` instead of read-modify-write for anything two servers could touch at once
- Namespace keys by feature and entity, e.g. `feature:entity-id:field`
- Use `setEx` for anything that should expire instead of tracking expiry yourself

**Do not**

- Assume `get`/`hget` succeeded without checking for `null`
- Treat `Redis` as a database for data that needs querying, filtering or joins, that is what
  [`Sql`](./spec-009-database.md) is for
- Store large blobs; Redis is for hot, small, frequently accessed state

## Redis versus the other shared-state globals

- **`GlobalMap` / `GlobalNotification`** for state and messaging within a single server process. No network
  hop, no external dependency, but invisible to every other server in the network.
- **`Redis`** for the same shapes of problem the moment more than one server needs to see them.
- **[`Sql`](./spec-009-database.md)** for data that needs to be queried, filtered, or joined, not just
  fetched by key.

See [Shared state and messaging](./spec-013-shared-state.md) for the single-JVM globals.
