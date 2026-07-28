---
project: pixelscript
slug: script-logic
title: How to think about a script
description: A concept guide for Java and Skript developers moving to PixelScript.
status: stable
tags: [tips, concepts, beginners]
updated: 2026-07-28
parent: Getting started
nav_order: 3
permalink: /script-logic/
---
# How to think about a script

PixelScript has "Script" in the name, which sets the wrong expectation depending on where you are coming
from. Let's clear that up before you write anything.

## If you are coming from Java

**The good news:** PixelScript is Java with JavaScript syntax and hot reloading. That is genuinely it.

When you call `Bukkit.broadcastMessage("Hello World")`, you are calling the real
`org.bukkit.Bukkit.broadcastMessage(String)` method on the real server object. Your code compiles down to
JVM bytecode. The only differences are the syntax and the fact that you can reload without restarting.

What actually changes:

- **Syntax.** JavaScript instead of Java. `let`/`const`, arrow functions, template literals, optional semicolons.
- **Hot reloading.** Save the file, the change applies. No server restart.
- **Dynamic typing.** You do not declare types, though they still exist on the JVM underneath. See [Java interop](./tips-001-java-interop.md) for the rare cases where you have to help the runtime pick an overload.

**The mental shift:** stop thinking "I am learning a scripting language." Everything you know about the
Bukkit API, design patterns, and Java libraries still applies. You can import Java libraries directly and
even [hand JavaScript implementations back to Java plugins](./spec-014-java-implementations.md).

The "script" part only means you can iterate faster. That is the whole trick.

## If you are coming from Skript

**The reality check:** PixelScript is nothing like Skript.

Skript gives you prebuilt commands, event handlers, and utilities designed for non-programmers. It is
batteries included and opinionated, and it holds your hand through everything.

PixelScript is the opposite. It is a blank canvas.

- **No simplified syntax.** Skript lets you write `give player 1 diamond`. PixelScript wants the actual Bukkit call: `player.getInventory().addItem(new $.ItemStack($.Material.DIAMOND, 1))`.
- **You write the logic.** Want a GUI? You write a function that opens an inventory, adds items, and handles clicks. PixelScript hands you the tools; you build the solution.
- **It is real programming.** Functions, classes, modules, and a codebase you organise yourself.

**The mental shift:** Skript is a Lego set with instructions. PixelScript is a bucket of bricks. More
thinking up front, complete control after.

## The PixelScript philosophy

PixelScript is **unopinionated by design**. What it gives you:

- Full access to the Bukkit API and any Java library
- JavaScript (or TypeScript) syntax for faster, cleaner code
- Hot reloading so you can iterate quickly
- Complete freedom to structure your project however you want
- Quality of life bindings for common platform tasks: [Scheduler](./spec-007-scheduler.md), [Sql](./spec-009-database.md), [commands](./spec-005-commands.md), [events](./spec-006-events.md)
- An OOP sandbox you can take as deep as you want

What it deliberately does not give you:

- Prebuilt utilities for GUIs, entity management, or network setup
- Batteries-included solutions
- Training wheels for programming fundamentals

Think of PixelScript as a power tool, not a template. Need GUI utilities? Build them once the way you like,
then reuse them across projects. Need a custom event system? Create it. Need a Java library? Import it.

## Where to steal from

You do not have to start from zero. The plugin jar ships an example project under `scripts/`, which is
copied into `plugins/PixelScript/scripts` on first run. It contains working implementations of:

- A monolithic entrypoint that branches on a `SERVER_TYPE` environment variable
- A moderation system backed by [the Sql module](./spec-009-database.md)
- A playtime tracker with sessions
- A chat formatter using Adventure and MiniMessage
- A complete inventory GUI framework

Read those before you write your own. They are the fastest way to internalise the idioms.

## Next up

[Set up your work environment](./tutorial-001-setting-up-your-work-env.md).
