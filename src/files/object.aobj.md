# Object file

An object file (or object) is simply a binary blob that can be anything.

Each object gets identified by the `sha256` hash of its contents.

Typically, the objects are named after the base64 representation of the object's hash. In this case, the `URL` alphabet is used ([`RFC 4648 §5`](https://datatracker.ietf.org/doc/html/rfc4648#section-5)) and padding is used.
