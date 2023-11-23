---
layout: post
title: Formula
permalink: /packages/formula
---

# Package formula

AcaciaLinux uses formulae to build its packages. A formula consists of a `toml` file that can link to other files relative to it for building.

More information on how packages are built can be found [here](/packages/building).

# TOML file

```toml
version = 1

[package]
name = "package name"
version = "package version"
real_version = 0
description = "Some description for the package"

arch = ["architecture"]
strip = true

host_dependencies = ["build tools"]
target_dependencies = ["linked packages"]
extra_dependencies = ["additional dependencies"]

prepare = "prepare step"
build = "build step"
check = "check step"
package = "package step"

[[package.sources]] # There can be multiple source files
url = "The url to the file to fetch"
dest = "The destination to fetch the file to"

[[package.maintainers]] # There can be multiple maintainers
name = "The name of the maintainer"
email = "The EMail address of the maintainer"
```

# Dependencies <a id="dependencies"></a>

- `host_dependencies`: These dependencies are used by the host process to build the package (the toolchain)

- `target_dependencies`: The dependencies linked-to by the built package

- `extra_dependencies`: Dependencies that are not required at build-time, but at run time

AcaciaLinux is built with multiple architectures in mind, so its build system assumes a cross compilation by default, optimizing the packaging process for exactly that use case. The builder uses a toolchain and the `host_dependencies` using the architecture of the building system to construct a root at `/host`.

The `target_dependencies` are the dependencies that the newly built package links against. They are populated at `/`, so that the build environment meets the expectations of most of the build tools and compilers. After packaging, the builder checks all the binaries that are part of the built package for linkage against the wrong libraries to prevent cross-compilation issues. After that, the builder patches all the ELF binaries and notes down the dependencies needed by the binaries. Any target dependency that is not linked to by the ELF files does not get included as a dependency of the final package. That is the job of the `extra_dependencies`, they get included as dependencies regardless if something links to them or not. This allows the maintainer to include dependencies that are not caught by the dependency analyzer.

The build process gets spawned with the `PATH` variable set to include binaries from the `/host` directory so that the building machine uses compatible binaries and does not leak any host binaries and dependencies into the target package.

# Field definitions

### `name`

The name of the package

This field is available to the build scripts under the `PKG_NAME` environment variable

### `version`

The version string of the package

This field is available to the build scripts under the `PKG_VERSION` environment variable

### `real_version`

The real version of the package, allows for changes in the packaged content without changing the version

This field is available to the build scripts under the `PKG_RELV` environment variable

### `description`

A short description of the package to inform the user about the package contents

### `arch`

An array of architectures this `formula` can build binaries for.

The architecture string is the one output by the `uname -m` command and is part of the toolchain identifier (`x86_64`, `aarch64`).

Architecture-independent packages may use `*` instead of the architectures array: `arch = "*"`.

This field is available to the build scripts under the `PKG_ARCH` environment variable, set to the architecture the package is built for.

The builder will refuse to build a formula for a non-supported architecture, if it is not in the `arch` array to protect from build failures.

### `strip`

Tells the builder if the binaries should be stripped using the `strip` command to reduce file size

### `prepare`, `build`, `check`, `package`

The build commands that get executed in the 4 steps of building:

- `prepare`: Prepare the sources for building, invoke configure scripts and check for dependencies

- `build`: Execute the build command to build the binaries

- `check`: Run the test suite

- `package`: Install the package into the `PKG_INSTALL_DIR`

#### Environment variables:

The builder exposes the following environment variables to the scripts:

- `PKG_NAME`: The name of the package
- `PKG_VERSION`: The version string of the package
- `PKG_RELV`: The real version of the package
- `PKG_ARCH`: The architecture the package is built for
- `PKG_INSTALL_DIR`: The directory the package should install into
  - Example for `make`: `DESTDIR="$PKG_INSTALL_DIR"`
- `PKG_ROOT`: The root the package should expect its root at: `/pkg/<pkg_name>/<pkg_version>/root`
  - Example for `configure`: `--prefix="$PKG_ROOT"`
