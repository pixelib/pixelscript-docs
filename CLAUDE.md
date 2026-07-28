# PixelScript

A hot-reloadable JavaScript/TypeScript runtime for Minecraft servers, running on Texel (a JVM JS engine
forked from Nashorn). Scripts call the real Bukkit/Paper API directly and are reloaded on save without
restarting the server.

This file is a condensed reference for writing PixelScript scripts. It is the distilled version of the
full documentation, which is:

- in `.pixelscript-docs/` if this project was set up with the one-command installer, **read it when you need detail**
- otherwise at <https://pixelib.github.io/pixelscript-docs/>

**Before writing anything**, read `PixelScript/scripts/` if this server still has the example project the
plugin ships on first boot. It is a complete, working codebase and the fastest way to absorb the idioms.

Everything below describes the runtime, not this specific project. Project-specific conventions belong in a
"This project" section at the bottom of this file.

---

## Global API surface

Everything below is available in every script with no import. This is the complete list.

### Core

| Global | Signature / type |
|--------|------------------|
| `Script` | `loadClass(fqn)`, `addUnloadCallback(fn)`, `getCaller()`, `loadLibrary(relPath)`, `loadGradleDependency(coord)`, `measure(name, fn)` |
| `Bukkit` | The live `org.bukkit.Server` |
| `__plugin` | The PixelScript `JavaPlugin`, for APIs that demand a `Plugin` |
| `$` | Resolves a Java class by simple name from `class-searchcontext.yml` packages |
| `Watcher` | `watch(path)` / `watch(path[])` |

### Registration (auto-unregistered on reload)

| Global | Signature |
|--------|-----------|
| `registerCommand` | `(name, (sender, args) => void)` |
| `registerTabCompleter` | `(name, (sender, args) => string[])` |
| `registerCommandAlias` | `(name, string \| string[])` |
| `registerListener` | `(eventClass, (event) => void, priority?)` |
| `registerImplementation` | `(javaInterfaceFqn, implObject)` |
| `getImplementation` | `(javaInterfaceFqn) => proxy` |

### Scheduler

`Scheduler.runSync(fn)`, `runAsync(fn)`, `runSyncLater(fn, ticks)`, `runAsyncLater(fn, ticks)`,
`runSyncTimer(fn, delay, period)`, `runAsyncTimer(fn, delay, period)`, `cancel(taskId)`.

All return an `int` task id. Bukkit-style aliases exist (`runTask`, `runTaskTimerAsynchronously`, ...).

### Data

| Global | Notes |
|--------|-------|
| `Sql` | `isAvailable()`, `makeQuery(q, prep, handler)`, `makeUpdate(q, prep, handler)`, `transaction(ctx => ...)`, `createTableIfNotExists(...)`, `addColumnIfNotExists(...)`. **All blocking** |
| `DataFile` | `new DataFile('scripts/resources/x.yml')`, path relative to `plugins/PixelScript` |
| `StaticStorage` | `set(k,v)`, `get(k)`, `getOrFillDefault(k, default)`. Survives reloads |
| `GlobalMap` | `GlobalMap.getInstance(name)` → `put`/`set`/`get`/`remove`. Survives reloads |
| `GlobalNotification` | `getInstance(channel)` → `on`/`subscribe`/`registerListener`, `send`/`publish`/`notify` |

### Utility

| Global | Notes |
|--------|-------|
| `fetch(url, options?)` | **Blocking.** Options: `method`, `headers`, `postBody`. Returns `{ asString(), asObject(), getStatusCode(), isSuccessful(), isFailed() }` |
| `log` / `warn` / `error` | Also as `console.log/warn/error` |
| `newLocation(world, x, y, z, pitch?, yaw?)` | Note: pitch before yaw |
| `asInt/asLong/asFloat/asDouble/asShort/asByte` | Disambiguate Java numeric overloads |
| `ArrayList`, `HashMap`, `HashSet`, `UUID`, `Gson`, `Arrays`, `List` | Preloaded Java classes |
| `Java.extend(AbstractClass, overrides)` | Subclass a Java abstract class |
| `Java.synchronized(fn, lockObject)` | Java `synchronized` semantics |
| `JSON`, `Map`, `Set`, `Math`, `Date` | Standard JS |

`Java.type()` still works but is **deprecated** and logs on every call. Use `Script.loadClass`.

---

## The load tree (get this right first)

Every script is in exactly one of three roles, and the role determines reload behaviour.

**Root**: any `.js`/`.ts` directly in `scripts/`. Auto-loaded on boot in **undefined order**. Reserve for
the entrypoint and emergency hotfixes.

**Watched**: loaded via `Watcher.watch()` from a parent. **Reload barrier**: reloading the parent does
*not* reload it. Editing it reloads it and its children. Removing the watch line unloads it and everything
under it.

**Imported**: loaded via `import`. **Single shared instance** across all importers, so exported objects
are true singletons. Editing it **cascades**: it reloads, then every importer, then their children.

> Name a file `x.isolate.js` to get a fresh instance per importer instead of a shared one. Rare.

### The rule

- **Features get watched.** They register commands/listeners and export nothing.
- **Utilities get imported.** They export functions/singletons and register nothing.

Mixing these is what produces surprise reload cascades.

### `Watcher.watch()` gotchas

1. **Paths are relative to the calling script's own directory**, not the `scripts` root.
2. **Must be called during initial top-level evaluation.** The watch list commits once, right after the
   script finishes evaluating. Calls from inside a listener/callback/command are silently ignored.

```javascript
// scripts/global/init-global.js
Watcher.watch([
  'feature/chatformat/chatformat.js',   // → scripts/global/feature/chatformat/chatformat.js
  'feature/punishments/init.js',
]);
```

### Canonical entrypoint

```javascript
// scripts/init.js  (root)
const System = Script.loadClass('java.lang.System');
const serverType = System.getenv('SERVER_TYPE') || 'lobby';

Watcher.watch('global/init-global.js');   // common to all server types

switch (serverType) {
  case 'lobby': Watcher.watch('servers/lobby/init-lobby.js'); break;
  case 'game':  Watcher.watch('servers/game/init-game.js');  break;
  default:
    console.warn(`Unknown SERVER_TYPE "${serverType}".`);
    Bukkit.shutdown();
    break;
}
```

Each `init-*.js` is nothing but a list of watches. That gives a tree of barriers where a change deep in one
feature never disturbs the rest.

### Layout

```text
scripts/
├── init.ts                      # root, branches on server type
├── config/constants.js          # exported constants
├── global/
│   ├── init-global.js           # watches everything below
│   ├── feature/<name>/          # watched features
│   └── utils/                   # imported modules
└── servers/<type>/
    ├── init-<type>.js
    └── feature/<name>/
```

---

## Module imports

Named exports only. **No default exports.** No `import * as ns`. No `import { a as b }`.

| Form | Resolves from |
|------|---------------|
| `@/utils/messages` | scripts root **(preferred)** |
| `/utils/messages` | scripts root |
| `./economy`, `../chat/messages` | current script's directory |
| `sys/shop` | current script's directory |

- **`.js`/`.ts` extension is optional**. The resolver tries `.js`, then `.ts`.
- **Always write `@/` with the slash.** Bare `@utils/x` does not resolve correctly.
- `tsconfig.json` maps `@/*` to the scripts root, so the IDE agrees with the runtime.
- Circular imports are detected and rejected at load with a readable trace.

Prefer `@/` over `../../../../`. Both work; only one is readable.

---

## Threading (the source of most real bugs)

**Script evaluation is always synchronous on the main thread**, on boot and on every reload. Deliberate: it
makes registration safe and makes reloads atomic from a gameplay perspective (no window where a protection
listener is half-installed). Cost: keep top-level work cheap.

After that, functions run wherever they are invoked:

| Trigger | Thread |
|---------|--------|
| Command executor | main |
| Event listener | whatever fires it (usually main; `AsyncChatEvent` is not) |
| `Scheduler.runSync*` | main |
| `Scheduler.runAsync*` | background pool |
| Unload callbacks | main |

**Never touch game state from an async thread.** Players, worlds, entities, blocks, inventories,
scoreboards are main-thread only.

**`Sql` and `fetch` are blocking, and their callbacks run inline on the calling thread.** Always inside
`runAsync`.

The pattern, in full:

```javascript
registerCommand('balance', (sender) => {
  const uuid = sender.getUniqueId().toString();     // read on main thread FIRST

  Scheduler.runAsync(() => {
    const balance = queryBalance(uuid);              // slow work off-thread

    Scheduler.runSync(() => {
      sender.sendRichMessage(`<green>Balance: ${balance}`);   // apply on main
    });
  });
});
```

Passing a `Player` into an async task and calling methods on it there is the most common version of this
bug. Extract primitives before the hop.

For shared mutable state across threads: `Java.synchronized(fn, lockObject)`, or use the
`ConcurrentHashMap`-backed `GlobalMap`/`StaticStorage`.

---

## Cleanup contract

**Automatic** (never write teardown for these):
- Commands, tab completers, aliases
- Event listeners registered via `registerListener`
- Scheduler tasks
- `GlobalNotification` listeners
- Jars from `loadLibrary` (once all users unload)

**Yours** via `Script.addUnloadCallback(fn)`:
- Boss bars, scoreboards, teams
- Spawned entities, holograms, displays
- Open inventories
- Anything registered directly against a third-party Java API (ProtocolLib listeners, etc.)
- Anything registered against `__plugin` directly

```javascript
const bossBar = Bukkit.createBossBar('Welcome!', $.BarColor.BLUE, $.BarStyle.SEGMENTED_6);

registerListener($.PlayerJoinEvent, (e) => bossBar.addPlayer(e.getPlayer()));
registerListener($.PlayerQuitEvent, (e) => bossBar.removePlayer(e.getPlayer()));

Script.addUnloadCallback(() => {
  bossBar.removeAll();
  bossBar.setVisible(false);
});

// Reloading mid-session must not leave current players out
Bukkit.getOnlinePlayers().forEach(p => bossBar.addPlayer(p));
```

That last line is a required habit. A script that only reacts to join events looks broken after every
reload with players online.

For utilities that create resources on a caller's behalf, tie cleanup to the *caller*:

```javascript
// utils/entities.js
export function makeHologram(location, text) {
  const entity = spawnDisplay(location, text);
  Script.getCaller().addUnloadCallback(() => entity.remove());
  return entity;
}
```

`getCaller()` throws if the caller has multiple instances (`.isolate.js`). Pass `Script` explicitly then.

---

## Java interop gotchas

**Java collections are not JS collections.** `Bukkit.getOnlinePlayers()` has `size()`, not `.length`, and
no `map`/`filter`. `forEach` is bridged.

```javascript
const names = [];
Bukkit.getOnlinePlayers().forEach(p => names.push(p.getName()));   // now a real JS array
```

**Java `Map#forEach` is `(key, value)`**, the opposite of JS `Map#forEach` `(value, key)`.

**Numeric map keys box badly.** Use a JS `Map` for internal state, or normalise with `Math.trunc()`. Use
Java collections only when Java receives them.

**Ambiguous numeric overloads** (e.g. `World#spawnParticle`) need explicit casts:

```javascript
world.spawnParticle(Particle.FLAME, asDouble(x), asDouble(y), asDouble(z), asInt(count));
```

**Raw `Class` objects** come from `.class`:

```javascript
const ArchModule = Script.loadClass('dev.pixelib.pp.core.arch.ArchModule').class;
```

**Functional interfaces take plain functions.** Multi-method interfaces take an object literal via the
constructor:

```javascript
const impl = new SomeInterface({ methodA() {}, methodB(x) { return x; } });
```

**Abstract classes need `Java.extend`:**

```javascript
const MyAdapter = Java.extend(PacketAdapter, {
  onPacketReceiving: function (event) { /* ... */ }
});
const listener = new MyAdapter(__plugin, ListenerPriority.MONITOR, [PacketType.Play.Client.ARM_ANIMATION]);
```

**`super` reaches parent methods and constructors, not parent fields.**

---

## Exposing JS to Java (implementation API)

When a Java plugin needs to call into script logic. Type-safe on the Java side and **never stale across
reloads**.

```java
// 1. Contract, in the Java plugin
public interface ItemRegistry {
    boolean is(ItemStack stack, String id);
}
```

```javascript
// 2. Implement + register in JS (validated at REGISTRATION time; script fails to load on mismatch)
class JavaItemRegistry {
  is(itemStack, id) {
    return !!itemStack && !!id && DcItem.is(itemStack, id);
  }
}
registerImplementation('dev.pixelib.pp.scripts.api.ItemRegistry', new JavaItemRegistry());
```

```java
// 3. Use in Java. Fetch once, hold forever. The proxy re-points itself on reload.
ItemRegistry registry = JS.getImplementation(ItemRegistry.class);
```

- Nested interfaces use `$`: `'com.example.Outer$MyApi'`
- `JS.getImplementation` never returns null; before any script registers it returns a placeholder that throws on call
- `JS` lives in `dev.pixelib.pixelscript:API` on `https://repo.pixelib.dev/artifacts/pixelib-public`

Do **not** use this for one-off callbacks. Pass an object literal instead.

---

## Idioms worth copying

### Command middleware

Executors are just functions, so wrap them. This is the standard way to share guards.

```javascript
export function makeAdminCommand(permission, handler, options = { mustBePlayer: true }) {
  return (sender, args) => {
    if (options.mustBePlayer && !(sender instanceof $.Player)) {
      sender.sendRichMessage('<red>Players only.</red>');
      return;
    }
    if (!sender.hasPermission(permission)) {
      sender.sendRichMessage('<red>No permission.</red>');
      return;
    }
    handler(sender, args);
  };
}

registerCommand('gamemode', makeAdminCommand('command.gamemode', (sender, args) => { /* ... */ }));
```

Composes: `cooldownMiddleware(5, makeAdminCommand('perm', handler))`.

### Messages module (MiniMessage)

Centralise formatting once and import it everywhere.

```javascript
// patches/chat/messages.js
const MiniMessage = Script.loadClass('net.kyori.adventure.text.minimessage.MiniMessage');
const Component = Script.loadClass('net.kyori.adventure.text.Component');
const MM = MiniMessage.miniMessage();
const PREFIX = MM.deserialize('<gray>[<bold><#B65ED3>D<#497af7>C</bold>]</gray> ');

export function mm(message) { return MM.deserialize(message); }

export function dcError(message) {
  if (message instanceof Component) return PREFIX.append(message);
  return PREFIX.append(mm(`<#E74C3C><italic>${message}</italic>`));
}
```

Prefer `player.sendRichMessage('<green>...')` (MiniMessage) over legacy `§` codes on Paper.

### Class constants at module top

```javascript
const ChatEvent = Script.loadClass('io.papermc.paper.event.player.AsyncChatEvent');
const GameMode = $.GameMode;   // hoist $ lookups out of hot paths
```

### Data access module

Wrap `Sql` so the async dance and the SQL live in one place, and everything else gets a callback API.

```javascript
class SessionManager {
  constructor() {
    if (!Sql.isAvailable()) throw new Error('Sql is not available');   // fail loudly at load
    Sql.createTableIfNotExists('player_sessions', `CREATE TABLE ...`, (created) => {
      if (created) log('Created player_sessions table');
    });
  }

  getTotalPlaytime(uuid, callback) {
    Scheduler.runAsync(() => {
      Sql.makeQuery(
        'SELECT SUM(session_duration) AS total FROM player_sessions WHERE player_uuid = ?',
        (stmt) => stmt.setString(1, uuid),
        (rs) => callback(rs !== null && rs.next() ? rs.getInt('total') : 0)
      );
    });
  }
}

export const sessions = new SessionManager();   // singleton via module semantics
```

### Load config once into memory

```javascript
const WARP_FILE = new DataFile('scripts/resources/warps.yml');
const WARP_CACHE = new HashMap();

const ids = WARP_FILE.getKeysInSection('warps');   // Java String[], use an indexed loop
for (let i = 0; i < ids.length; i++) {
  WARP_CACHE.put(ids[i], {
    id: ids[i],
    name: WARP_FILE.getString(`warps.${ids[i]}.name`),
    location: WARP_FILE.getLocation(`warps.${ids[i]}`),
  });
}
```

Re-read happens naturally on reload, so editing the YAML plus touching the script picks up changes.

### Runtime Maven dependencies

```javascript
Script.loadGradleDependency('com.saicone:rtag:1.5.16');
Script.loadGradleDependency('com.saicone:rtag-item:1.5.16');
const RtagItem = Script.loadClass('com.saicone.rtag.RtagItem');
```

Cached in `plugins/PixelScript/maven_cache`. **Transitive dependencies are not resolved**, so list every
artifact.

### GUIs

There is no GUI framework. Build one on `InventoryHolderAttachment`:

```javascript
const AttachmentHolder = Script.loadClass('dev.pixelib.pixelscript.script.api.InventoryHolderAttachment');

const holder = new AttachmentHolder();
const inv = Bukkit.createInventory(holder, 54, MM.deserialize('<dark_gray>Menu'));
holder.setInventory(inv);
holder.setAttachment('__internalClickHandler', (player, slot, clickType, event) => { /* ... */ });
```

A single listener script then checks `inventory.getHolder() instanceof AttachmentHolder`, cancels the
event, and dispatches to the attachment. Working implementation:
`PixelScript/Plugins/pixelscript-paper/src/main/resources/scripts/global/utils/guilib/`.

Always close open GUIs in an unload callback, or reloading strands players in a dead inventory.

---

## TypeScript

`.ts` files are compiled at load and behave identically to `.js` afterwards. Mix freely; extensions are
optional in imports.

- Compiled with `target: ESNEXT`, `module: ESNEXT`, missing imports/names tolerated
- **Type errors go to the console (`[TSC]` prefix) and do NOT block loading.** Run `tsc --noEmit` if you want them to.
- No source maps: runtime stack traces point at compiled JS
- `.d.ts` files are excluded from script discovery

`scripts/definitions.d.ts` is **generated**. Never edit, never commit, always gitignore. It contains every
global plus every Java class your scripts have touched, growing as you use more. New classes appear in
autocomplete after the next reload.

---

## Debugging

| Command | Use |
|---------|-----|
| `/script` | Version, script count, boot time. First check after a deploy. |
| `/script tree` | Full dependency hierarchy. **The** command for "why did editing X reload Y?" |
| `/script info <path>` | Parents, extensions, load time, profiler data (avg/min/max) |
| `/script list` | Every loaded script path |
| `/script timings <enable\|disable>` | Profiler collection, on by default |

`Script.measure('name', fn)` adds your own entry to `/script info`.

The runtime already logs, with a script stack trace pointing at your source line: uncaught errors in
commands/listeners/tasks, SQL errors, fetch errors, unload callback failures, and load failures. You rarely
need try/catch just to see a failure.

Common performance culprits, in order: per-tick timers doing per-player work; blocking `Sql`/`fetch` on the
main thread; `$` lookups in hot loops; oversized reload cascades.

---

## Writing style for scripts

- 2-space indent, semicolons, single quotes, `const` by default
- Feature scripts register at top level and export nothing; utility modules export and register nothing
- Hoist `Script.loadClass` / `$` results to module-level `const`
- Use `@/` imports for anything outside the current directory
- MiniMessage (`sendRichMessage`) over legacy `§` codes
- JSDoc on exported functions; the IDE uses it
- Throw at the top level when a hard dependency is missing (`if (!Sql.isAvailable()) throw ...`) so the failure shows in `/script list`

---

## Where to read more

When this file is not enough, the full articles are in `.pixelscript-docs/` (or at
<https://pixelib.github.io/pixelscript-docs/>). The ones worth opening:

| Topic | Article |
|-------|---------|
| Load tree, watchers, project layout | `tutorial-003-scripts.md` |
| Module resolution and reload cascades | `spec-003-import-export.md` |
| Threading rules in full | `spec-008-threading-model.md` |
| Scheduler, with worked patterns | `spec-007-scheduler.md` |
| Sql queries and transactions | `spec-009-database.md` |
| Exposing JS to Java plugins | `spec-014-java-implementations.md` |
| Interop gotchas | `tips-001-java-interop.md` |

---

## This project

<!--
  Replace this section with conventions specific to this codebase. Useful things to record:
  server types and the env var that selects them, the entrypoint file, the message/formatting
  module everything routes through, the permission namespace, the Paper version targeted,
  and any house rules that are not obvious from reading a single file.
-->

_Not filled in yet._
