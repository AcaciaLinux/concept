# Package formula

AcaciaLinux uses a formulae for building its packages. These formulae are written in TOML, describing the structure of the resulting package(s).

A formula can link to other files relative to itself. This allows the scripts to be decoupled from the specification, allowing easier management and cleaner code.

Relative paths start from the path the formula lives in. `file.patch` is equal to `./file.patch`.

### Single- vs Multi- formula

A formula can come in 2 forms:

- `1:1`: One formula builds exactly one package, this form does not require a `package` section, because there is no ambiguity.
- `1:n`: One formula can build multiple packages, this form requires `package` sections, communicating the names and packaging procedures for the packages built.

# File structure:

The `toml` file has the following structure:

```toml
[formula.formula_name]
version = "The version string"
real_version = "The real version"
description = "Some description for the formula"

build_dependencies = ["dep1", "dep2"]      # Dependencies needed for the build process
extra_dependencies = ["edep1", "edep2"]    # Dependencies that might are not catched by the dependency analyzer

prepare = "path to the 'prepare' script for the formula"
build = "path to the 'build' script for the formula"
check = "path to the 'check' script for the formula"
package = "path to the 'package' script for the formula"

strip = true        # If the ELF files should be stripped

[[formula.formula_name.sources]] # There can be multiple source files
url = "The url to the file to fetch"
dest = "The destination to fetch the file to"

[[formula.formula_name.maintainers]] # There can be multiple maintainers
name = "The name of the maintainer"
email = "The EMail address of the maintainer"


[formula.formula_name.packages.package_name] # There can be multiple packages in one formula
#
# Each field of the package inherits the value of the formula if not specified
#

version = "The version string"
real_version = "The real version"
description = "Some description for the package"

extra_dependencies = ["edep1", "edep2"]    # Dependencies that might are not catched by the dependency analyzer

prepare = "path to the 'prepare' script for the package"
build = "path to the 'build' script for the package"
check = "path to the 'check' script for the package"
package = "path to the 'package' script for the package"

strip = true        # If the ELF files should be stripped
```

# Dependencies

Dependencies are tricky, but the builder can help the packager with them.

The `build_dependencies` field specifies the dependencies needed for `prepare`, `build`, `check` and `package`. But not all are included in the final package.

In step `8` of the build pipeline the builder builds a tree of all libraries needed by the `ELF` files in the resulting package and automatically includes them as dependencies in the final package, so there can not be unresolved dependencies.

If a program or library needs a dependency not catched by the builder, the package can specify it in the `extra_dependencies` array for the builder to include them into the package dependencies.

# Environment variables

The builder exposes some environment variables to the scripts:

- `FORMULA_NAME`: The name of the formula
- `PKG_NAME`: The name of the package
- `PKG_VERSION`: The version string of the package
- `PKG_RELV`: The real version of the package
- `PKG_INSTALL_DIR`: The directory the package should install into

# Build pipeline

A build for a package is divided into several steps, constructing a package step-by-step until it is ready for deployment by the package manager

1. Dependency installation
2. Source fetching
3. Build root composition
4. `prepare`
5. `build`
6. `check`
7. `package`
8. Dependency analysis, ELF patching and stripping
9. Packaging
   10. Signing

#### 1: Dependency installation

This step is optional and depends on whether there are some `build_dependencies`. If so, the package manager will install the needed dependencies.

#### 2: Source fetching

The packager fetches the sources and if they are archives, extracts them in the build directory.

The following strings get replaced:

- `$FORMULA_NAME`: The `name` of the formula
- `$FORMULA_VERSION`: The `version` of the formula

#### 3: Build root composition

The builder constructs a build root using `overlayfs` to create a safe jail the build scripts can work in, only the dependencies needed are linked in to ensure a clean build environment.

#### 4: `prepare` (chroot)

> **Note**
> 
> This step gets run in a `chroot` environment in the build root

The `prepare` script, if specified, gets executed, calling the `

#### 5: `build`. (chroot)

> **Note**
> 
> This step gets run in a `chroot` environment in the build root

The `build` script, if specified, gets executed.

#### 6: `check`. (chroot)

> **Note**
> 
> This step gets run in a `chroot` environment in the build root

The `check` script, if specified, gets executed.

#### 7: `package`. (chroot)

> **Note**
> 
> This step gets run in a `chroot` environment in the build root

The `package` script, if specified, gets executed.

#### 8: Dependency analysis, ELF patching and stripping

Once the package is built, the packages iterates over all files and analyzes all `ELF` files in the package, executing following steps:

- **Dependency analysis**: The packager checks that all dependencies in the form of dynamically linked libraries are satisfied
- **ELF patching**: Once all dependencies are ensured, the builder patches the `ELF` files to the correct `Interpreter` and sets all `RUNPATH` entries for all dynamically linked libraries to be discoverable by the dynamic linker.
- **Stripping**: If `strip` is set to `true`, the builder uses the `strip` program on all `ELF` files to significally reduce their size

#### 9: Packaging

The builder generates all the necessary files and directories for the package to be ready for archiving, which will transform a directory into an AcaciaLinux package.

#### 10: Signing

The builder uses the building maintainer's keys to generate a signature of the package for the package manager to verify package integrity.

# Example: `1:1` formula

This formula describes just one package, the ratio is `1:1`: 1 formula builds 1 package. This allows for a lot of stuff to be omitted, simplifying the process of packaging.

```toml
[formula.glibc]
version = "2.38"
real_version = 0
description = "The GNU C library"

build_dependencies = ["linux-api-headers"]

prepare = "prepare.sh"
build = "build.sh"
check = "check.sh"
package = "package.sh"

[[formula.glibc.sources]]
url = "https://ftp.gnu.org/gnu/$FORMULA_NAME/$FORMULA_NAME-$FORMULA_VERSION.tar.xz" # The URL to fetch from
# Archives are extracted automatically

[[formula.glibc.maintainers]]
name = "Max Kofler"
email = "max@acacialinux.org"
```

# Example: `1:n` formula

This formula builds 2 packages, requiring a bit more information:

```toml
[formula.gcc]
version = "13.2.0"
real_version = 0
description = "The GNU Compiler Collection"

build_dependencies = ["binutils"]

prepare = "prepare.sh"    # The 'prepare' stage is shared for all packages and gets executed only once
build = "build.sh"        # So is the 'build' stage
check = "check.sh"        # And the 'check' stage

[[formula.gcc.sources]]
url = "https://ftp.gnu.org/gnu/$FORMULA_NAME/$FORMULA_NAME-$FORMULA_VERSION/$FORMULA_NAME-$FORMULA_VERSION.tar.xz"

[[formula.gcc.maintainers]]
name = "Max Kofler"
email = "max@acacialinux.org"

[formula.gcc.packages.gcc]
# The 'gcc' package inherits all properties from the formula
package = "package_gcc.sh"
extra_dependencies = ["some non-linked-to dependency"]

[formula.gcc.packages.libstdcpp]
# The 'libstdcpp' package inherits all properties from the formula
description = "Dynamic libraries for the C++ standard library" # But the description is overridden
package = "package_libstdcpp.sh"
```