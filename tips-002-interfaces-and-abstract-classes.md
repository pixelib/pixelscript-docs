---
project: pixelscript
slug: interfaces-and-abs-classes
title: Interfaces and abstract classes
description: Implementing Java interfaces and extending abstract classes from JavaScript.
status: stable
tags: [jvm, interop, java]
updated: 2026-07-28
parent: Tips & tricks
nav_order: 2
permalink: /interfaces-and-abs-classes/
---
# Interfaces and abstract classes

Some Java APIs will not take a plain callback. They want an object that implements an interface, or a
subclass of an abstract class, and they hold on to it. PixelScript can produce both.

## When to reach for this

Only when a Java API needs to hold a reference to your JavaScript code. This is entirely separate from
writing JavaScript classes for your own use, which needs none of this machinery.

If you are exposing a whole API surface to a Java plugin rather than satisfying a single callback
parameter, use [the implementation API](./spec-014-java-implementations.md) instead. It is type-safe on the
Java side and survives reloads.

## Functional interfaces: just pass a function

Any single-method interface is satisfied by an arrow function. Nothing else is needed.

```java
@FunctionalInterface
public interface MyFunctionalInterface {
    String execute(String message);
}
```

```javascript
MyClass.someFunction((message) => 'Hello, ' + message);
```

`Runnable`, `Consumer`, `Supplier`, `Predicate` and friends all work this way, which is why
[`Scheduler`](./spec-007-scheduler.md) takes plain functions.

## Multi-method interfaces: call the interface as a constructor

Load the interface and call it with an object literal. The runtime generates a real implementing class.

```java
public interface MyInterface {
    void doSomething();
    int calculate(int a, int b);
}
```

```javascript
const MyInterface = Script.loadClass('com.example.MyInterface');

const impl = new MyInterface({
  doSomething() {
    console.log('Doing something!');
  },
  calculate(a, b) {
    return a + b;
  }
});

someJavaApi.register(impl);
```

Method names must match the interface exactly. A real-world example, a Paper chat renderer:

```javascript
const ChatRenderer = Script.loadClass('io.papermc.paper.chat.ChatRenderer');
const Component = Script.loadClass('net.kyori.adventure.text.Component');

const myRenderer = new ChatRenderer({
  render(source, sourceDisplayName, message, viewer) {
    return Component.text()
      .append(getPrefix(source))
      .append(sourceDisplayName)
      .append(Component.text(': '))
      .append(message)
      .build();
  }
});

registerListener(AsyncChatEvent, (event) => event.renderer(myRenderer));
```

## Abstract classes: `Java.extend`

Abstract classes need `Java.extend(BaseClass, overrides)`, which produces a new class you then instantiate
with the parent's constructor arguments.

Here is the canonical case, a ProtocolLib packet adapter:

```javascript
const ProtocolLibrary = Script.loadClass('com.comphenix.protocol.ProtocolLibrary');
const PacketAdapter = Script.loadClass('com.comphenix.protocol.events.PacketAdapter');
const ListenerPriority = Script.loadClass('com.comphenix.protocol.events.ListenerPriority');
const PacketType = Script.loadClass('com.comphenix.protocol.PacketType');

const MyPacketAdapter = Java.extend(PacketAdapter, {
  onPacketReceiving: function (event) {
    console.log('Swing: ' + event.getPlayer().getName());
  }
});

const listener = new MyPacketAdapter(
  __plugin,
  ListenerPriority.MONITOR,
  [PacketType.Play.Client.ARM_ANIMATION]
);

const protocolManager = ProtocolLibrary.getProtocolManager();
protocolManager.addPacketListener(listener);
```

Note `__plugin`, [the PixelScript plugin instance](./spec-004-bukkit.md), passed where the API wants a
`Plugin`.

## Clean up after yourself

Nothing you register directly against a third-party Java API is cleaned up on reload. The runtime only
manages what it handed you: commands, listeners registered through `registerListener`, and scheduler tasks.

Always pair a manual registration with an unload callback:

```javascript
protocolManager.addPacketListener(listener);

Script.addUnloadCallback(() => {
  protocolManager.removePacketListener(listener);
});
```

Skip this and every reload stacks another listener on top of the last one. The symptom is a handler that
fires two, then three, then ten times, and it is the single most common bug in PixelScript codebases that
touch external APIs.

## `super` has limits

Inside a generated class, `super` reaches parent **methods and constructors**, but not parent **fields**.
If you need a protected field, expose it through a getter on the Java side.
