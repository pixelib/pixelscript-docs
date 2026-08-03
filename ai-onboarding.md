---
title: Onboard your AI
nav_order: 2
permalink: /ai/
---
# Onboard your AI

PixelScript is JavaScript, so coding agents are already fluent in the language. What they do not know is the
runtime: the load tree, the globals, the threading rules, and the cleanup contract. Given those, they are
genuinely productive on a PixelScript codebase. Without them, they will confidently write a Bukkit plugin
in JavaScript syntax and leave listeners leaking on every reload.

This page sets that up in one command.

## The quick version

Run this from your scripts directory (`plugins/PixelScript/scripts`):

```bash
git clone --depth 1 https://github.com/pixelib/pixelscript-docs .pixelscript-docs \
  && cp .pixelscript-docs/CLAUDE.md ./CLAUDE.md \
  && grep -qxF '.pixelscript-docs/' .gitignore 2>/dev/null || echo '.pixelscript-docs/' >> .gitignore
```

That gives you two things:

- **`CLAUDE.md`** in your project root, a condensed reference to the whole runtime. Claude Code reads this automatically at the start of every session.
- **`.pixelscript-docs/`**, the complete documentation in the workspace, so the agent can read the long-form article on scheduling or transactions when it needs the detail.

Nothing in `.pixelscript-docs/` is loaded by PixelScript. The runtime only picks up `.js` and `.ts` files,
and only files directly in `scripts/` load automatically, so a subfolder of Markdown is inert.

## Just the reference file

If you would rather not have the docs in your workspace:

```bash
curl -fsSL https://raw.githubusercontent.com/pixelib/pixelscript-docs/main/CLAUDE.md -o CLAUDE.md
```

## Keeping it current

PixelScript ships often. Refresh both with:

```bash
git -C .pixelscript-docs pull --ff-only && cp .pixelscript-docs/CLAUDE.md ./CLAUDE.md
```

If you have edited `CLAUDE.md` with project-specific notes, keep them in a separate section at the bottom
and merge by hand instead, or move your notes into a second file and reference it.

## Other agents

`CLAUDE.md` is plain Markdown with no Claude-specific syntax, so it works anywhere a tool reads project
context. Rename or symlink as needed:

| Tool | File |
|------|------|
| Claude Code | `CLAUDE.md` |
| Cursor | `.cursorrules` or `.cursor/rules/pixelscript.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Windsurf | `.windsurfrules` |
| Anything else | Point it at `CLAUDE.md` directly |

```bash
# example: same content, Copilot's location
mkdir -p .github && cp .pixelscript-docs/CLAUDE.md .github/copilot-instructions.md
```

## What the reference file covers

- The complete global API surface, in tables rather than prose
- The load tree: root, watched and imported scripts, and which one to reach for
- `Watcher.watch()` path resolution and its single-commit behaviour, the two things everyone gets wrong
- The threading contract, and the read-on-main, work-async, apply-on-main pattern
- The cleanup contract: what unregisters itself and what needs `Script.addUnloadCallback`
- Java interop gotchas, from collections to numeric overloads
- Idioms lifted from production codebases: command middleware, message modules, data-access modules, GUIs

## Adding your own project context

The shipped `CLAUDE.md` describes PixelScript, not your server. Once it is in place, append a section about
your own codebase. Things that pay off immediately:

```markdown
## This project

- Server types: `lobby`, `survival`, `minigames`, selected by `$SERVER_TYPE`
- Entrypoint: `init.ts`, which watches `global/index.js` plus one server module
- Messages go through `@/lib/chat/messages`, never raw `sendMessage`
- Permissions are namespaced `mynetwork.<feature>.<action>`
- We target Paper 1.21 and use Adventure/MiniMessage, never legacy colour codes
```

Or let the agent write it for you. From your scripts directory, ask Claude Code to read the codebase and
append a "This project" section to `CLAUDE.md`. It is a good first task, and it tells you quickly whether
the setup landed.

## A starting prompt

If you want to check that everything is wired up, this is a reasonable first request:

> Read CLAUDE.md and the existing scripts. Then add a `/warp` command as a watched feature under
> `global/features/warps/`, backed by a `DataFile` at `scripts/resources/warps.yml`. Follow the existing
> conventions for permissions and messages.

A correctly onboarded agent will register the feature from a watcher rather than an import, load the
`DataFile` once at the top level rather than per invocation, and not need to be told about cleanup.
