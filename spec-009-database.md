---
project: pixelscript
slug: api-sql
title: Database
description: Execute SQL queries and transactions through the built-in Sql module, with connection pooling.
status: stable
tags: [api, sql, database, storage]
updated: 2026-07-28
parent: API reference
nav_order: 9
permalink: /api-sql/
---
# The `Sql` API

The `Sql` global gives your scripts a pooled JDBC connection with prepared statements, transactions and
formatted error reporting. SQLite works out of the box; MySQL and MariaDB are a config change away.

> **Every `Sql` call is blocking, and every callback runs inline on the calling thread.** Always wrap them
> in `Scheduler.runAsync()`. See [the concurrency model](./spec-008-threading-model.md).

Configuration lives in `database.yml`. See [Configuration](./tutorial-002-configuration.md).

## Checking availability

`Sql.isAvailable()` (and its alias `Sql.isConnected()`) tell you whether the pool is up. Modules that
cannot work without a database should fail loudly at load time:

```javascript
if (!Sql.isAvailable()) {
  throw new Error('Sql is not available');
}
```

Throwing at the top level marks the script as failed and shows up in `/script list`, which is exactly what
you want. Silently degrading is worse.

## Schema management

### `Sql.createTableIfNotExists(tableName, createTableSql, callback)`

Creates a table only if it does not already exist. The callback receives `true` if it was created, `false`
if it already existed or creation failed.

```javascript
Sql.createTableIfNotExists(
  'player_sessions',
  `CREATE TABLE player_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    player_uuid VARCHAR(36) NOT NULL,
    player_name VARCHAR(32) NOT NULL,
    login_time TIMESTAMP NOT NULL,
    logout_time TIMESTAMP,
    session_duration INTEGER
  )`,
  (created) => {
    if (created) console.log('Created player_sessions table');
  }
);
```

### `Sql.addColumnIfNotExists(tableName, columnName, columnDefinition, callback)`

Your migration path. The callback receives `true` if the column was added.

```javascript
Sql.addColumnIfNotExists('player_data', 'level', 'INT DEFAULT 1', (added) => {
  if (added) console.log("Added 'level' column");
});
```

Both of these detect existence by attempting a query and catching the failure, so a first run against a
fresh database logs a harmless SQL error before creating the table.

## Querying

### `Sql.makeQuery(query, preparer, resultHandler)`

`preparer` binds parameters, `resultHandler` receives a `ResultSet`. Parameter and column indices start at
**1**, not 0.

```javascript
Scheduler.runAsync(() => {
  Sql.makeQuery(
    'SELECT username, coins FROM player_data WHERE uuid = ?',
    (stmt) => stmt.setString(1, uuid),
    (rs) => {
      if (rs === null) {
        error('Query failed, check the server log');
        return;
      }

      if (rs.next()) {
        console.log(`${rs.getString('username')} has ${rs.getInt('coins')} coins`);
      }
    }
  );
});
```

`resultHandler` receives `null` when the query failed. Always check.

Collect rows into a plain JS array before leaving the handler. The `ResultSet` is closed once it returns:

```javascript
Sql.makeQuery(
  'SELECT login_time, session_duration FROM player_sessions WHERE player_uuid = ? ORDER BY login_time DESC LIMIT ?',
  (stmt) => {
    stmt.setString(1, uuid);
    stmt.setInt(2, limit);
  },
  (rs) => {
    const sessions = [];
    while (rs !== null && rs.next()) {
      sessions.push({
        loginTime: rs.getString('login_time'),
        duration: rs.getInt('session_duration'),
      });
    }
    callback(sessions);
  }
);
```

### `Sql.makeUpdate(query, preparer, resultHandler)`

For `INSERT`, `UPDATE` and `DELETE`. The handler receives the affected row count, or `-1` on error.

```javascript
Scheduler.runAsync(() => {
  Sql.makeUpdate(
    'INSERT INTO player_sessions (player_uuid, player_name, login_time) VALUES (?, ?, CURRENT_TIMESTAMP)',
    (stmt) => {
      stmt.setString(1, uuid);
      stmt.setString(2, playerName);
    },
    (rows) => {
      if (rows === -1) error('Failed to record login');
    }
  );
});
```

## Transactions

`Sql.transaction(handler)` runs a group of statements atomically. Everything commits together or rolls back
together.

The handler receives a context with three methods:

| Method | Purpose |
|--------|---------|
| `ctx.update(query, preparer)` | Run an `INSERT`/`UPDATE`/`DELETE` |
| `ctx.execute(query, preparer)` | Alias for `update` |
| `ctx.query(query, preparer, resultHandler)` | Run a `SELECT` and process results |

```javascript
Scheduler.runAsync(() => {
  Sql.transaction((ctx) => {
    ctx.query(
      'SELECT id FROM player_sessions WHERE player_uuid = ? AND logout_time IS NULL ORDER BY login_time DESC LIMIT 1',
      (stmt) => stmt.setString(1, uuid),
      (rs) => {
        if (!rs.next()) return;

        const sessionId = rs.getInt('id');

        ctx.update(
          `UPDATE player_sessions
             SET logout_time = CURRENT_TIMESTAMP,
                 session_duration = ROUND((JULIANDAY(CURRENT_TIMESTAMP) - JULIANDAY(login_time)) * 86400)
           WHERE id = ?`,
          (stmt) => stmt.setInt(1, sessionId)
        );
      }
    );
  });
});
```

Note that `ctx.query` is synchronous within the transaction, so you can read a value and immediately act on
it. That is the main reason to reach for a transaction even when you are not strictly writing multiple rows.

## Parameter and column types

```javascript
// Binding
stmt.setString(1, 'text');
stmt.setInt(1, 42);
stmt.setLong(1, 1234567890);
stmt.setDouble(1, 3.14);
stmt.setBoolean(1, true);
stmt.setTimestamp(1, timestamp);
stmt.setNull(1, java.sql.Types.VARCHAR);

// Reading
rs.getString('column_name');
rs.getInt('column_name');
rs.getLong('column_name');
rs.getDouble('column_name');
rs.getBoolean('column_name');
rs.getTimestamp('column_name');
rs.getDate('column_name');
```

Columns can be read by index as well, also 1-based.

## Worked example: a data access module

Wrapping the raw API in a module keeps the async dance and the SQL in one place, and gives the rest of your
codebase a clean callback interface.

```javascript
// lib/session-manager.js
class SessionManager {
  constructor() {
    if (!Sql.isAvailable()) throw new Error('Sql is not available');

    Sql.createTableIfNotExists('player_sessions', `CREATE TABLE player_sessions (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      player_uuid VARCHAR(36) NOT NULL,
      player_name VARCHAR(32) NOT NULL,
      login_time TIMESTAMP NOT NULL,
      logout_time TIMESTAMP,
      session_duration INTEGER
    )`, (created) => {
      if (created) console.log('Created player_sessions table');
    });
  }

  recordLogin(uuid, playerName, callback) {
    Scheduler.runAsync(() => {
      Sql.makeUpdate(
        'INSERT INTO player_sessions (player_uuid, player_name, login_time) VALUES (?, ?, CURRENT_TIMESTAMP)',
        (stmt) => {
          stmt.setString(1, uuid);
          stmt.setString(2, playerName);
        },
        (rows) => { if (callback) callback(rows > 0); }
      );
    });
  }

  getTotalPlaytime(uuid, callback) {
    Scheduler.runAsync(() => {
      Sql.makeQuery(
        'SELECT SUM(session_duration) AS total FROM player_sessions WHERE player_uuid = ? AND session_duration IS NOT NULL',
        (stmt) => stmt.setString(1, uuid),
        (rs) => callback(rs !== null && rs.next() ? rs.getInt('total') : 0)
      );
    });
  }
}

export const sessions = new SessionManager();
```

```javascript
// playtime.js
import { sessions } from './lib/session-manager';

registerListener($.PlayerJoinEvent, (event) => {
  const player = event.getPlayer();
  sessions.recordLogin(player.getUniqueId().toString(), player.getName());
});
```

Because the module is imported, `sessions` is a single shared instance across every script that imports it.
See [modules](./spec-003-import-export.md).

## Best practices

**Do**

- Wrap every call in `Scheduler.runAsync()`
- Bind parameters with `?`, never string-concatenate user input
- Check for `null` result sets and `-1` row counts
- Use transactions when operations must succeed or fail together
- Guard the module with `Sql.isAvailable()` at load time

**Do not**

- Concatenate user input into SQL, that is an injection
- Run queries on the main thread, that freezes the server
- Hold on to a `ResultSet` past its handler
- Assume a query succeeded without checking

## Database-specific SQL

The API is JDBC, but SQL dialects differ. The common cases:

**SQLite**
```sql
id INTEGER PRIMARY KEY AUTOINCREMENT
INSERT OR IGNORE INTO tbl (col) VALUES (?)
INSERT OR REPLACE INTO tbl (col) VALUES (?)
```

**MySQL / MariaDB**
```sql
id INT PRIMARY KEY AUTO_INCREMENT
INSERT IGNORE INTO tbl (col) VALUES (?)
INSERT INTO tbl (col) VALUES (?) ON DUPLICATE KEY UPDATE col = ?
```

**PostgreSQL**
```sql
id SERIAL PRIMARY KEY
INSERT INTO tbl (col) VALUES (?) ON CONFLICT (col) DO NOTHING
INSERT INTO tbl (col) VALUES (?) ON CONFLICT (col) DO UPDATE SET col = ?
```

If your scripts have to run against more than one dialect, keep the SQL in a single module per feature so
there is exactly one place to adjust.
