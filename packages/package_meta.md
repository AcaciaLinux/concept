---
layout: post
title: Package Metadata
---

Each package has some metadata the tooling uses to get crucial information about the package, found in the package installation directory ( `/acacia/<arch>/<name>/<version>/package.toml`)

# `package.toml`:

```toml
version = 1

[package]
name = "Package name"
version = "Package version"
arch = "Package architecture"
real_version = Real version
description = "Package description"

[[dependencies]]
name = "Dependency name"
arch = "Architecture"
req_version = "Requested version"
lnk_version = "Linked version"
```

# Field definitions

### `package.name`

The name of the package

### `package.version`

The version of the package

### `package.arch`

The architecture the package is in

### `real_version`

The real version of the package

### `description`

A description for the package

### `package.dependencies`

An array of dependencies the package has

### `package.dependencies.name`

The name of the dependency

### `package.dependencies.arch`

The architecture of the dependency

### `package.dependencies.req_version`

The version of the dependency requested by the package

### `package.dependencies.lnk_version`

The version of the dependency that is linked to the package
