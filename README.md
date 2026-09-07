# crypto-nv

Cryptographic hashes and HMAC, written in Novo and depending on
nothing. SHA-256 and SHA-512 for work you are starting today; SHA-1 and
MD5 because Git object names, old TLS suites and half the package
indexes on the internet still publish them, and reading those needs an
implementation rather than an opinion.

What makes it worth having beside `std.crypto` — which is OpenSSL, and
faster — is where it runs and what it costs. The digest state is a
`@value` struct, so it lives on the stack or in `.bss` with no heap
cell and no reference count, and the compression functions allocate
nothing at all: they are checked at `@tier(embedded)` on every build
and cross-compiled to a Cortex-M4 on every test run. `std.crypto` is
host-only, `Str`-shaped and one-shot. This is the one you can put in
firmware, or on a hot path you have promised will not allocate.

```
novo pkg add crypto-nv
```

```novo
use std.bytes
use hashing

fn main() [io]
    println(bytes.to_hex(hashing.sha256("abc")))
    // ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad

    let tag = hashing.hmac_sha256("key", "message",
                                  bytes.zeros(hashing.SHA256_DIGEST_BYTES))
    println(bytes.to_hex(tag))
```

```bash
novo pkg build                       # the package compiles
novo test tests                      # 34 tests: the published vectors, and more
```

## The two shapes

Every hash has a **one-shot** form and a **streaming** one, and the
streaming form is the real one — the other is it plus an allocation.

```novo
let d = hashing.sha256(data)                       // one call, one allocation
```

```novo
var st = hashing.sha256_new()                      // nothing allocated
for chunk in block_aligned_chunks
    st = hashing.sha256_update(st, chunk)
let d = hashing.sha256_finish(st, whatever_is_left, out)
```

`finish` writes into a buffer **you** own and hands it back, so hashing
a thousand messages needs one 32-byte buffer rather than a thousand.

**`update` absorbs whole blocks only.** The state carries no partial
block — that is exactly what keeps it nine inline words with nothing to
own — so bytes past the last whole block are left unabsorbed, and
`finish` takes any length. In practice this costs a caller nothing: any
I/O buffer size that is a multiple of 128 satisfies all four hashes at
once, and 4096, 8192 and 65536 all are. If your chunks are irregular,
hand each one to `finish` on a fresh state, or buffer them yourself.

## Comparing tags

```novo
if digest.ct_eq(expected, received)
```

Never `==`. `==` stops at the first differing byte and the time it took
says how many bytes matched, which is enough to forge a tag one byte at
a time. `ct_eq` reads both buffers whole and folds every difference into
one accumulator, so the work depends on the lengths and never on the
contents.

## On a device

The compression cores import `word` and nothing else. No `Bytes`, no
lists, no strings — none of which the embedded surface admits — so they
cross-compile on their own:

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

`compress` takes a `Block` — sixteen words — rather than a buffer,
which is why it needs no allocator: you already have the bytes
somewhere, and turning sixteen of them into words is arithmetic. The
`Bytes` half of the package (`hashing`, `digest`) stays on the host;
it is a separate module for exactly this reason, since one host-only
function anywhere in a compilation unit is an undefined symbol at
embedded link time whether or not the firmware ever calls it.

## What it costs

Measured on the emitted LLVM, and asserted by the test suite: **63 core
functions, zero `novo_alloc` calls.** The compression functions, the
word algebra, the padding and `update` and `finish` allocate nothing;
the only functions here that touch the heap are `hashing.<alg>` and
`hashing.hmac_<alg>`, which allocate the result buffer they hand back,
and `hmac` two block-sized scratch buffers besides.

The state is nine `Int` fields — eight chaining words and a byte count
— laid out inline. Copying one costs those nine words; there is no
cell, no header, no reference count and nothing to drop.

## What it deliberately is not

- **It is not constant-time as a whole.** `ct_eq` is. The hashes are
  not, and they do not need to be: their input is the message, which
  is not the secret. HMAC's key-dependent work is the same table-free
  arithmetic on every input, but no claim beyond that is made here and
  none should be relied on.
- **It is not fast.** It is written to be read against the standard
  that defines it — the message schedule and the rounds are spelled
  out, one line each, so a reader can check them line by line. Where
  throughput matters and a heap is available, `std.crypto` is OpenSSL.
- **SHA-1 and MD5 are here to read old formats, not to secure new
  ones.** Both are broken for collision resistance — MD5 catastrophically,
  SHA-1 thoroughly — and their doc comments say so at more length.
- **No ciphers and no signatures.** AES-GCM, ChaCha20-Poly1305 and
  signatures are a later slice. This one is hashes and HMAC.

## What it cannot do yet

**The state cannot buffer a partial block**, which is why `update`
takes whole blocks. A `@value` struct's inline `[T; N]` array cannot be
written into and has no functional-update form (SPEC §14.3), so a
64-byte scratch buffer inside the state would have to be respelled
element by element on every byte. When value-typed fixed-capacity
storage lands, `update` can take any length and the contract in this
README goes away without changing a signature.

**The effect rows say `[]`, not `[alloc]`.** The allocation discipline
the memory model designs — a function that may reach the arena says so
in its row — is not in the language yet, so `hashing.sha256` and
`hashing.sha256_update` carry the same empty row despite one of them
allocating. When the effect exists, the convenience wrappers take it
and the split this README describes in prose becomes checkable.

## The reference

Every public function, its signature, its effect row and a worked
example of each entry point are on
[the package's page](https://novo-lang.org/packages/crypto-nv),
generated from these sources at every publish. A list of names here
would be a second original, and the second original is the one that
goes stale.

## The layout

| Path | |
|---|---|
| `src/word.nv` | the word algebra — masks, rotates, the wrapping 64-bit add — and `Block`. Embedded |
| `src/sha256_core.nv`, `sha512_core.nv`, `sha1_core.nv`, `md5_core.nv` | the state, `new`, and the compression function. Embedded, and importing only `word` |
| `src/digest.nv` | `Bytes` to `Block`, the padding, `ct_eq`. Host |
| `src/hashing.nv` | the host API for all four: streaming, one-shot, HMAC |
| `tests/` | the published vectors, the boundaries, the probes the suite compiles |

## Tests

```bash
novo test tests                             # every suite
novo test tests/hash_vectors_tests.nv       # the published digests
novo test tests/hmac_vectors_tests.nv       # RFC 4231 and RFC 2202
novo test tests/streaming_tests.nv          # a million bytes, streamed
novo test tests/word_tests.nv               # the pieces, one at a time
```

The expected digests are the ones FIPS 180-4, RFC 6234, RFC 1321,
RFC 4231 and RFC 2202 publish, checked against OpenSSL before they were
written down — which is what makes them evidence about this
implementation rather than about whoever typed them. The round
constants are generated from the square roots, cube roots and sines
that define them, never transcribed.

## Licence

Apache-2.0. See `LICENSE`.
