# AcaciaLinux - An introduction

This page gives a general overview over the AcaciaLinux project, what it is and what its goals are.

## What is AcaciaLinux?

AcaciaLinux is a `Linux` distribution that aims to take the headache off users. It aims to combine a couple of core concepts:

- Ease of use

- Immutability

- Hash-based package system

### Ease of use

AcaciaLinux aims to provide a seamless user experience. Some distributions simply get in the way of the user. AcaciaLinux tries to create a seamless and easy-to-use operating system that just works, giving the normal user an operating system that doesn't get in its way and an experienced user enough room to play around.

### Immutability

AcaciaLinux uses immutable packages, this means that once released into the wild, a package will **never** be altered again. There will be new versions, but a package can not be replaced under any circumstance. This does also apply to the installed packages: Each package lives in its own directory that, apart from being populated at installation and removed when no longer needed, does not get altered.

This ensures that a package will never experience any unintended changes that aren't intended by the maintainers or the user.

### Hash-based package system

A package's files are internally known as objects. Each one gets hashed and uploaded into the package system. A package is then just a collection of objects and a where they get placed. This allows for deduplication on the package system and for consistency.

This concept extends to dependencies, too: A dependency is seeded at the object level, meaning that an object depends on another object. The object doesn't care which package provides the dependency as long as the hash matches, the depending object is fine with any package providing it.
