---
project: pixelscript
slug: overview
title: PixelScript overview
description: What PixelScript is, how it fits into your Minecraft plugin workflow, and what to expect from the runtime.
status: stable
tags: [intro, runtime, minecraft]
updated: 2026-07-28
parent: Getting started
nav_order: 1
permalink: /overview/
---
![thumb](./assets/img/thumb.png)

# PixelScript overview

PixelScript is a modern JavaScript and TypeScript runtime for Minecraft servers. It keeps the important
fundamentals of Paper/Spigot plugin development, but lets you author your logic in hot-reloadable
JavaScript. The goal is simple: iterate fast without giving up type safety or observability.

You are not learning a new scripting language. You are writing Java with JavaScript syntax, and getting
your changes live the moment you hit save.

## Why PixelScript exists

- **No rebuild loop.** Plugin authoring usually means rebuild, restart, reconnect, repeat. PixelScript removes the restart.
- **Flexible releases.** Ship fixes and features without a full build and deploy cycle. Hide a feature behind a watcher, then flip it on when you are ready.
- **Fix bugs live.** Found a bug in production? Patch it now instead of making players wait for the next release.
- **Interop first.** Call into Java classes, reuse existing services, and hand implementations back to Java plugins.
- **AI friendly.** It is JavaScript. Every model on the planet already speaks it.

## Core ideas

1. **Live reload.** Edits to scripts are detected and applied without restarting the server.
2. **Guard rails.** The runtime validates exports, enforces a lifecycle, and surfaces errors in a readable way.
3. **Bridge API.** The runtime (an engine named *Texel*) exposes Minecraft server hooks and utility helpers as globals.
4. **Generated types.** Every Java class your scripts touch lands in a generated `definitions.d.ts`, so your IDE autocompletes against the real Bukkit API.

## What the documentation covers

These articles walk you through your work environment, project setup, and the core API concepts, roughly in
the order you will need them. Budget an hour or two and you will be building at full speed.

Start at [Download PixelScript](./getting-started-001-download.md), or jump straight to the
[API reference](./api.md) if you already have a server running.

## Catch the spark, not the gospel

A lot of PixelScript is best understood by doing. The reload tree in particular is hard to put into words,
but you will know the moment it clicks. Follow the guide, or poke at the example project that ships inside
the plugin jar under `scripts/`.

The API surface documented here looks small, and that is the point. **Everything** you can do in a regular
Java plugin you can do in PixelScript, without the boilerplate. The globals are the shortcuts; the Bukkit
API is the actual toolbox.

## On the horizon

PixelScript is under active development. Currently in flight:

- Native Redis support, shaped like the `Sql` module
- More documentation: error components, debugging guides, more worked examples
- Broader TypeScript tooling
