---
project: pixelscript
slug: api-json
title: JSON
description: Parse and serialize JSON for storage, API communication and data interchange.
status: stable
tags: [api, json, data, parsing]
updated: 2026-07-28
parent: API reference
nav_order: 12
permalink: /api-json/
---
# JSON

`JSON.stringify` and `JSON.parse` work exactly as they do in any JavaScript runtime. They are the standard
way to persist structured data and talk to external APIs.

```javascript
const playerData = {
  name: 'Notch',
  level: 50,
  inventory: ['diamond_sword', 'golden_apple'],
};

const json = JSON.stringify(playerData);
// {"name":"Notch","level":50,"inventory":["diamond_sword","golden_apple"]}

const parsed = JSON.parse(json);
log(parsed.name); // Notch
```

## Always guard `JSON.parse`

Parsing untrusted or stored input can throw. Wrap it:

```javascript
try {
  const data = JSON.parse(jsonString);
  applySettings(data);
} catch (e) {
  error('Failed to parse stored settings:', e.message);
}
```

Data that has been in your database since three schema versions ago counts as untrusted.

## Java objects do not serialize

This is the one PixelScript-specific gotcha. `JSON.stringify` only understands JavaScript values. A Java
object such as a `Location`, `ItemStack` or `Player` will not round-trip.

Pull out the primitives first:

```javascript
// Wrong: produces something useless
JSON.stringify({ location: player.getLocation() });

// Right: extract what you actually need
JSON.stringify({
  world: player.getWorld().getName(),
  x: player.getLocation().getX(),
  y: player.getLocation().getY(),
  z: player.getLocation().getZ(),
});
```

The same applies to Java collections. Convert an `ArrayList` or `HashMap` to a JS array or object before
serializing.

For locations specifically, [`DataFile`](./spec-013-shared-state.md) has `setLocation` and `getLocation`
helpers that handle the round-trip for you in YAML.

Other unsupported values behave as they do everywhere else in JavaScript: functions, `undefined` and
symbols are dropped, and circular references throw. Convert `Date` objects with `toISOString()`.

## Storing JSON in the database

A JSON blob in a `TEXT` column is a perfectly reasonable way to store a bag of settings that you never
query by field:

```javascript
function savePlayerSettings(uuid, settings) {
  Scheduler.runAsync(() => {
    Sql.makeUpdate(
      'UPDATE player_data SET settings = ? WHERE uuid = ?',
      (stmt) => {
        stmt.setString(1, JSON.stringify(settings));
        stmt.setString(2, uuid);
      },
      (rows) => { if (rows === -1) error('Failed to save settings for', uuid); }
    );
  });
}

function loadPlayerSettings(uuid, callback) {
  Scheduler.runAsync(() => {
    Sql.makeQuery(
      'SELECT settings FROM player_data WHERE uuid = ?',
      (stmt) => stmt.setString(1, uuid),
      (rs) => {
        let settings = null;

        if (rs !== null && rs.next()) {
          try {
            settings = JSON.parse(rs.getString('settings'));
          } catch (e) {
            error('Corrupt settings blob for', uuid, e.message);
          }
        }

        Scheduler.runSync(() => callback(settings));
      }
    );
  });
}
```

If you find yourself wanting to filter or sort on a value inside the blob, promote it to a real column with
[`Sql.addColumnIfNotExists`](./spec-009-database.md).

## Deep cloning

The usual trick works, with the usual caveat that it only survives JSON-representable values:

```javascript
const clone = JSON.parse(JSON.stringify(original));
```

## Related

- [Database](./spec-009-database.md) for persistence
- [HTTP fetch](./spec-010-fetch.md), whose `response.asObject()` parses JSON for you
- [Shared state](./spec-013-shared-state.md) for YAML config files, which are usually nicer for anything a human edits
