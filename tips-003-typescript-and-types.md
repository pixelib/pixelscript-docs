---
project: pixelscript
slug: type-generation
title: TypeScript and type generation
description: How definitions.d.ts is generated, and how to write .ts scripts.
status: beta
tags: [typescript, types, ide, tooling]
updated: 2026-07-28
parent: Tips & tricks
nav_order: 3
permalink: /type-generation/
---
# TypeScript and type generation

PixelScript writes TypeScript definitions for the Java classes your scripts actually use, and it can run
`.ts` files directly. Together these give you real autocomplete and real type checking against the Bukkit
API.

## `definitions.d.ts`

The runtime generates `scripts/definitions.d.ts` on boot, and again whenever your scripts reach for
something new.

What ends up in it:

- Every global the platform binds: `Bukkit`, `Script`, `Watcher`, `Scheduler`, `Sql`, `DataFile`, `registerCommand`, `registerListener`, `fetch`, and the rest
- Every class you have loaded with [`Script.loadClass`](./spec-001-script.md)
- Every class resolved through [`$`](./spec-002-magic-imports.md)
- Everything those classes reference in their signatures, so `player.getLocation()` returns a `Location` that is actually defined
- Classes from jars added with `Script.loadLibrary` or `Script.loadGradleDependency`

The file starts with `//@ts-nocheck` and declares globals inside `declare global`, with Java packages
mirrored as TypeScript namespaces:

```typescript
declare global {
    const Script: virtualclass_dev.pixelib.pixelscript.SimpleScript;
    const Bukkit: org.bukkit.Server;
    const Scheduler: dev.pixelib.pixelscript.script.extensions.SchedulerExtension$ScriptScheduler;
    function registerCommand(commandName: string, callbackWithSenderAndArgs: (sender: org.bukkit.command.CommandSender, args: string[]) => void): void;
    function registerListener<Ctor>(eventClass: Ctor, listener: (event: InstanceType<Ctor>) => void, eventPriority: EventPriority): void;
}
```

Two consequences worth knowing:

**The file grows as you use more classes.** A class only becomes a full definition once a script imports
it. This keeps the file to a manageable size instead of emitting every class on the classpath.

**Autocomplete for a brand new class appears after the next reload.** Load the class, save, let the script
reload, and your IDE picks up the regenerated file.

Never edit `definitions.d.ts` by hand and never commit it. Put it in your `.gitignore`.

## `tsconfig.json`

The generated `tsconfig.json` wires everything together:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["*"]
    },
    "allowJs": true,
    "checkJs": false
  },
  "files": ["definitions.d.ts"],
  "include": ["**/*.ts", "**/*.js"]
}
```

- `files` pulls in the generated globals
- `paths` maps [the `@/` import alias](./spec-003-import-export.md) to the scripts root, so your IDE resolves `@/utils/messages`
- `allowJs` with `checkJs: false` means your `.js` files get completion without being type checked

Turn on `checkJs` if you want type errors in plain JavaScript too. Expect noise the first time.

## Writing `.ts` scripts

Any `.ts` file in your scripts folder is compiled to JavaScript at load time and then behaves exactly like
a `.js` script. Root scripts, watched scripts and imported modules can all be TypeScript.

```typescript
// init.ts
const SystemInstance = Script.loadClass('java.lang.System');
const serverType: string = SystemInstance.getenv("DC_SERVER_TYPE") || "park";

Watcher.watch([
  'patches/init.js'
]);

switch (serverType) {
  case "park":
    Watcher.watch('servers/park/init.js');
    break;

  case "lobby":
    Watcher.watch('servers/lobby/init.js');
    break;

  default:
    log('Unknown server type: ' + serverType);
    Bukkit.shutdown();
    break;
}
```

You can mix freely. A `.ts` file can import a `.js` module and the other way around, and the extension is
optional in import paths: the resolver tries `.js` first and falls back to `.ts`.

### Compilation details

Files are compiled with `target: ESNEXT` and `module: ESNEXT`, with missing imports and names tolerated
(the generated globals are declared, not imported). Compilation is lenient by design: type errors are
reported to the console with a `[TSC]` prefix but do **not** stop your script from loading.

That is a deliberate trade-off. Hot reloading stays fast and a type complaint never takes a feature
offline. It does mean **the console is where TypeScript errors go, not the reload**. If you want type
errors to block, run `tsc --noEmit` in your editor or CI.

Source maps are not generated, so a runtime stack trace points at the compiled JavaScript. Keep that in
mind when debugging a `.ts` file.

### `.d.ts` files are never loaded as scripts

Files ending in `.d.ts` are explicitly excluded from script discovery, so `definitions.d.ts` sitting in
your scripts root is not treated as a root script.

## Practical setup

1. Open `plugins/PixelScript` (or `plugins/PixelScript/scripts`) as your IDE project root so `tsconfig.json` is picked up.
2. Add `definitions.d.ts` to `.gitignore`.
3. Work against a **local** server, so regenerated definitions land in your working copy immediately. See [Setting up your work environment](./tutorial-001-setting-up-your-work-env.md).
4. Use `@/` imports so your IDE and the runtime agree on where a module lives.

## Status

TypeScript support is newer than the rest of the runtime. The compilation path is stable, but the tooling
around it (stricter checking, source maps, editor integration) is still being built out. Plain JavaScript
remains the well-trodden path if you want zero surprises.
