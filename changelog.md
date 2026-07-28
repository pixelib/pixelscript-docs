---
project: pixelscript
slug: changelog
title: Changelog
description: Stay updated with the latest changes and improvements in PixelScript through our detailed changelog.
status: beta
tags: [performance]
updated: 2025-12-16
nav_order: 8
permalink: /changelog/
---
### 17 December 2025 - Version  29
- Added `Math.trunc(value)` to truncate decimal values.
- Improved error logging during script unload handlers.
- Simplified error logging to reduce compiler verbosity on licenced builds.
------

### 14 December 2025 - Version 28
- Added support for BigInt math operations, and dashed numeric literals (e.g., `1_000_000`).
- Implemented private methods on JS classes using the `#` syntax.
- Implemented static inheritance for JS classes.
------

### 3 December 2025 - Version 27
- Added support for `get` and `set` accessors in JS classes.
- Implemented "complex" destructuring assignments for arrays and objects, with support for rests and computed property names.
------

### 27 November 2025 - Version 26
- Added support for array#findIndex
------

### 7 November 2025 - Version 25
- Updated to java 21
- Fixed `dequeue` type on JS array like structures.
------

### 6 November 2025 - Version 24
- Implemented fields for JS classes based on the new ecma standard (and private fields, ofc).
- Added runtime support for guarded invocations to obscure access to sensitive private fields/methods.
------

# - start of public changelog, taking it out of Cam's DM, may update entries some day going back to February 2025 -
