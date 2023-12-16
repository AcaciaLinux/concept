---
layout: post
title: Package versioning
---

Each piece of software has a version attached to it, which indicates the compatibility and features available.

There are 3 pieces that make up a version of a package:

- Source version (`version`)

- Package version (`pkgver`)

- Build ID

# Source version (`version`)

This is the main version of the package and its sources. It is the version string that is used by the package provider and normally uses semantic versioning.

**Example:**

The `glibc` package has `2.38` as its current version. This is the source version.

# Package version (`pkgver`)

Sometimes, there is the need to publish multiple versions of a package within the same source version, but they are not binary compatible. This could happen due to bugfixes or changes in the packaging process.

Such updates change the `pkgver`. It will reset to 0 each time the source version changes.

# Build ID

The build id provides an additional layer of versioning for the packages.

Some hotfixes in the form of patches do not provide a new version of a package, but rather just a new build. If a package has the same version and is binary-compatible, but has another build ID, the package manager assumes binary compatibility and replaces the old with the new one.

For this in-place upgrade to be possible, the source version and the package version need to be the same!

> **Warning**
> 
> The package manager treats packages with the same `version/pkgver` as binary compatible! Incompatible changes should change the `version` or `pkgver`

# Procedures

If an update to a package introduces breaking changes, or a new source package version all together, the version string should reflect the new version in the package sources.

If the packager applies a hotfix or improves the packaging by enabling features that do not correspond to a new package version and does **NOT** introduce breaking changes, the version string should stay the same to allow the package manager to replace the old package.

> **Note**
> 
> The package manager will not overwrite packages with different version strings, but rather install them side-by-side
