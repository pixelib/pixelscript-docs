---
project: pixelscript
slug: download
title: Download PixelScript
description: Where to get a build, which release channel to pick, and where the Java API artifact lives.
status: stable
tags: [installation, download, license]
updated: 2026-07-28
parent: Getting started
nav_order: 2
permalink: /download/
---
# Download PixelScript

PixelScript sits at the heart of your server, and major version updates are scary. That is why builds are
published on separate channels, so you can decide how much risk you want to take.

| Channel | What it is | Use it for |
|---------|------------|------------|
| **latest** | A direct mirror of the main branch. Production-ready, always version bumped, supported until the next major release. | Production |
| **nightly** | The closest thing to a bleeding edge build. Every merged feature is published here automatically. | Staging, compatibility checks before you promote to production |

## Getting a build

Once your account has access, download builds from the artifact platform:

**<https://license-platform.pixelib.dev/artifacts>**

Grab the latest stable build, or a nightly if you are validating an upcoming change against your scripts.

Installing and activating the jar is covered in
[Setting up your work environment](./tutorial-001-setting-up-your-work-env.md).

## The Java API

If you are writing a Java plugin that needs to talk to PixelScript, whether that is a custom extension or
the [JS implementation API](./spec-014-java-implementations.md), the API artifact is on our public Maven
repository.

**<https://repo.pixelib.dev/artifacts/pixelib-public?group=dev.pixelib.pixelscript&artifact=API>**

```gradle
repositories {
    maven { url = 'https://repo.pixelib.dev/artifacts/pixelib-public' }
}

dependencies {
    compileOnly 'dev.pixelib.pixelscript:API:<version>'
}
```

## Next up

Continue to [How to think about a script](./getting-started-002-how-to-think-about-a-script.md)
for the mental model, or skip ahead to [Setting up your work environment](./tutorial-001-setting-up-your-work-env.md)
to get it running.
