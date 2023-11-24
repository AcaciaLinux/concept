---
layout: post
title: Building a package
permalink: /packages/building
---

This article explains how the `builder` tool builds a package, using a [formula](/packages/formula) for instructions.

The build process is subdivided into these main steps:

- [Preparing the environment](#prep-env)
  
  - [Scanning and indexing dependencies](#prep-env-deps)
  
  - [Mounting](#prep-env-mounting)

- [Building the package](#building)
  
  - [Build steps](#building-steps)
  
  - [Environment variables](#building-envvars)

- [Validating and patching the package](#validating)

- [Archiving and signing](#archiving)

## Terminology

This article uses some shorthands for the directories the builder tool uses for building packages:

- `dist`: The dist directory is the `/acacia` directory by default, this is where the builder will search for dependencies and toolchains

- `buildroot`: The directory where the builder will create a new root filesystem for the build process to `chroot` into. It consists of multiple mounts that go into that directory, explained in the [mounting section](#prep-env-mounting)

# Preparing the environment <a id="prep-env"></a>

An AcaciaLinux package is built in a completely isolated environment, this means that the package that gets built can only see the dependencies and files it requires, specified in the [formula's dependencies](/packages/formula#dependencies).

The `root` directories of the `target_dependencies` are part of the `overlayfs` lower dirs, this allows the compiler and the build tooling to find all files at their default locations without the need of searching them in their respective `/acacia` directories.

### Scanning and indexing dependencies <a id="prep-env-deps"></a>

Prior to creating a `buildroot`, the builder ensures, that all dependencies are satisfied. It retrieves the current versions of all packages and stores them for computing the dependencies of the built package.

After gathering informations about the dependencies, the builder indexes all the dependencies for their files. This allows the builder to know where to find dependencies for the built binaries in the patching stage to set them correctly and include the correct dependencies.

### Mounting <a id="prep-env-mounting"></a>

The `buildroot` is constructed using a combination of `overlayfs` and `bind` mounts:

- `/` `[overlayfs]`: The root of the `buildroot` is constructed using an overlayfs including the following lowerdirs:
  
  - A `base` filesystem containing a minimal root filesystem with the `/dev`, `/proc`... directories
  
  - `target_dependencies`: All the `/root` directories of the `target_dependencies`

- `vkfs`: Kernel virtual filesystems (`/dev/{pts}`, `proc`, `tmpfs`, `sysfs`) to allow for successfull chrooting

- `/acacia` `[bind (ro)]`: The `acacia` directory gets mounted into the `buildroot` for the toolchain to be available

- `/formula` `[overlayfs]`: The parent directory of the formula, this passes in the scripts but makes them immutable

- `/<pkg_name>` `[bind (rw)]`: The root of the final package, the `PKG_INSTALL_DIR` variable will be set to `/<pkg_name>/data`.

# Building the package <a id="building"></a>

With the build environment in place, the builder constructs the environment variables for the `chroot` environment and chroots into the `buildroot`, executing the commands specified in the build steps with the prefix of `sh -c` to allow direct commands.

The builder changes the spawned processes working directory to the path where the formula is mounted, so that `./` references do work:

```shell
env -C $FORMULA_DIRECTORY sh -c $BUILDSTEP_COMMAND
```

### Build steps <a id="building-steps"></a>

There are 4 (optional) build steps a formula runs through while building a package:

- `prepare`: Prepare the sources for compilation

- `build`: Compile the sources

- `check`: Run the test suite if desired / available

- `package`: Installs the package into `$PKG_INSTALL_DIR` and adjusts it for its destination

### Environment Variables <a id="building-envvars"></a>

The builder does not inherit **any** environment variables from the parent process to provide a clean environment for the packages to build in. The only variables passed in are listed here:

* `PATH`: Exposes just the binaries of the host toolchain and the `host_dependencies`

* `PKG_NAME`: The name of the package

* `PKG_VERSION`: The version string of the package

* `PKG_RELV`: The real version of the package

* `PKG_ARCH`: The architecture the package is built for

* `PKG_INSTALL_DIR`: The directory the package should install into

* `PKG_ROOT`: The root the package should expect its root at: `/pkg/<arch>/<pkg_name>/<pkg_version>/root`

# Validating and patching the package <a id="validating"></a>

Once a package is built, it will run through a suite of validators for each file based on their file type:

- `ELF files`:
  
  - Patch the `Interpreter` if available (`ld-linux...`)
  
  - Patch the `RUNPATH` if available (Shared libraries)

- `symlinks`:
  
  - Validate the destination of symlinks, so no broken symlinks are possible

- `scripts` (files with a shbang):
  
  - Adjust the `SHBANG` to the correct executable (Not `#!/bin/sh`, but `#!/acacia/<arch>/bash/<version>/root/bin/sh`)

If desired, this step will use the `strip` command on `ELF` files to reduce their file size.

# Archiving and signing the package file <a id="archiving"></a>

### Archiving

This step constructs an AcaiaLinux package archive from the raw directory to ease distribution of the package.

More information on the layout of a package archive can be found [here](/packages/package).

### Signing

If desired, the packager will use the packagers keys to sign the package file.
