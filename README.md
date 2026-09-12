# crypto-nv

A cryptographic hash turns a message of any length into a short, fixed-length
digest. A message authentication code (MAC) does the same under a secret key, so
that only a holder of the key can produce or check the value. This package
implements four hashes and the HMAC construction over them, in novo-lang, with
no dependencies.

| Algorithm | Specification | Digest | Block |
| --- | --- | --- | --- |
| SHA-256 | [FIPS 180-4](https://csrc.nist.gov/pubs/fips/180-4/upd1/final) | 32 bytes | 64 bytes |
| SHA-512 | FIPS 180-4 | 64 bytes | 128 bytes |
| SHA-1 | FIPS 180-4 | 20 bytes | 64 bytes |
| MD5 | [RFC 1321](https://www.rfc-editor.org/rfc/rfc1321) | 16 bytes | 64 bytes |
| HMAC over any of them | [RFC 2104](https://www.rfc-editor.org/rfc/rfc2104) | the hash's digest | — |

SHA-256 and SHA-512 are for work you are starting today. SHA-1 and MD5 are here
because Git object names, old TLS suites and half the package indexes on the
internet still publish them, and reading those needs an implementation.

Two other packages on the registry are defined in terms of this one:
[hkdf-nv](https://novo-lang.org/packages/hkdf-nv), whose key derivation is a
chain of HMACs, and any caller of
[blake2-nv](https://novo-lang.org/packages/blake2-nv) that needs a constant-time
digest comparison, which lives here as `digest.ct_eq`.

## What it is for

novo-lang's standard library already has `std.crypto`, which is OpenSSL and is
faster. The difference is where this package runs and what it costs.

The digest state is a `@value` struct. It lives on the caller's stack or in
`.bss`, with no heap cell and no reference count. The compression functions
allocate nothing at all, and the modules that hold them are declared to build
for a microcontroller with no heap allocator: every build type-checks them
against that restriction, and every test run cross-compiles them for a
Cortex-M4. `std.crypto`, by contrast, runs only on a host, takes `Str`, and
offers only the one-shot form.

This is therefore the implementation to put in firmware, or on a code path you
have promised will not allocate.

## Install

```
novo pkg add crypto-nv
```

## Example

```novo
use std.bytes
use hashing

fn main() [io]
    println(bytes.to_hex(hashing.sha256(bytes.from_str("abc"))))
    // ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad

    let tag = hashing.hmac_sha256(bytes.from_str("key"),
                                  bytes.from_str("message"),
                                  bytes.zeros(hashing.SHA256_DIGEST_BYTES))
    println(bytes.to_hex(tag))
```

Build and test with:

```bash
novo pkg build                       # the package compiles
novo test tests                      # 34 tests: the published vectors, and more
```

## What the package contains

| Module | Contents |
| --- | --- |
| `word` | The word algebra shared by all four hashes: masks, rotations and the wrapping 64-bit add, and the `Block` type. Builds for a microcontroller. |
| `sha256_core`, `sha512_core`, `sha1_core`, `md5_core` | One hash each, at the lowest level: the state as a plain value, `new`, and the compression function. Each imports only `word`. Builds for a microcontroller. |
| `digest` | The pieces that need `Bytes`: reading a block out of a buffer, the padding rules, and the constant-time comparison `ct_eq`. Host only. |
| `hashing` | The interface most programs call: one-shot digests, the streaming form, HMAC, and the digest and block length constants, for all four hashes. Host only. |

## How to choose an entry point

Every hash has a **one-shot** form and a **streaming** form.

The one-shot form is a single call that allocates the result buffer for you.

```novo
let d = hashing.sha256(data)                       // one call, one allocation
```

The streaming form allocates nothing. Start a state, absorb the message in
pieces, then finish into a buffer you own.

```novo
var st = hashing.sha256_new()                      // nothing allocated
for chunk in block_aligned_chunks
    st = hashing.sha256_update(st, chunk)
let d = hashing.sha256_finish(st, whatever_is_left, out)
```

`finish` writes into a buffer you own and hands it back, so hashing a thousand
messages needs one 32-byte buffer rather than a thousand.

**Firmware calls a `*_core` module directly.** Those modules use no `Bytes`, no
strings and no lists, so they compile for a device with no heap. See "Running on
a microcontroller".

## The rules a user needs

1. **`update` absorbs whole blocks only.** The state carries no partial block,
   which is what keeps it nine inline words with nothing to own. Bytes past the
   last whole block are left unabsorbed. `finish` takes any length, so the
   remainder goes there.
2. **Any chunk size that is a multiple of 128 satisfies all four hashes at
   once.** 4096, 8192 and 65536 all are. If your chunks are irregular, hand each
   one to `finish` on a fresh state, or buffer them yourself.
3. **Compare tags with `digest.ct_eq`, never with `==`.** See "Timing
   behaviour".
4. **`compress` takes a `Block`, which is sixteen words, not a buffer.** You
   already have the bytes somewhere, and turning sixteen of them into words is
   arithmetic. This is what lets the core modules need no allocator.
5. **SHA-1 and MD5 are for reading formats that specify them, not for securing
   new ones.** Both are broken for collision resistance, MD5 catastrophically
   and SHA-1 thoroughly. Their documentation comments say so at more length.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device with no
heap allocator, and the compiler checks that claim on every build. Here the
claim covers `word` and the four `*_core` modules. They contain only integer
arithmetic over fixed-size values.

```novo
use word
use sha256_core

fn main() [hw]
    let st: Sha256 = sha256_core.new()
    let next: Sha256 = sha256_core.compress(st, block_from_your_dma_buffer())
```

```bash
novo build --target=nrf52-qemu src/main.nv
```

The `Bytes` half of the package, `hashing` and `digest`, stays on the host. It
is a separate module for exactly this reason: one host-only function anywhere in
a compilation unit is an undefined symbol at link time on the device, whether or
not the firmware ever calls it.

## What it costs

Measured on the emitted LLVM and asserted by the test suite: **63 core
functions, zero `novo_alloc` calls.** The compression functions, the word
algebra, the padding, `update` and `finish` allocate nothing. The only functions
here that touch the heap are `hashing.<alg>` and `hashing.hmac_<alg>`, which
allocate the result buffer they hand back, and HMAC two block-sized scratch
buffers besides.

The state is nine `Int` fields: eight chaining words and a byte count, laid out
inline. Copying one costs those nine words. There is no cell, no header, no
reference count and nothing to drop.

## Timing behaviour

- **`digest.ct_eq` is constant-time.** It reads both buffers whole and folds
  every difference into one accumulator, so the work depends on the lengths and
  never on the contents. A tag must be checked with it. `==` stops at the first
  differing byte, and the time it took says how many bytes matched, which is
  enough to forge a tag one byte at a time.
- **The hashes are not constant-time, and do not need to be.** Their input is
  the message, which is not the secret.
- **HMAC's key-dependent work is the same table-free arithmetic on every
  input.** No claim beyond that is made here, and none should be relied on.

## What is not included

- **Ciphers and signatures.** AES-GCM, ChaCha20-Poly1305 and signature schemes
  are separate packages. This one is hashes and HMAC.
- **Speed.** The code is written to be read against the standard that defines
  it: the message schedule and the rounds are spelled out one line each, so a
  reader can check them against the specification line by line. Where throughput
  matters and a heap is available, `std.crypto` is OpenSSL.
- **A partial-block buffer inside the state.** This is why `update` takes whole
  blocks. A `@value` struct's inline `[T; N]` array cannot be written into and
  has no functional-update form (SPEC section 14.3), so a 64-byte scratch buffer
  inside the state would have to be respelled element by element on every byte.
  When value-typed fixed-capacity storage lands, `update` will take any length
  and this restriction goes away without any signature changing.
- **An `[alloc]` effect on the functions that allocate.** The language has no
  such effect yet, so `hashing.sha256` and `hashing.sha256_update` both declare
  the empty effect list `[]` even though one of them allocates. When the effect
  exists, the one-shot wrappers will carry it and the split described above
  becomes machine-checkable.
- **A BLAKE hash.** [blake2-nv](https://novo-lang.org/packages/blake2-nv) is the
  package for BLAKE2.

## Related packages

- [blake2-nv](https://novo-lang.org/packages/blake2-nv) is BLAKE2b and BLAKE2s.
  A keyed BLAKE2 is a MAC in its own right and needs no HMAC wrapper. It depends
  on nothing, so a device that needs only a hash need not link this package as
  well.
- [hkdf-nv](https://novo-lang.org/packages/hkdf-nv) derives many keys from one
  secret. It is defined over the HMAC published here.
- `std.crypto` in the standard library is OpenSSL: faster, host only, `Str`
  shaped, one-shot only.

## The reference

Every public function, its signature, its declared effects and a worked example
of each entry point are on
[the package's page](https://novo-lang.org/packages/crypto-nv), generated from
these sources at every publish. Listing the names here as well would be a second
original, and the second original is the one that goes stale.

## Tests

```bash
novo test tests                             # every suite
novo test tests/hash_vectors_tests.nv       # the published digests
novo test tests/hmac_vectors_tests.nv       # RFC 4231 and RFC 2202
novo test tests/streaming_tests.nv          # a million bytes, streamed
novo test tests/word_tests.nv               # the pieces, one at a time
```

The expected digests are the ones FIPS 180-4, RFC 6234, RFC 1321, RFC 4231 and
RFC 2202 publish, checked against OpenSSL before they were written down, which
is what makes them evidence about this implementation rather than about whoever
typed them. The round constants are generated from the square roots, cube roots
and sines that define them, never transcribed.

`tests/alloc_probe.nv` and `tests/embedded_probe.nv` are the two probes the
suite compiles: the first asserts the allocation count above, the second builds
the core modules for the device target.

## Implementation status

Everything listed here is implemented and passing.

| Item | Implemented |
| --- | --- |
| `hashing.sha256`, `.sha512`, `.sha1`, `.md5` | yes |
| `hashing.<alg>_new`, `_update`, `_finish`, for all four | yes |
| `hashing.hmac_sha256`, `.hmac_sha512`, `.hmac_sha1`, `.hmac_md5` | yes |
| The eight digest and block length constants | yes |
| `digest.ct_eq` | yes |
| `digest`'s block readers and padding functions | yes |
| `sha256_core`, `sha512_core`, `sha1_core`, `md5_core`: the state, `new`, `compress` | yes |
| `word`: the masks, rotations, wrapping add and `Block` | yes |

## Licence

Apache-2.0. See `LICENSE`.
