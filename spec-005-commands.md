---
project: pixelscript
slug: api-command
title: Commands
description: Registering commands, tab completers and aliases, with automatic cleanup on reload.
status: stable
tags: [api, commands, tabcomplete]
updated: 2026-07-28
parent: API reference
nav_order: 5
permalink: /api-command/
---
# Commands

Commands are everything in Minecraft, so they get first-class support. Three globals are available in every
script:

| Function | Purpose |
|----------|---------|
| `registerCommand(name, executor)` | Register a command |
| `registerTabCompleter(name, completer)` | Add tab completion to a registered command |
| `registerCommandAlias(name, aliases)` | Add one or more aliases |

Commands are inserted directly into the server's command map, which means they can **override existing
commands**, including ones from other plugins. They are tied to the script that registered them and are
unregistered automatically when it unloads.

## `registerCommand(command: string, executor: (sender, args) => void): void`

The executor receives the `CommandSender` and a plain JS array of arguments. Return values are ignored;
there is no `true`/`false` contract like in Bukkit.

```javascript
const GameMode = $.GameMode;

const gameModes = {
  survival: GameMode.SURVIVAL,
  creative: GameMode.CREATIVE,
  adventure: GameMode.ADVENTURE,
  spectator: GameMode.SPECTATOR,
  '0': GameMode.SURVIVAL,
  '1': GameMode.CREATIVE,
  '2': GameMode.ADVENTURE,
  '3': GameMode.SPECTATOR,
};

registerCommand('gamemode', (sender, args) => {
  if (!(sender instanceof $.Player)) {
    sender.sendRichMessage('<red>This command can only be run by a player.</red>');
    return;
  }

  if (!sender.hasPermission('command.gamemode')) {
    sender.sendRichMessage('<red>You do not have permission to use this command.</red>');
    return;
  }

  if (args.length === 0) {
    sender.sendRichMessage('<gray>Usage: /gamemode <mode>');
    return;
  }

  const mode = args[0].toLowerCase();
  const mapped = gameModes[mode];

  if (!mapped) {
    sender.sendRichMessage(`<red>Unknown gamemode: ${mode}</red>`);
    return;
  }

  sender.setGameMode(mapped);
  sender.sendRichMessage(`<green>Your gamemode has been set to ${mode}.</green>`);
});
```

If the executor throws, the error is logged with a script stack trace and the sender gets a generic
"an error occurred" message. Your command never crashes the server.

## `registerTabCompleter(command: string, completer: (sender, args) => string[]): void`

Called as the player types. Return an array of suggestions. Filtering against what has been typed so far is
your job.

```javascript
registerTabCompleter('gamemode', (sender, args) => {
  if (args.length !== 1) return [];

  const modes = ['survival', 'creative', 'adventure', 'spectator', '0', '1', '2', '3'];
  const input = args[0].toLowerCase();

  return modes.filter(mode => mode.startsWith(input));
});
```

The command must already be registered with `registerCommand` in the same script. Completing online player
names is common enough to be worth a helper:

```javascript
function onlinePlayerNames() {
  const names = [];
  Bukkit.getOnlinePlayers().forEach(p => names.push(p.getName()));
  return names;
}

registerTabCompleter('playtime', (sender, args) => {
  if (args.length !== 1) return [];
  return onlinePlayerNames().filter(n => n.toLowerCase().startsWith(args[0].toLowerCase()));
});
```

## `registerCommandAlias(command: string, aliases: string | string[]): void`

```javascript
registerCommandAlias('gamemode', ['gm', 'gmode']);
registerCommandAlias('playtime', 'pt');
```

Aliases are removed along with the command when the script unloads.

## Middleware

Because an executor is just a function, the cleanest way to share permission checks, cooldowns and sender
type guards is to wrap it. This pattern shows up all over production PixelScript codebases.

```javascript
// util/admin-command-middleware.js
export function makeAdminCommand(permission, handler, options = { mustBePlayer: true }) {
  return (sender, args) => {
    if (options.mustBePlayer && !(sender instanceof $.Player)) {
      sender.sendRichMessage('<red>This command can only be run by a player.</red>');
      return;
    }

    if (!sender.hasPermission(permission)) {
      sender.sendRichMessage('<red>You do not have permission to use this command.</red>');
      return;
    }

    handler(sender, args);
  };
}
```

```javascript
import { makeAdminCommand } from '@/common/util/admin-command-middleware';

registerCommand('gamemode', makeAdminCommand('command.gamemode', (sender, args) => {
  // The guards already ran, get straight to the point
}));
```

Cooldowns work the same way:

```javascript
// util/cooldown-middleware.js
const onCooldown = new HashSet();

export function cooldownMiddleware(seconds, handler) {
  return (sender, args) => {
    const id = sender.getUniqueId();

    if (onCooldown.contains(id)) {
      sender.sendRichMessage('<red>Please wait a moment before using this again.</red>');
      return;
    }

    onCooldown.add(id);
    Scheduler.runAsyncLater(() => onCooldown.remove(id), seconds * 20);

    handler(sender, args);
  };
}
```

```javascript
registerCommand('warp', cooldownMiddleware(5, (sender, args) => { /* ... */ }));
```

Middleware composes, so `cooldownMiddleware(5, makeAdminCommand('perm', handler))` reads exactly the way
you would hope.

## Subcommands

There is no built-in subcommand router. A `switch` is usually enough:

```javascript
registerCommand('playtime', (sender, args) => {
  if (args.length === 0) return showOwnPlaytime(sender);

  const [sub, ...rest] = args;

  switch (sub.toLowerCase()) {
    case 'sessions':
    case 'history':
      return showSessions(sender, rest);
    case 'help':
      return showHelp(sender);
    default:
      // Not a subcommand, treat it as a player name
      return showOwnPlaytime(sender, args);
  }
});
```

## Automatic cleanup

Commands, tab completers and aliases registered through these globals are unregistered when the script
unloads or reloads. You do not need an unload callback for them.

The only thing to keep in mind: because commands go straight into the command map, registering the same
command name from two different scripts means the last one to load wins.
