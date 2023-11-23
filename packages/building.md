---
layout: post
title: Building a package
permalink: /packages/building
---

This article explains how the `builder` tool builds a package, using a [formula](/packages/formula) for instructions.

The build process is subdivided into these main steps:

- [Preparing the environment](#prep-env)

- Building the package

- Patching binaries and libraries and checking the package contents

- Archiving and signing

## Terminology

This article uses some shorthands for the directories the builder tool uses for building packages:

- `dist`: The dist directory is the `/acacia` directory by default, this is where the builder will search for dependencies and toolchains

- `buildroot`: The directory where the builder will create a new root filesystem for the build process to `chroot` into. It consists of multiple mounts that go into that directory, explained in the [mounting section](#mounting)

# Preparing the environment <a id="prep-env"></a>

An AcaciaLinux package is built in a completely isolated environment, this means that the package that gets built can only see the dependencies and files it requires, specified in the [formula's dependencies](/packages/formula#dependencies).

## Mounting <a id="mounting"></a>

The `buildroot` is constructed using a combination of `overlayfs` and multiple `bind mounts`:

- `/` `[overlayfs]`: The root of the `buildroot` is constructed using an overlayfs including the following lowerdirs:
  
  - A `base` filesystem containing a minimal root filesystem with the `/dev`, `/proc`... directories
  
  - `target_dependencies`: All the `/root` directories of the `target_dependencies`

- `/acacia` `[bind (ro)]`: The `acacia` directory gets mounted into the `buildroot` for the toolchain to be available

- `/formula` `[overlayfs]`: The parent directory of the formula, this passes in the scripts but makes them immutable

- `/install` `[bind (rw)]`: The root of the final package, the `PKG_INSTALL_DIR` variable will be set to `/install/data`.

# Package validation and patching

Once a package is built, it will run through a suite of validators for each file based on their file type:

- `ELF files`:
  
  - Patch the `Interpreter` if available (`ld-linux...`)
  
  - Patch the `RUNPATH` if available (Shared libraries)

- `symlinks`:
  
  - Validate the destination of symlinks, so no broken symlinks are possible

- `scripts` (files with a shbang):
  
  - Adjust the `SHBANG` to the correct executable (Not `#!/bin/sh`, but `#!/acacia/<arch>/bash/<version>/root/bin/sh`)
