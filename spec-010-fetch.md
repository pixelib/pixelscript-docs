---
project: pixelscript
slug: api-fetch
title: HTTP fetch
description: Call external APIs and web services from your scripts.
status: stable
tags: [api, http, fetch, web, rest]
updated: 2026-07-28
parent: API reference
nav_order: 10
permalink: /api-fetch/
---
# The `fetch` API

`fetch` makes HTTP requests from your scripts. Use it for REST APIs, webhooks, and anything else that lives
outside the server.

> **`fetch` is synchronous and blocks until the response arrives.** Always call it inside
> `Scheduler.runAsync()`. Calling it on the main thread freezes your server for as long as the remote
> service takes to answer, up to the 30 second timeout.

It is deliberately *not* the browser `fetch`. There are no promises, no `await`, and the response object
has its own small API.

## `fetch(url, options?): Response`

| Option | Description |
|--------|-------------|
| `method` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE`. Defaults to `GET` |
| `headers` | Object of header name to value |
| `postBody` | Request body, sent with `POST`, `PUT` and `PATCH` |

```javascript
Scheduler.runAsync(() => {
  const response = fetch('https://gateway.openaudiomc.net/');

  if (!response.isSuccessful()) {
    warn(`Request failed with status ${response.getStatusCode()}`);
    return;
  }

  const data = response.asObject();
  log(`MOTD: ${data.motd}`);
});
```

## The `Response` object

| Method | Returns |
|--------|---------|
| `asString()` | The raw response body |
| `asObject()` | The body parsed as JSON, or `null` if parsing failed |
| `getStatusCode()` | The HTTP status code, or `-1` on a network failure |
| `isSuccessful()` | `true` when the status is between 1 and 399 |
| `isFailed()` | `true` when the status is negative or 400 and above |

A status code of `-1` means the request never completed: DNS failure, connection refused, or a timeout.
That distinction is worth surfacing in your logs:

```javascript
if (response.isFailed()) {
  if (response.getStatusCode() < 0) {
    error('Network error or timeout');
  } else {
    error(`Request failed: HTTP ${response.getStatusCode()}`);
  }
}
```

Error responses still have a readable body, so `asString()` on a 4xx gives you the API's explanation.

## Sending data

```javascript
Scheduler.runAsync(() => {
  const payload = {
    server: 'PSCraft',
    event: 'player_join',
    player: playerName,
  };

  const response = fetch('https://api.example.com/events', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${apiToken}`,
    },
    postBody: JSON.stringify(payload),
  });

  if (!response.isSuccessful()) {
    warn(`Event dispatch failed: ${response.getStatusCode()}`);
  }
});
```

`PUT`, `PATCH` and `DELETE` work the same way. Only `POST`, `PUT` and `PATCH` send `postBody`.

## Coming back to the main thread

The response arrives on the async thread. Anything touching game state has to hop back:

```javascript
Scheduler.runAsync(() => {
  const response = fetch(`https://api.mojang.com/users/profiles/minecraft/${name}`);
  if (!response.isSuccessful()) return;

  const profile = response.asObject();

  Scheduler.runSync(() => {
    sender.sendRichMessage(`<green>UUID: <white>${profile.id}`);
  });
});
```

See [the concurrency model](./spec-008-threading-model.md).

## Parsing responses

`asObject()` catches its own parse errors and returns `null` rather than throwing, so check the result:

```javascript
const data = response.asObject();

if (data === null) {
  error('Response was not valid JSON');
  log(`Raw body: ${response.asString()}`);
  return;
}
```

If you need finer control, take `asString()` and use [`JSON.parse`](./spec-012-json.md) in a `try`/`catch`.

## Timeouts

Both connection and read timeouts are fixed at **30 seconds**. A request that exceeds them fails with
status `-1`. There is no way to raise this from a script, so if you are talking to a slow service, do the
work on a timer rather than in response to a player action.

## Rate limiting

There is no built-in rate limiting. If you are calling an API on a schedule or per player, protect it
yourself:

```javascript
const lastCall = new Map();
const COOLDOWN_MS = 60_000;

function fetchProfileCached(uuid, callback) {
  const now = Date.now();

  if (lastCall.has(uuid) && now - lastCall.get(uuid) < COOLDOWN_MS) {
    return callback(null);
  }

  lastCall.set(uuid, now);

  Scheduler.runAsync(() => {
    const response = fetch(`https://api.example.com/profile/${uuid}`);
    callback(response.isSuccessful() ? response.asObject() : null);
  });
}
```

## Best practices

**Do**

- Wrap every call in `Scheduler.runAsync()`
- Check `isSuccessful()` before reading the body
- Handle `asObject()` returning `null`
- Keep tokens in a `DataFile` or an environment variable, not in the script
- Set `Content-Type` when you send a body

**Do not**

- Call `fetch` on the main thread
- Call it per tick, or once per player in a loop over everyone online
- Assume the response is valid JSON
- Hardcode credentials
