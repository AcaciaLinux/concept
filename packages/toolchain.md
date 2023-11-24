# Toolchain

The AcaciaLinux builder uses toolchains to build its packages. A toolchain is a directory with the following contents:

- `/host`: The directory containing the host programs that are used to compile the package.

The builder will construct a build root from the following things:

- A basic filesystem containing kernel filesystem mount points (`/dev`, `/sys`, `/proc`, `/run`...)

- The toolchain directory, exposing the `/host` and `/acacia` directories if available
