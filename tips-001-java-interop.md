---
project: pixelscript
slug: interop-quirks
title: Java interop
description: Working around the rough edges where JavaScript values meet Java method signatures.
status: stable
tags: [tips, interop, java, casting, overloads]
updated: 2026-07-28
parent: Tips & tricks
nav_order: 1
permalink: /interop-quirks/
---
# Java interop

*Interop* is short for interoperability: PixelScript's ability to call Java code and be called by it.
Values are converted automatically in both directions, and it works well almost all of the time.

This article is about the "almost".

## Numeric arguments and ambiguous overloads

Java has `int`, `long`, `float`, `double`, `short` and `byte`. JavaScript has one `Number`. When you call a
Java method, the runtime picks a conversion based on the method signature.

That works until a class has several overloads that differ **only** in their numeric types. The classic
offender is `World#spawnParticle`, which has a pile of overloads mixing `int` and `double`. Given a plain
JS number, the runtime cannot tell which one you meant, and the call fails.

Six globals let you say it explicitly:

| Function | Produces |
|----------|----------|
| `asInt(value)` | `java.lang.Integer` |
| `asLong(value)` | `java.lang.Long` |
| `asFloat(value)` | `java.lang.Float` |
| `asDouble(value)` | `java.lang.Double` |
| `asShort(value)` | `java.lang.Short` |
| `asByte(value)` | `java.lang.Byte` |

```javascript
// May fail: which overload is this?
world.spawnParticle(Particle.FLAME, x, y, z, count, offsetX, offsetY, offsetZ);

// Unambiguous
world.spawnParticle(
  Particle.FLAME,
  asDouble(x), asDouble(y), asDouble(z),
  asInt(count),
  asDouble(offsetX), asDouble(offsetY), asDouble(offsetZ)
);
```

That example casts everything for clarity. In practice one hint is usually enough to disambiguate, so start
by casting the single argument that differs between the candidate overloads.

This is rare, and when it happens it is a symptom of an overloaded Java API rather than anything you did
wrong.

## Java collections are not JavaScript collections

`Bukkit.getOnlinePlayers()` returns a Java `Collection`, not an array. It has `size()`, not `.length`, and
it does not have `map` or `filter`.

```javascript
const players = Bukkit.getOnlinePlayers();

players.size();              // works
players.length;              // undefined
players.forEach(p => ...);   // works, this one is bridged

// To get a real JS array
const names = [];
players.forEach(p => names.push(p.getName()));
names.filter(n => n.startsWith('a'));  // now this works
```

The same applies the other way. A Bukkit method that wants a `List` will not take a JS array, so build a
real one:

```javascript
const lore = new ArrayList();
lore.add(mm('<gray>A mysterious item'));
meta.lore(lore);
```

`ArrayList`, `HashMap`, `HashSet` and `List` are [preloaded globals](./spec-004-bukkit.md).

Java `Map#forEach` takes a `BiConsumer`, so the callback receives `(key, value)`, the opposite order from
JavaScript's `Map#forEach`:

```javascript
javaMap.forEach((key, value) => { /* ... */ });   // Java order
jsMap.forEach((value, key) => { /* ... */ });     // JavaScript order
```

## Numeric map keys

JavaScript numbers arriving in Java get boxed, and `5` and `5.0` may not compare equal as map keys. If you
are keying on numbers, either use a JS `Map` (which the runtime keeps on the JS side), or normalise with
`Math.trunc()` first:

```javascript
this.elements = new Map();  // JS Map: avoids the boxing mismatch entirely

setElement(slot, element) {
  this.elements.set(Math.trunc(slot), element);
}
```

## `null` versus `undefined`

Java returns `null`. JavaScript has both `null` and `undefined`, and truthiness checks treat them the same,
so `if (player)` is usually fine. Be explicit when it matters:

```javascript
const player = Bukkit.getPlayer('Notch');
if (player !== null && player.isOnline()) { /* ... */ }
```

## Getting the `Class` object

Some APIs want `java.lang.Class` rather than the type. Use `.class`:

```javascript
const ArchModule = Script.loadClass('dev.pixelib.pp.core.arch.ArchModule').class;
plugin.getModule(ArchModule);
```

## Strings

Java `String` and JS string are converted transparently, and JS string methods work on values that came
from Java. Note that Java's `String#contains` also survives, so both of these work:

```javascript
javaString.contains('x');   // Java method
javaString.includes('x');   // JS method
```

Prefer the JS forms for consistency.

## Related

- [Interfaces and abstract classes](./tips-002-interfaces-and-abstract-classes.md) for implementing Java types in JS
- [Implementing Java interfaces](./spec-014-java-implementations.md) for exposing JS to Java plugins
