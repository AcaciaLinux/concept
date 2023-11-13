# Package formula

AcaciaLinux uses formulae for building its packages. They are written in TOML, describing the structure of the resulting package(s).

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

In step `6` of the build pipeline the builder builds a tree of all libraries needed by the `ELF` files in the resulting package and automatically includes them as dependencies in the final package, so there can not be unresolved dependencies.

If a program or library needs a dependency not catched by the builder, the packager can specify it in the `extra_dependencies` array for the builder to include them into the package dependencies.

# Environment variables

The builder exposes the following environment variables to the scripts:

- `FORMULA_NAME`: The name of the formula
- `PKG_NAME`: The name of the package or formula
- `PKG_VERSION`: The version string of the package or formula
- `PKG_RELV`: The real version of the package or formula
- `PKG_INSTALL_DIR`: The directory the package should install into
  - Example for `make`: `DESTDIR="$PKG_INSTALL_DIR"`
- `PKG_ROOT`: The root the package should expect its root at: `/pkg/<pkg_name>/<pkg_version>`
  - Example for `configure`: `--prefix="$PKG_ROOT"`

### A note on inheritance:

All scripts should generally use the `PKG_*` variables, because the builder populates them with the according `$FORMULA_*` fields inherited from the parent formula, even in a `1:1` formula.

# Build pipeline

A build for a package is divided into several steps, constructing a package step-by-step until it is ready for deployment by the package manager

1. Dependency installation
2. Source fetching
3. Build root composition
4. Formula scope (Once per formula):
   1. `prepare`
   2. `build`
   3. `check`
   4. `package`
5. Package scope (Once per package):
   1. `prepare`
   2. `build`
   3. `check`
   4. `package`
6. Dependency analysis, ELF patching and stripping
7. Packaging
8. Signing

### 1: Dependency installation

This step is optional and depends on whether there are some `build_dependencies`. If so, the package manager will install the needed dependencies and the builder appends their `root` directories to the list of `lowerdir` for the `overlayfs` that constructs the `build root`.

### 2: Source fetching

The packager fetches the sources and if they are archives, extracts them in the build directory.

The following strings get replaced:

- `$FORMULA_NAME`: The `name` of the formula
- `$FORMULA_VERSION`: The `version` of the formula

### 3: Build root composition

The builder constructs a build root using `overlayfs` to create a safe jail the build scripts can work in, only the dependencies needed are linked in to ensure a clean build environment.

### 4.1 / 5.1: prepare

> **Note**
> 
> This step gets run in a `chroot` environment in the build root

The `prepare` script, if specified, gets executed, executing the provided string in a `sh` process.

This step should prepare the sources for compilation or deployment, calling `configure`, `cmake` or other tools to prepare everything for the `build` step.

This step should only be used once, even in a `1:n` formula.

### 4.2 / 5.2: build

> **Note**
> 
> This step gets run in a `chroot` environment in the build root

The `build` script, if specified, gets executed, executing the provided string in a `sh` process.

This step should compile all the binaries for the package, calling `make -j$(nproc)` or other build tools.

This step should only be used once, even in a `1:n` formula.

### 4.3 / 5.3: check

> **Note**
> 
> This step gets run in a `chroot` environment in the build root

The `check` script, if specified, gets executed.

This step checks the build artifacts for functionality, run test suites and do sanity checks using `make check` or other test suite runners.

This step should only be used once, even in a `1:n` formula.

### 4.4 / 5.4: package

> **Note**
> 
> This step gets run in a `chroot` environment in the build root

The `package` script, if specified, gets executed.

This script should populate the `$PKG_INSTALL_DIR` with the files needed for installation using `make install` or other installation tools.

> **Note**
> 
> Files that are not in the `$PKG_INSTALL_DIR` by the end of this step do not reach the package archive and are thus excluded from the final package

This step can be used multple times, once per package ideally to justify a `1:n` formula.

### 6: Dependency analysis, ELF patching and stripping

> **Note**
> 
> Refer to the [Linking](linking) article for more information

Once the package is built, the packager iterates over all files and analyzes all `ELF` files in the package, executing following steps:

- **Dependency analysis**: The packager checks that all dependencies in the form of dynamically linked libraries are satisfied
- **ELF patching**: Once all dependencies are ensured, the builder patches the `ELF` files to the correct `Interpreter` and sets all `RUNPATH` entries for all dynamically linked libraries to be discoverable by the dynamic linker.
- **Stripping**: If `strip` is set to `true`, the builder uses the `strip` program on all `ELF` files to significally reduce their size

### 7: Packaging

The builder generates all the necessary files and directories for the package to be ready for archiving, which will transform a directory into an AcaciaLinux package.

### 8: Signing

The builder uses the building maintainer's keys to generate a signature of the package for the package manager to verify package integrity.

# Example: `1:1` formula: `glibc`

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