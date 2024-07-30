# Object file

An object file (or object) is simply a binary blob that can be anything.

# Binary structure

The file is stored in `little-endian` binary format, using `UTF-8` for all of its strings, which are not `NULL` - terminated as their length is always defined.

The file starts with the following structure:

| Offset | Count | Description        |
| :----: | :---: | ------------------ |
|   0    |   4   | File magic: `AOBJ` |
|   4    |   1   | Version: `0x00`    |

## Version 0

For version `0x00`, the file continues with the following data:

| Offset | Count | Description                |
| :----: | :---: | -------------------------- |
|   0    |  32   | Sha256 Hash of Data        |
|   32   |   2   | Object type                |
|   34   |   2   | Compression type           |
|   36   |   2   | GPG Signature length (`s`) |
|   38   |   2   | Dependencies count (`d`)   |
|   40   |   8   | Data length (`b`)          |
|   48   |  `s`  | GPG Signature data         |
|        |  `d`  | Dependencies               |
|        |  `b`  | Data                       |

The Sha256 hash is the result of hashing the binary data wrapped in the object file in raw form without any compression applied to it. This is the object ID.

### Dependencies

A dependency declares where an object expects an other object to exist, the dependency is identified with its object ID and the path it expects the object to be at. The path is relative to the path of the depending object.

| Offset | Count | Description     |
| :----: | :---: | --------------- |
|   0    |   4   | OID length      |
|   4    |   4   | Path length     |
|        |       | Object id `OID` |
|        |       | Path            |

# Object type ???

There are different types of objects:

- `0x1___`: AcaciaLinux object

  - `0x1

- `0x2___`: Object

  - `0x21__`: Executable files

    - `0x210_`: Other

    - `0x211_`: ELF files
      - `0x2111`: Executable file
      - `0x2112`: Shared object
      - `0x2113`: Relocatable file
      - `0x2114`: Core file

  - Other
  - ELF
  -

- `0x2___`: Index
