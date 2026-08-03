---
project: pixelscript
slug: changelog
title: Changelog
description: Stay updated with the latest changes and improvements in PixelScript through our detailed changelog.
status: beta
tags: [performance]
updated: 2026-07-28
nav_order: 8
permalink: /changelog/
---
### 4 August 2026 - Version 40
- Merged new template repository that gets used for default projects. The new example project is based on real organization and structure from ImagineFun and DroomCraft
- Cleaned up logging
- Fixed a rare filewatcher bug where it would watch script during initalization of the example workspace, leading to weird race conditions and errors
- Updated claude.md instructions that reflect better practices and organization

### 31 July 2026 - Version 39
- Migrated TS pre-processor to the new architecture
- Added support for `musl` (alpine linux), alongside glibc, which allows PixelScript to run on more platforms. (needed because `itzg/minecraft-server` is now alpine-based)

### 28 July 2026 - Version 38
- Fixed the `/script` command not registering on Paper servers.
- The `/script info <path>` command now shows more profiling information.
- Fixed a compiler bug related to callsite generation in ES6 classes.
- Revised dependency resolution stratigy in cases where one or more child scripts failed to load, but got resurected later.
- Implemented a ton of missing standard libraries for Strings and Arrays.

### 27 July 2026 - Version 37
- Overhauled the TypeScript toolchain: the old bundled compiler is gone, `tscjni` is now pulled from the repo instead.
- Fixed `@/` imports not resolving in every case.
- The loader no longer tries to load candidates it isn't allowed to (`.d.ts` files and friends), which cleans up boot noise.
- Added a configurable startup timeout for servers that boot terribly slowly.
- The engine is now consumed from the Maven repo instead of being vendored into the build.

---

### 10 July 2026 - Version 36
- Internal housekeeping.

---

### 7 July 2026 - Version 35
- Added `Script.loadGradleDependency(coord)` to resolve Maven/Gradle artifacts at runtime, with type generation for whatever gets loaded. Note that transitive dependencies are not resolved.
- TLS is now warmed up on boot, so the first `fetch` isn't slower than the rest.
- Broader type inclusion in the generated definitions.
- Cleaned up logging.

---

### 4 July 2026 - Version 34
- Improved error logging when a script fails on its initial load.
- Fixed a class loader boundary issue with `Script.loadClass`.
- Added support for forcing types in the generated definitions, which fixes the hinting on `GlobalNotification`.

---

### 2 July 2026 - Version 33
- Internal housekeeping.

---

### 2 July 2026 - Version 32
The first release since December, so it carries a lot. (mostly because a lot of work has been done on the engine, which is now a separate artifact, and the plugin has been updated to use it.)

s- Added the Java implementation registry: `registerImplementation` / `getImplementation` in scripts, and `JS.getImplementation` on the Java side through the new public `dev.pixelib.pixelscript:API` artifact. Implementations are validated at registration, and fall back to a throwing placeholder when the script that provided them unloads.
- Rewrote type generation: it moved into the engine, gained a searchable class generator with known types, much better type resolution, types for the database modules, typed command handlers, a `typeof` option, and qualified types for extensions.
- Decoupled the import resolver and the database layer from the platform.
- Experimental Hytale platform support.
- Fixed running on Java 25.
- Added missing `$` bindings, including enum fields.
- Fixed the default chat format in the example scripts.

It also carries six months of engine work:

- Implemented object destructuring and merging.
- Added nullish coalescing (`??`) and optional chaining (`?.`), and fixed the line numbers in errors caused by optional chaining.
- Added `Math.toRadians`.
- Allowed ES2020 trailing commas in method calls (thanks to Gongo's feedback).
- Fixed a bug (reported by Gongo) where objects from another context could not be used in a rest operation, plus a follow-up runtime bug he reported.
- Dropped the legacy call resolver.
- Updated the engine to Java 17.

---

### 20 December 2025 - Version 31
- Added support for destructuring in `for` loops.
- Removed the bundled library plugins.
- Reworked artifact authentication and set up the publishing workflow.

---

### 20 December 2025 - Version 30
- Added `/script tree`, which prints the full script dependency hierarchy.
- Added protection against circular imports, which are now rejected at load with a readable trace.
- Reworked how script loads are counted and timed, and fixed a leak in the timing data.
- Fixed parent scripts not being resurrected after one of their children failed to load.
- Added `$` type generation, so classes resolved through `$` show up in autocomplete.
- The plugin now ships a complete example script project on first boot.
- The data folder is created on start.
- Improved import and linker error messages.
- Engine: unassigned globals can now be read before entering scope.

---

### 17 December 2025 - Version 29
- Added `Math.trunc(value)` to truncate decimal values.
- Improved error logging during script unload handlers.
- Simplified error logging to reduce compiler verbosity on licenced builds.

---

### 14 December 2025 - Version 28
- Added support for BigInt math operations, and dashed numeric literals (e.g., `1_000_000`).
- Implemented private methods on JS classes using the `#` syntax.
- Implemented static inheritance for JS classes.

---

### 3 December 2025 - Version 27
- Added support for `get` and `set` accessors in JS classes.
- Implemented "complex" destructuring assignments for arrays and objects, with support for rests and computed property names.

---

### 27 November 2025 - Version 26
- Added support for array#findIndex

---

### 7 November 2025 - Version 25
- Updated to java 21
- Fixed `dequeue` type on JS array like structures.

---

### 6 November 2025 - Version 24
- Implemented fields for JS classes based on the new ecma standard (and private fields, ofc).
- Added runtime support for guarded invocations to obscure access to sensitive private fields/methods.

---

### ??
- start of public changelog, taking it out of Cam's DM, may update entries some day going back to February 2025 -
