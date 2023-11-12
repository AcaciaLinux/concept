# Linking packages

Linking packages to eachother happens deterministically and reliably. After compilation, each `ELF` file gets some of its properties changed:

#### Interpreter

AcaciaLinux allows multiple versions of the C library to coexist, providing multiple versions of the `ELF` interpreter. Changing the interpreter at compile-time is not trivial, so AcaciaLinux does it in post, taking each `ELF` binary and changing its `Interpreter` to the one of the C library used for building.

#### RUNPATH

AcaciaLinux packages are very specific in the version of their dependency libraries and use the `RUNPATH` entries in the `ELF` file to guide the dynamic linker to the desired versions of the library files. This, too happens after compiation to ease management. As with the `Interpreter`, the `RUNPATH` entries get altered (or added) in the final packaging stage.

# Runtime

At runtime, the following happens:

A binary gets executed using a syscall, which invokes the interpreter of the correct C library, specified by the `Interpreter` field in the `ELF` file. For each dependency library, there is a `RUNPATH` entry that dictates the dynamic linker to the one and only correct dynamic library file needed for execution.

# Advantages

This system allows each binary to receive **exactly** the dependencies it was built against, mitigating the risk of dependency hell or updates breaking binaries.

Additionally, this eases package management, removing the need of keeping track of packages and their files when they get installed, removed, updated or altered. The packages can simply be read-only, speeding up installation, too.

Lastly, this approach to linking binaries and libraries allows for really cool stuff, like having multiple versions of not only libraries but also the C library coexist peacefully. It gets even better: One can have `musl` and `glibc` installed at the same time in multiple different versions, so every binary can pick its favourite one.

# Building

Yes, building is hell, but not with the right tooling. Using the correct tooling allows for a build root to be constructed for every package, composing an `overlayfs` with all dependencies linked together into one root filesystem for linking:

- The build daemon constructs a combined filesystem for building

- The package gets built, using its final destination as a prefix for building

- Once the package is built, the deamon tears down the build filesystem and modifies all the `ELF` files to link against the packages used for building

- The daemon packages everything up nicely, ready for installation by the package manager

### Build advantages

This can massively speed up build times, removing the need of painfully installing packages every time a package is to be built. Using this approach, constructing a build root is as simple as creating an overlayfs from the dependencies. If a dependency is needed for multiple packages, it can be reused without any concern.

Furthermore, this allows for clean and isolated builds. Each build environment is controllable and repeatable, no matter which system. The packages get built in a root that only contains what is specified and nothing more.

# System root

To provide a "normal" Linux rootfs, the package manager can construct one using symlinks to all the dependencies needed for the root. This approach of packaging allows for multiple different environments to be constructed from packages, allowing infinite customization.

Constructing such a "normal" Linux root filesystem allows for programs and binaries to be compiled without the use of all the fancy packaging magic.
