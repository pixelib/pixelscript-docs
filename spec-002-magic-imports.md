---
project: pixelscript
slug: magic-imports
title: Magic imports ($)
description: The $ global, a shorthand resolver for Java classes in configured packages.
status: stable
tags: [api, classloading, import]
updated: 2026-07-28
parent: API reference
nav_order: 2
permalink: /magic-imports/
---
# `$` magic imports

You are a developer, and therefore lazy in one way or another. You will be reaching for Java classes
constantly, whether that is the Bukkit `Material` enum or your own plugin's `UserManager`, and typing
`Script.loadClass('org.bukkit.Material')` every time gets old.

The `$` global resolves a class by its **simple name** by searching a configured list of packages.

```javascript
const buildableBlocks = [
  $.Material.OAK_PLANKS,
  $.Material.STONE_BRICKS,
  $.Material.GLASS_PANE,
];

if (!(sender instanceof $.Player)) {
  sender.sendRichMessage('<red>This command can only be run by a player.</red>');
  return;
}

registerListener($.PlayerJoinEvent, (event) => { /* ... */ });
```

## Configuring the search path

`class-searchcontext.yml` in `plugins/PixelScript` lists the packages `$` searches:

```yaml
search-packages:
  - 'org.bukkit'
  - 'com.destroystokyo.paper'
  - 'org.spigotmc'
```

Add your own framework or plugin package here and `$.UserManager` starts working. See
[Configuration](./tutorial-002-configuration.md).

## Performance

The first lookup of a given name walks the search path, which costs real time proportional to how many
packages you have listed. Every lookup after that is a cached map hit.

That is fast, but it is never as fast as a plain `const` holding the class. So:

- **Use `$`** for enums, event classes, and one-off `instanceof` checks in code paths that can spare a map lookup.
- **Use `Script.loadClass`** in hot loops, and for anything outside your search path.

Reaching for `$` inside a per-tick timer that runs for every online player is the one case where this
actually shows up in a profile.

## The recommended alternative

For anything you use across many files, the cleanest option is a small module that exports the classes once:

```javascript
// utils/bukkit-types.js
export const Material = Script.loadClass('org.bukkit.Material');
export const ItemStack = Script.loadClass('org.bukkit.inventory.ItemStack');
export const Player = Script.loadClass('org.bukkit.entity.Player');
```

```javascript
import { Material, ItemStack } from '@/utils/bukkit-types';
```

You get one resolution for the whole runtime, real import statements your IDE can follow, and a single
place to change when a class moves. See [import and export](./spec-003-import-export.md).

## Failure behaviour

If `$` cannot find a class it logs `Class not found: <name>` and returns `null`. It does not throw, so a
typo shows up as a confusing `TypeError` further down rather than at the point of the mistake. When
something is mysteriously undefined, check the console first.
