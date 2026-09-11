# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.1 — 2026-09-08

- **Declares its layer**: `layer = "core"` in the manifest — the public API requires no effects, and `novo pkg publish` now checks the code against that budget.  The layers are described under Design in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).
- `bits.*` calls are the operators they lower to (`&`, `|`, `^`, `<<`, `>>>`, `~`), rewritten by `novo rewrite --bits-to-operators` where the checker proves the operands `Int`; every test vector byte-identical.  Sources reformatted to the canonical form.

## 0.1.0 — 2026-09-07

First release. Hashes and HMAC; ciphers and signatures are a later
slice.

- `hashing.sha256`, `sha512`, `sha1` and `md5` — the one-shot digest of
  a `Bytes`, in a buffer of its own.
- `hashing.<alg>_new` / `_update` / `_finish` — the streaming shape,
  which allocates nothing. `update` takes whole blocks; `finish` takes
  any length and writes into a caller-provided buffer.
- `hashing.hmac_sha256`, `hmac_sha512`, `hmac_sha1` and `hmac_md5` —
  RFC 2104 over any of the four.
- `digest.ct_eq` — the comparison a tag must be checked with.
- `sha256_core` and its three siblings — the state, `new` and
  `compress`, at `@tier(embedded)` and importing only `word`, for
  callers with no heap and no `Bytes`.
- `word` — the shared word algebra and the `Block` type.

SHA-1 and MD5 are for reading formats that specify them. Both are
broken for collision resistance; their documentation says so.
