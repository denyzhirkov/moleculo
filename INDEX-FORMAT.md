# The moleculo index format, version 4

This describes what a shard holds, byte for byte: how its files are framed and
checksummed, how records are stored and addressed, how row identity works,
which rows are live, and how a molecule is laid out in `mol.pack`. It is enough
to write a program, without moleculo, that lists a database's live molecules
with their row ids, their original input records and their stored graphs, and
that verifies every checksum on the way.

It is deliberately not everything. What a **fingerprint bit** means, and what
the **feature statistics** in the manifest count, are named below and not
derived — those are the search engine. The bytes that hold them are specified;
how they were computed is not.

**Version 4 is the format of moleculo 2.0, and it is frozen from 2.0.0-rc.1 on**: a change to it after that is a new version, not an edit to this one. It replaces version 3, the format of every 1.x release, which is
frozen and remains documented here: [Version 3](#version-3) at the end says
what it differs in, because a v3 index is still something you may need to read
— to convert it, or to get your records out of it. A future version will change
the header's version field, and a reader written against this document must
refuse what it does not recognise rather than guess — see
[What a reader must do](#what-a-reader-must-do).

> ⚠ The index is a **derived artifact**. It is reproducible from its input and
> the builder version stamped in its manifest, and it is never edited in place.
> If a tool of yours writes into a shard, everything in this document stops
> applying to it.

---

## Contents

- [The shape on disk](#the-shape-on-disk)
- [`database.json`](#databasejson)
- [`manifest.json`](#manifestjson)
- [The file header](#the-file-header)
- [CRC-32C](#crc-32c)
- [Row identity](#row-identity)
- [The block store](#the-block-store)
- [zstd blocks](#zstd-blocks)
- [The record store: `smi.blk`, `smi.off`, `smi.sum`](#the-record-store-smiblk-smioff-smisum)
- [Fingerprints: `fp.col`, `fp.off`, `fp.sum`](#fingerprints-fpcol-fpoff-fpsum)
- [Popcounts: `popcount.col` and `popcount.sum`](#popcounts-popcountcol-and-popcountsum)
- [The molecule pack: `mol.pack` and `mol.off`](#the-molecule-pack-molpack-and-moloff)
- [Deletions: `deleted.roar`](#deletions-deletedroar)
- [Files a v4 shard does not have](#files-a-v4-shard-does-not-have)
- [What a reader must do](#what-a-reader-must-do)
- [A worked reading](#a-worked-reading)
- [Version 3](#version-3)

---

## The shape on disk

A **database** is a directory holding `database.json` and one directory per
shard:

```
mydb/
  database.json
  shard-0000/
    manifest.json
    mol.pack      mol.off          the molecule graphs, and block offsets
    smi.blk       smi.off          the input records, and block offsets
    smi.sum                        per-block CRC-32C of smi.blk — only when it is stored raw
    fp.col        fp.off           fingerprints, and block offsets
    fp.sum                         per-block CRC-32C of fp.col — only when it is stored raw
    popcount.col  popcount.sum     per-row fingerprint popcounts, and their per-block CRC-32C
    deleted.roar                   the deletion bitmap
  shard-0001/
    ...
```

That is every file a v4 shard contains. A build writes all of them except
`smi.sum`, which exists exactly when the manifest's `record_codec` is `"none"`,
and `fp.sum`, which exists exactly when its `fingerprint_codec` is. What
moleculo does when one of the fingerprint files is absent is in
[Fingerprints](#fingerprints-fpcol-fpoff-fpsum).

Shard directories are named `shard-%04d` by the shard id in their manifest, and
the id — not the position in a listing — is what identifies the shard.

⚠ **Find a shard by building its name from `database.json`, not by listing the
directory.** A directory present but not named in `shards` is not part of the
database — a half-finished build, a shard from another generation left behind —
and reading it in would merge a collection nobody asked for.

⚠ **The manifest's `shard` field is what the row ids are built from, and nothing
checks it against the directory name.** moleculo verifies the generation and not
this, so a `shard-0001` directory whose manifest says `"shard": 5` is served, and
its rows carry 5 in the high half. If the two disagree, say so rather than
picking quietly; the manifest is what the row ids actually follow.

⚠ **A bare shard directory is also a database of one shard.** A directory
holding `manifest.json` and the files above, with no `database.json`, is what
moleculo's own default build produces and is a valid thing to read: treat it as
a database whose shard list is that one shard and whose generation is the
shard's own. A directory holding both `manifest.json` and `database.json` is
read by moleculo as a bare shard.

Every file except the two `.json` ones begins with the same 64-byte header.

---

## `database.json`

```json
{ "format_version": 1, "generation": 0, "shards": [0, 1, 2] }
```

| field | meaning |
|---|---|
| `format_version` | schema version of **this file**, currently `1`. It is independent of the shard format version, and a reader refuses a value it does not know. |
| `generation` | the generation every shard must agree with. See [Row identity](#row-identity). |
| `shards` | shard ids expected to be present, **strictly ascending**, each below 2³¹. Every build, append and migration writes them so. |

This file is unchanged from version 3.

⚠ **A shard whose manifest names a different generation is not a member of this
database**, however much it looks like one: its row ids mean something else. A
reader either refuses it or reports it as a fault; it must not merge its rows in.

⚠ **A `shards` list that repeats an id, lists them out of ascending order, or
names an id of 2³¹ or more is refused, and the database with it** — not sorted
or deduplicated. moleculo reports it as a database that does not open, naming
the id, as it reports one whose `database.json` does not parse. A list nobody
of ours wrote is not ours to reinterpret, and before moleculo 2.0 an id listed
twice was opened twice and its rows served twice.

⚠ **A shard named here whose directory is missing is a degraded database, not a
corrupt one.** moleculo serves the rest and says which shard is absent; a count
taken from a degraded database is a floor. Say so, rather than presenting the
total as final.

⚠ **A shard that is present and refused is degraded in the same way.** A
version-3 shard, a shard of another generation, and one that fails any refusal
in this document at open are each left out, named with the reason, and the rest is
served — moleculo reports them beside the missing ones in `/dt/data`. If every
shard is missing or refused, the database is refused; if any of those shards is
version 3, the refusal names `index migrate`. A bare shard that is refused
refuses its database.

---

## `manifest.json`

One per shard, beside its files: a JSON object. The fields a reader must
honour:

| field | type | meaning |
|---|---|---|
| `format_version` | `u16` | version of the files in this directory. `4` is this document. `3` is the 1.x format — refuse it by name (see [Version 3](#version-3)). **Refuse anything else.** |
| `shard` | `u32` | this shard's id — the high half of every row id in it |
| `generation` | `u64` | see [Row identity](#row-identity) |
| `molecules` | `u64` | how many rows the shard **addresses**, deleted ones included |
| `block_molecules` | `u32` | molecules per block of **`mol.pack` only**. 256 by default |
| `record_block_molecules` | `u32` | records per block of `smi.blk`. 64 by default. **Absent or `0` means "use `block_molecules`"** |
| `record_block_bytes` | `u64` | the largest `smi.blk` block **before** compression, in bytes: the bound a reader decompresses one into. Required, non-zero and at most 2³⁰, when `record_codec` is `"zstd"` and `molecules` is not zero |
| `record_codec` | `"none"` \| `"zstd"` | codec of `smi.blk` — **of `smi.blk` only**. Under `"none"`, `smi.sum` must be beside it |
| `molecule_codec` | string | codec of `mol.pack`. `"lz4hc-9"` is the only value. **Refuse any other, and refuse its absence** |
| `perception` | string | which models produced what `mol.pack` froze at build — see below |
| `fingerprint_bits` | `u32` | fingerprint width in bits; a multiple of 64. `0` means the shard has no fingerprints |
| `fingerprint_codec` | `"none"` \| `"zstd"` | codec of `fp.col`. Absent means `"none"` |
| `fingerprint_block_molecules` | `u32` | rows per block of `fp.col`, and of the blocks `fp.sum` and `popcount.sum` checksum. 1 024 by default. **Absent or `0` means "use `block_molecules`"** |
| `builder` | string | what wrote the shard, e.g. `moleculo-index 2.0.0-rc.1`. **Informational**: it is the crate's version, which a release moves, and a reader must not check it against the format version or refuse a value it has not seen |
| `features` | array of string | additive flags; see below |

**The three block sizes are separate numbers**, because the three stores are
read differently, and a reader assumes none of them. Each must lie in 1–4 096
after the fallbacks above; a value outside it is a damaged manifest, and so is a
`fingerprint_bits` that is not a multiple of 64.

⚠ **`block_molecules` covers `mol.pack` alone.** In version 3 it also sized
`smi.blk`; a reader that carries that rule over reads every `smi.blk` block at
256 records instead of 64 and returns the wrong record for most rows, with
nothing malformed to notice.

**`perception`** names the valence model behind the stored hydrogen counts, the
ring definition with its limits, and the aromaticity model, e.g.
`valence-1; relevant-cycles-1 (8, 256); aromaticity-1`. A reader does not need
to parse it. It says how the stored hydrogens, ring counts and aromatic flags
were derived: a shard answers as it was built, and a chemistry fix reaches a
shard only when it is rebuilt.

**The feature statistics** — `screening`, `screening_dense`,
`screening_path_atoms`, `screening_vocabulary` — are counts the build takes so
that moleculo can explain an empty answer: `screening` is an array of
`{"feature", "cardinality", "exact"}` objects, `screening_dense` an array of
feature strings. ⚠ **There is no index behind them.** Version 4 writes no
`screen.idx`; the names are kept from the version that did. What a feature
string means is not specified here.

`fingerprint_radius` and `fingerprint_algorithm` say how the fingerprints were
computed (radius 2, algorithm 1 today). A reader that only checks the stored
bytes ignores them. ⚠ moleculo scores only algorithm 1: a shard whose
`fingerprint_algorithm` is anything else, or absent, is served without
similarity rather than refused.

**Which fields are required.** moleculo cannot parse — and so refuses — a
manifest without `format_version`, `shard`, `generation`, `molecules`,
`block_molecules`, `record_codec` or `builder`. Every other field takes a
default when absent: `record_block_molecules`, `fingerprint_block_molecules`
and `record_block_bytes` 0 (with the fallbacks and requirements above);
`molecule_codec` and `perception` the empty string — so an absent
`molecule_codec` is refused as not `"lz4hc-9"`; `fingerprint_codec` `"none"`;
`fingerprint_bits`, `fingerprint_radius` and `fingerprint_algorithm` 0;
`features`, `screening` and the other statistics empty. Unknown fields are
ignored.

⚠ **A reader must not fail because a field it does not use is present, absent,
or a value it has not seen** — except the ones the table says to refuse.

⚠ **`molecules` counts addressed rows, not live ones.** A row can be deleted;
the count of live rows is `molecules` minus what the deletion bitmap marks, and
moleculo's own `recordsTotal` reports the live number.

⚠ **An unknown entry in `features` is not an error.** That field exists so a
shard can gain an additive property without a version bump, and a reader that
refuses unknown feature names defeats it. The rule is the other way round for
`format_version`: unknown there means *refuse*, because a layout change is not
something a reader can shrug at.

---

## The file header

Every non-JSON file in a shard starts with **64 bytes**:

| offset | size | content |
|---|---|---|
| 0 | 8 | magic, ASCII `MOLECULO` |
| 8 | 2 | format version, `u16` little-endian — `4` |
| 10 | 2 | file kind, `u16` little-endian — table below |
| 12 | 8 | a count, `u64` little-endian. **Only `deleted.roar` uses it**; zero everywhere else |
| 20 | 4 | a CRC-32C, `u32` little-endian, **in `*.off` and `deleted.roar` only**; zero everywhere else. See below |
| 24 | 40 | reserved, written as zero |

The payload begins at byte 64, so that it starts on a 64-byte boundary.

| kind | value | file |
|---|---|---|
| molecule pack | 1 | `mol.pack` |
| block offsets | 2 | `mol.off`, `smi.off`, `fp.off` |
| record store | 3 | `smi.blk` |
| deletions | 4 | `deleted.roar` |
| — | 5 | **retired.** It was version 3's `screen.idx` and is never reused: a file carrying it is a v3 file |
| fingerprints | 6 | `fp.col` |
| popcounts | 7 | `popcount.col` |
| — | 8 | **never written.** Reserved for a label column that was dropped before it shipped |
| checksums | 9 | `smi.sum`, `fp.sum`, `popcount.sum` |

⚠ **The kind is checked, not assumed.** A file found under a name it does not
match — `smi.blk` carrying kind 1, say — is an error, not something to interpret
by filename. That check is what catches a file copied into the wrong place.
`smi.sum`, `fp.sum` and `popcount.sum` share kind 9, so which of the three a
file is, is its name's to say; nothing inside tells them apart.

**What the header check covers.** moleculo checks magic, version and kind in
every file it opens, and nothing else in a header but the CRC of a small file.
A non-zero count, CRC or reserved byte anywhere it is specified as zero is
ignored — informational, not damage. Only the small files' headers are under a
checksum; a flipped bit in the header of `mol.pack`, `smi.blk`, `fp.col`,
`popcount.col` or a `*.sum` either fails the magic, version or kind check or
changes nothing that is read. `popcount.col`'s header is checked at open like
the rest: one that is not a version-4 kind-7 header refuses the shard. (Before
moleculo 2.0 it was checked only when the popcounts were used, and a foreign
one was ignored and every popcount recomputed.)

**The small files carry their own checksum.** In `mol.off`, `smi.off`, `fp.off`
and `deleted.roar`, header bytes 20..24 hold the CRC-32C of **the whole file,
header included, with those four bytes read as zero**:

```
crc = crc32c(file[0:20] + b"\0\0\0\0" + file[24:])
```

A file whose field does not match is damaged — refuse it. A field of zero is no
exception: it is what a shard built before these checksums carries, and such a
shard must be rebuilt. ⚠ It covers the header as well as the payload because
`deleted.roar` keeps its count at bytes 12..20, and a flipped bit there changes
the live count with nothing else wrong.

---

## CRC-32C

Every CRC in this format is **CRC-32C** (Castagnoli), not the CRC-32 of zip and
Ethernet:

| parameter | value |
|---|---|
| polynomial | `0x1EDC6F41` (reflected: `0x82F63B78`) |
| initial value | `0xFFFFFFFF` |
| input and output reflected | yes |
| final XOR | `0xFFFFFFFF` |
| check: `crc32c(b"123456789")` | **`0xE3069283`** |

It is the CRC that SSE4.2's `crc32` instruction and ARMv8's `crc32c`
instructions compute, and **not** the one `zlib.crc32` computes (that one gives
`0xCBF43926` for the same nine bytes). If your implementation does not produce
the check value, every checksum in the shard will look damaged.

Where a CRC is stored, it is a `u32` little-endian.

---

## Row identity

A row id is a `u64`:

```
row_id = (shard << 32) | local
```

where `shard` is the manifest's `shard` and `local` is the row's position in its
shard, counting from zero. The ordering that falls out — shard-major, position
within shard — is the order moleculo returns rows in.

**Row `local` is the same row in every store of its shard**: molecule `local`
of `mol.pack`, record `local` of `smi.blk`, fingerprint `local` of `fp.col`,
popcount `local` of `popcount.col` and bit `local` of `deleted.roar`. Each
store is blocked by its own size, so the same row sits in a different block
number in each.

⚠ **A row id means something only inside one generation.** A rebuild that
reuses ids for different molecules must raise `generation`, and anything that
stored row ids across such a rebuild is stale. Identify a molecule outside the
index by the identifier in its record, never by its row id.

**The `arthor.index` column on the HTTP wire is this row id**, the full
`(shard << 32) | local`, so it is unique within one answer and within one
generation. In shard 0 it equals the row's position. Arthor
itself numbers rows per index part, so its own column repeats; ours does not.
⚠ It is still a row id, valid for one generation only: match a row you got
from the API to a row in the index across a rebuild by its **identifier**.

⚠ **Bit 63 is reserved and is zero in every shard this format writes.** It is
kept for a future id that is *derived* rather than stored, so that a reader can
tell the two apart; treat a row id with the high bit set as one you do not
understand, and shard ids as bounded by 2³¹ rather than 2³². **A shard whose
`shard` is 2³¹ or more is refused**, rather than its rows skipped: moleculo
refuses it at open, as damage — left out and named, like any shard refused at
open — refuses a `database.json` that names one, and `index build --shard`
refuses to write one.

---

## The block store

`smi.blk` + `smi.off` and `fp.col` + `fp.off` are the same structure: a
**payload** of variable-length records and a side file of **block offsets**.
`mol.pack` + `mol.off` share the offsets file and nothing else — its blocks are
described in [their own section](#the-molecule-pack-molpack-and-moloff).

**In the payload**, each record is written as its length, then its bytes:

- the length is an unsigned **LEB128** varint: seven bits of the value per byte,
  least-significant group first, with the high bit `0x80` set on every byte but
  the last;
- then exactly that many bytes of the record.

Records are grouped into **blocks** of the store's block size — `smi.blk` by
`record_block_molecules`, `fp.col` by `fingerprint_block_molecules` (the last
block may be short). ⚠ **The block, not the record, is the compression unit.**
Under codec `"zstd"`, the whole block — length prefixes and all — is one zstd
frame ([zstd blocks](#zstd-blocks)); under `"none"` the block's bytes are
written as they are. A record never spans two blocks.

**In the offsets file**, after its 64-byte header, there is one `u64`
little-endian per block, and then **one more**:

- entry `b` is where block `b` starts, counted from the **first payload byte
  after the payload's header** — not from the start of the file;
- the final entry is where the last block ends, which gives the last block a
  length without consulting the file size.

So the stored bytes of block `b` are `[offsets[b], offsets[b + 1])` and the
number of blocks is `ceil(molecules / block size)`. A build writes exactly
`blocks + 1` entries, the first `0` and the last the payload's length, with
no byte before the first block or after the last.

What moleculo holds each of `mol.off`, `smi.off` and `fp.off` to, beyond its
CRC — one rule for all three:

| | |
|---|---|
| entries | exactly `blocks + 1` — refused otherwise, a missing trailing entry and an entry past it included |
| last entry ≠ payload length | refused |
| first entry ≠ 0 | not checked: bytes before it are ignored |
| an entry running backwards, or past the payload | damage to that block, found when it is read |

The CRC is what makes the one leniency harmless: a file moleculo wrote cannot
reach it without failing its check first. A reader may be stricter, and
should be no laxer. ⚠ Before moleculo 2.0, `smi.off` and `fp.off` were held
only to at least `blocks` entries: a missing trailing entry ran the last block
to the end of the payload, and entries past `blocks + 1` were ignored.

**To read row `local`:**

1. `block = local / size`, `within = local % size`;
2. take the payload bytes `[offsets[block], offsets[block + 1])`;
3. verify them, and decompress them if the codec says so;
4. frame **the whole block**, even to read one record: from its start, read
   every record it holds — each a LEB128 length and that many bytes — and check
   that there are exactly `min(size, molecules - block * size)` of them and no
   byte after the last (see the corners below). moleculo does this on every
   read, a single record included; a reader that stops at the record it wants
   is laxer than the engine;
5. the record is the one at position `within`.

Sequential reading never needs the offsets at all: decode block by block.

**Corners, all of which occur:**

- a block holds exactly `min(size, molecules - block * size)` records — no more,
  no fewer, and no byte after the last. **Check that count.** Every block of a
  version-4 store is under a checksum — its zstd frame's, or its `*.sum` entry
  when raw — so a damaged length prefix is refused by that first; the count is
  what catches a block the checksum cannot, such as one a faulty writer framed
  wrongly before checksumming it. moleculo checks it on every read of a block,
  including a read of one record, which frames the whole block rather than
  stopping at the record it wants;
- a shard of zero molecules has no blocks and an offsets file holding the single
  trailing entry;
- a LEB128 length longer than ten bytes, or one whose groups shift past 64 bits,
  is corrupt rather than large.

---

## zstd blocks

When a block store's codec is `"zstd"` — `smi.blk` under `record_codec`,
`fp.col` under `fingerprint_codec` — each block is **exactly one zstd frame**
([RFC 8878](https://www.rfc-editor.org/rfc/rfc8878)) occupying the whole extent
its offsets give it: no dictionary, no skippable frame, nothing before or after
it. Decompressed, it is the block of LEB128-framed records above. **Parse the
frame header as RFC 8878 §3.1.1.1 defines it** — `Frame_Header_Descriptor`,
then a `Window_Descriptor` unless `Single_Segment_flag` is set, then a
`Dictionary_ID` field of the width `Dictionary_ID_flag` gives, then
`Frame_Content_Size` — not from the description of today's frames below.

**Every frame carries a content checksum**, and a reader must hold it to that:

- a frame starts with zstd's magic, bytes `28 B5 2F FD`. A block that does not is
  damaged;
- byte 4 is the `Frame_Header_Descriptor`. **Its bit 2, `Content_Checksum_flag`,
  is set on every frame.** A frame without it must be refused — never decoded
  unchecked: zstd verifies only what a frame says it carries, so one flipped
  header bit would otherwise switch the check off. A frame without the flag is
  also what a shard built before these checksums holds, and such a shard must be
  rebuilt;
- the frame's last four bytes are the low 32 bits of **XXH64, seed 0, over the
  decompressed block**, little-endian, as RFC 8878 §3.1.1 defines. A conforming
  zstd library checks it as it decodes; a mismatch is damage to that block;
- **no dictionary**: a frame naming a non-zero dictionary ID is damage. moleculo
  decodes with no dictionary loaded, and libzstd refuses such a frame. A
  `Dictionary_ID` field that holds 0 means *no dictionary*, and moleculo reads
  that frame (both measured, on a `smi.blk` frame edited and its offsets
  resealed);
- **the frame ends at its extent**: a reader must check that the frame — header,
  blocks and checksum — ends exactly where the extent does, because a stock
  decoder decodes concatenated frames and skips skippable ones. moleculo walks
  the frame's block headers to its end on every read of the block, before
  decoding it, and a skippable frame, a second frame — even one of no bytes —
  or any byte after the first frame is damage to that block. (Before moleculo
  2.0 it handed the extent to libzstd as it was, and a skippable frame or an
  empty second frame after the block's read as intact.) No build writes one.

**Each store has a bound on how large a decoded block may be**, and a frame
claiming more is damage, refused before anything is allocated for it:

| store | bound |
|---|---|
| `fp.col` | `fingerprint_block_molecules × (L + W)`, where `W = fingerprint_bits / 8` and `L` is the byte length of `W` as a LEB128 varint — 66 560 at 512 bits. Capped at 2³⁰. **The full-block figure for every block**, the short last one included: it decodes to less, and the framing holds it to its own count of records |
| `smi.blk` | the manifest's `record_block_bytes` |

**The frames written today**: `Single_Segment_flag` set, so no
`Window_Descriptor`; `Dictionary_ID_flag` 0, so no `Dictionary_ID` field; and so
`Frame_Content_Size` straight after the descriptor, its width by
`Frame_Content_Size_flag` — one byte for a block under 256 bytes (descriptor
`0x24`, a short last block); two bytes holding the size minus 256, up to
65 791 (`0x64` — every `smi.blk` block of the test collections); four bytes
beyond (`0xA4` — `fp.col`'s 66 560-byte blocks at 512 bits). ⚠ **The size field
is not a format requirement.** Read it
to size your buffer and to refuse a claim above the bound; a frame without it
is decoded into the bound. Decode once, into that size: a buffer grown and
retried on failure is how moleculo once looped for ever on a header claiming
zero bytes.

The compression level (3) is not part of the format.

---

## The record store: `smi.blk`, `smi.off`, `smi.sum`

A record is **the input line the molecule was built from**, with trailing
whitespace removed, as bytes. moleculo's inputs are SMILES files, so in practice
a record is:

```
CC(=O)Oc1ccccc1C(=O)O<TAB>CHEMBL25
```

The text before the first tab is the SMILES. **The identifier is the second
tab-separated field** — the text after the first tab, up to the next tab or the
end of the record — and it is what `index delete` names a row by and what search
results return as `Identifier`. It may be empty. A record with no tab is its
own identifier. Fields after the second stay in the record and are returned
nowhere else. Whitespace inside a record is as the input had it; only the input
line's trailing whitespace was removed.

`smi.blk` is a block store of `record_block_molecules` records per block under
`record_codec`. **How it is checked depends on its codec:**

- under `"zstd"` — the default — by the frame checksum, and there is no
  `smi.sum`. One found beside a zstd `smi.blk` is ignored, as an `fp.sum`
  beside a zstd `fp.col` is;
- under `"none"`, by **`smi.sum`**: a 64-byte header of kind 9, then one
  CRC-32C `u32` little-endian per `smi.blk` block, in block order, over that
  block's **stored bytes as `smi.off` places them**. It holds exactly
  `ceil(molecules / record_block_molecules)` entries — none for a shard of zero
  molecules. A raw `smi.blk` without `smi.sum` beside it is refused, never read
  unchecked; it is also what a v4 shard built before `smi.sum` holds, and such a
  shard must be rebuilt.

⚠ **Verify the block on every read of it, a single record included.** A hit,
an identity candidate and a result page each read one record, and that read
skips to it by the length prefixes — so a damaged prefix there shifts the
record rather than failing: before `smi.sum`, one length byte lowered by one in
a raw `smi.blk` rendered the hit `ZINC000245224194` as `ZINC00024522419`, with
no error, and a flipped byte inside a record's text read as a different record
everywhere. moleculo checks the block's `smi.sum` entry before it reads any
record in it, on the single-record read and the full walk alike; a mismatch
names `smi.blk`, the block and `smi.sum`, since either file may be the damaged
one. `record_block_bytes` does not apply to a raw `smi.blk`: moleculo does not
hold raw blocks to it.

⚠ **There is one record per addressed row, deleted rows included**, and records
are in row order. Row `local` of shard `s` is record `local` of that shard's
record store, and its row id is `(s << 32) | local`.

⚠ **A record is valid UTF-8, and that is a property of the builder rather than
of the format.** The ingest reads its input as UTF-8 lines and fails the build on
a byte sequence that is not, so no shard can hold a record that is not — decode
strictly and treat a failure as a corrupt file rather than as text to salvage.

⚠ **A line the builder rejected is not a row at all.** A line whose SMILES does
not parse, or whose molecule `mol.pack` cannot hold, is skipped, so input line
numbers and row ids do not correspond; do not compute one from the other. The
one exception is a shard converted from version 3, where a refused row keeps
its place as a deleted row with an empty molecule — see
[Version 3](#version-3).

---

## Fingerprints: `fp.col`, `fp.off`, `fp.sum`

Every build writes fingerprints: the builder refuses a width of 0, so a built
shard's `fingerprint_bits` is never 0. What moleculo does with what it finds:

- `fp.col`, `fp.off` and `popcount.col` are each opened if present;
- whenever `fp.col` is present and `fingerprint_codec` is `"none"` — as it is
  when the field is absent — `fp.sum` must be beside it, **whatever
  `fingerprint_bits` says**, and the shard is refused without it; whenever
  `popcount.col` is present, `popcount.sum` must be;
- an `fp.sum` beside a zstd `fp.col` is ignored;
- ⚠ when `fingerprint_bits` is not 0 and `fingerprint_algorithm` is 1, the
  column must open and cover every row, or the shard is refused — a damaged
  shard, not a shard without similarity. Under any other algorithm moleculo does
  not use the column and does not hold it to that.

`fp.col` is a block store of `fingerprint_block_molecules` records per block,
under `fingerprint_codec`. **Every record is exactly `fingerprint_bits / 8`
bytes** (its LEB128 length is therefore one byte at 512 bits): the
fingerprint's 64-bit words in order, each little-endian. A record of any other
length is damage, and the rows after it are misaligned. Which feature sets
which bit is the fingerprint algorithm's and is not specified here.

**How it is checked depends on its codec:**

- under `"zstd"` — the default — by the frame checksum, and there is no `fp.sum`;
- under `"none"`, by **`fp.sum`**: a 64-byte header of kind 9, then one CRC-32C
  `u32` little-endian per `fp.col` block, in block order, over that block's
  **stored bytes as `fp.off` places them**. It holds exactly one per block. A raw
  `fp.col` without `fp.sum` beside it is refused, never read unchecked.

---

## Popcounts: `popcount.col` and `popcount.sum`

`popcount.col` is a 64-byte header of kind 7 and then one `u16` little-endian
per row, `2 × molecules` bytes: the number of set bits in that row's fingerprint
(saturating at 65 535, which no width under 65 536 bits reaches). It is not a
block store and has no offsets file. moleculo refuses a column shorter than the
rows it reads and ignores bytes past them.

`popcount.sum` exists whenever `popcount.col` does — a `popcount.col` without it
is refused. It is a 64-byte header of kind 9, then one CRC-32C `u32`
little-endian per **fingerprint block**: entry `b` covers the popcount bytes of
rows `b × F` up to `min((b + 1) × F, molecules)`, where
`F = fingerprint_block_molecules` — that is, bytes
`[2·b·F, 2·min((b+1)·F, molecules))` of the payload. It holds exactly
`ceil(molecules / F)` entries.

⚠ **The blocks are `fp.col`'s, not `mol.pack`'s**: the two columns are read
together, by the similarity scan, a fingerprint block at a time.

---

## The molecule pack: `mol.pack` and `mol.off`

`mol.pack` holds every row's molecule graph, `block_molecules` molecules per
block. `mol.off` is an offsets file exactly as in [the block
store](#the-block-store), giving each block's stored extent; it holds exactly
`blocks + 1` entries and its last entry is the payload's length.

### A stored block

| offset | size | content |
|---|---|---|
| 0 | 4 | `raw_len`, `u32` little-endian: the block's length once decompressed |
| 4 | 4 | CRC-32C of the **compressed bytes** — byte 8 to the end of the block's extent — `u32` little-endian |
| 8 | … | one **LZ4 block** (not an LZ4 frame) that decompresses to exactly `raw_len` bytes |

To open a block: check the CRC over the compressed bytes **first**, then
decompress. A block whose CRC does not match, that will not decompress, or that
decompresses to any length but `raw_len`, is damaged. Since no LZ4 block decodes
to more than 255 bytes per compressed byte, a `raw_len` above
`255 × compressed length` is damage too, refused before allocating.

The manifest's `molecule_codec` names this: `"lz4hc-9"` is LZ4 HC at level 9.
The level is how the bytes were chosen, not how they are read — any LZ4 block
decoder reads them.

**The LZ4 block format**, in brief (the reference is `lz4_Block_format.md` in
the LZ4 project). A block is a sequence of *sequences*, each:

1. a **token** byte: the high four bits are a literal length, the low four a
   match length;
2. if the literal length is 15, more bytes follow, each added to it, until one
   is below 255;
3. that many **literal** bytes, copied to the output;
4. if the input ends here, the block is complete — the last sequence has no
   match;
5. otherwise a **match offset**, `u16` little-endian, at least 1 and at most the
   number of bytes output so far;
6. the match length is the token's low four bits plus 4; if those bits are 15,
   more bytes follow and are added as in step 2;
7. copy that many bytes from `offset` bytes back in the output, one at a time —
   a match may overlap the bytes it is producing.

What moleculo's decoder requires: the last sequence is literals only — a block
that ends after a match offset or a match is truncated; every offset is at least
1 and at most the bytes output so far; and the output never passes `raw_len`
and ends exactly at it. **The format's end rules** — the last five bytes are
literals, no match starts within twelve bytes of the end — are how a compressor
writes, and moleculo's compressor keeps them, but its decoder does not enforce
them, and a reader must not require them.

### A raw block

A decompressed block is **column-major**: a directory, then seven sections,
each holding one field of every molecule of the block in turn — molecule 0's
entries, then molecule 1's, and so on. Section 6 runs to the end of the block,
so the directory gives the lengths of the first six:

```
directory     6 × LEB128: the byte lengths of sections 0..5
section 0     counts
section 1     atoms
section 2     rings
section 3     exceptions
section 4     edge flags
section 5     deltas
section 6     connectivity           (to the end of the block)
```

Section 0 begins right after the directory, and each section right after the
one before. A directory whose lengths run past the end of the block is damage.

Every varint here is an unsigned LEB128 of at most ten bytes, as in the block
store. `zigzag` maps a signed value to an unsigned one, `0, -1, 1, -2, 2…` to
`0, 1, 2, 3, 4…`: `z = (v << 1) ^ (v >> 63)`, and back, `v = (z >> 1) ^ -(z & 1)`.

What each section holds **per molecule**, for a molecule of `n` atoms and `e`
bonds:

| section | per molecule |
|---|---|
| 0 counts | varint `n`, varint `e` |
| 1 atoms | `n` × two bytes, `[element, packed]`, in atom order |
| 2 rings | for each atom whose `packed` bit 7 is set, in atom order: `u8 R`, varint `k`, then `k` × varint ring size |
| 3 exceptions | varint `c`, then `c` entries in ascending atom order, each: varint atom index, *`u8` hydrogens — present only if that atom's `packed` hydrogens are 7*, `i8` charge, varint isotope + 1, varint map class |
| 4 edge flags | `e` bytes, one per bond, in bond order |
| 5 deltas | for each bond whose flags lack bit 7, in bond order: varint `zigzag(a − previous a)`, varint `zigzag(b − a)` |
| 6 connectivity | `n` bytes, one per atom, in atom order |

So reading molecule `i` of a block means reading molecules `0..i-1` in every
section first: the counts, rings, exceptions and deltas are of variable width.
⚠ **A single row is reached by walking its block.**

**`element`:**

| bits | meaning |
|---|---|
| 0–6 | atomic number, 0–118; 0 is the wildcard atom `*`. Anything above 118 is damage |
| 7 | the atom has an entry in the exceptions section |

**`packed`:**

| bits | meaning |
|---|---|
| 0–2 | attached hydrogens, 0–6; **7 means "seven or more: read the count from the atom's exception"** |
| 3 | aromatic |
| 4 | chirality `@@` (clockwise) |
| 5 | chirality `@` (anticlockwise) — never both 4 and 5 |
| 6 | written in brackets in the input |
| 7 | on a ring: the atom has an entry in the rings section |

A build gives an atom an exception exactly when its charge is not zero, it
carries an isotope or a map class, or it has seven or more hydrogens. An
exception's fields replace the atom's defaults, which are charge 0, no isotope,
map class 0. The charge is an `i8`, any value. The isotope is stored **plus
one**, so `0` is *no isotope* and `1` is the isotope **0** (`[0CH4]`, which a
query tells apart from no isotope); **the isotope** fits a `u16`, so the stored
value is at most 65 536. The map class fits a `u16`. An exception whose fields
are all defaults is never written by a build, and moleculo does not refuse
one.

**Hydrogens are stored, filled at build**: the count is every hydrogen attached
to the atom that is not itself an atom of the graph — implicit or written in
brackets — as the valence model `perception` names computed it. A hydrogen
written as its own atom (`[H][H]`, `[2H]C`) is an atom with element 1.

**A ring entry**: `R` is how many rings the atom lies on, saturating at 255, and
the `k` sizes are the distinct sizes of those rings, ascending. "Rings" are the
ones `perception`'s ring definition counts — what a query's `R<n>` and `r<n>`
read. The sizes are varints, so a ring of 300 atoms does not saturate. A build
writes `R ≥ 1`, `k ≥ 1`, `k ≤ R` and sizes strictly ascending. moleculo
refuses, as damage to the block, an `R` or `k` of 0, sizes that do not strictly
ascend, and a size that does not fit a `u16`, on every read that walks the
entries — a whole-molecule decode and a structure scan alike. ⚠ It does not
check `k` against `R`; a reader may.

**Edge flags**, one byte per bond:

| bits | meaning |
|---|---|
| 0–1 | order: 0 single, 1 double, 2 triple, 3 quadruple |
| 2 | aromatic |
| 3 | in a ring |
| 4 | direction up, `/` |
| 5 | direction down, `\` — never both 4 and 5 |
| 6 | written explicitly as single, `-` |
| 7 | **chain step**: this bond's endpoints are not in the deltas section |

There is no aromatic bond *order*: aromaticity is bit 2, and the order is the
Kekulé one. A molecule the builder could not express that way is rejected at
build rather than stored differently.

**Bond endpoints.** Walk a molecule's bonds in order with `previous_a`, which
starts at **0** for each molecule and becomes each bond's `a` after it:

- a bond with the chain step bit has `a = previous_a + 1` and `b = previous_a + 2`;
- any other bond reads two deltas: `a = previous_a + unzigzag(first)`,
  `b = a + unzigzag(second)`.

Both endpoints must name distinct atoms of the molecule, `0 ≤ a, b < n` and
`a ≠ b`. `a > b` is legal — the second delta is then negative. Two bonds
joining the same pair are never written by a build, and moleculo does not
refuse them. ⚠ Because `previous_a` starts at 0, a molecule's first bond is a
chain step only if it joins atoms 1 and 2; a first bond between atoms 0 and 1
is not, and is stored as the two varint bytes `00 02` — the zigzag codes of a
delta of 0 and of +1.

**Connectivity**, one byte per atom:

| bits | meaning |
|---|---|
| 0–3 | how many bonds of this molecule end at the atom — its explicit degree, every neighbour that is an atom of the graph, hydrogen atoms included |
| 4–7 | how many of those bonds have the ring flag set |

⚠ **Each nibble saturates at 15, and 15 means "count it from the edges"**, not
fifteen: an atom of degree 17 stores 15, and a reader that takes the nibble at
its word answers `D15` for it. Both nibbles are exactly what counting the
molecule's own edge list gives, below 15.

⚠ **Neighbour order is part of the format.** A chirality mark is a parity over
the atom's neighbours **in ascending bond index**. An adjacency built from the
edge list must list each atom's neighbours in that order, or every stereocentre
reads mirrored.

### The integrity rule

A block holds exactly `min(block_molecules, molecules − block × block_molecules)`
molecules. Decode that many, then **every section must be consumed exactly** —
no section may end before its molecules do, and none may hold a byte past its
last molecule. ⚠ This is what catches a writer that wrote the wrong extents, or
counts that do not match the sections: the CRC says the bytes are the ones that
were written, not that what was written is consistent.

Within that, moleculo's reader also refuses, as damage to the block: an element
above 118; both chirality bits; hydrogens of 7 on an atom without the exception
bit; an exception count that is not the number of atoms with the exception bit,
an exception index out of order, out of range or naming an atom without the bit;
an exception hydrogen count below 7; an isotope, map class or ring size that
does not fit a `u16`; a ring entry whose `R` or `k` is 0 or whose sizes do not
strictly ascend; both direction bits on a bond; and a bond endpoint outside
the molecule or equal to the other. It does not check a ring entry's `k`
against its `R`, nor the entries' agreement with the edges, nor connectivity
against the edges,
nor repeated bonds; a build writes them consistent, and a reader may check
them.

---

## Deletions: `deleted.roar`

A header and then a plain **bitmap**, one bit per addressed row:

- row `local` is deleted when bit `local % 8` of payload byte `local / 8` is set
  (bit 0 is the least significant);
- the header's count field (bytes 12..20) says how many rows are marked, so a
  live count costs an open rather than a popcount;
- the header's CRC field (bytes 20..24) covers the whole file, as for every small
  file — [the file header](#the-file-header);
- when the count is zero the payload may be empty, and is, in a shard nothing
  was deleted from. **When the count is not zero the payload must cover every
  row** — at least `ceil(molecules / 8)` bytes — and a shorter one is a truncated
  file, refused: reading the missing rows as live would answer from rows
  somebody deleted;
- only bits below `molecules` mean anything. A payload longer than
  `ceil(molecules / 8)`, or a bit set at or past `molecules`, addresses a row the
  shard does not have: ignore it rather than refusing — and never let it reach a
  count.

⚠ **The bitmap is the authority and the header count is a cached popcount of
it.** They are written together, and since the CRC covers both, they cannot
disagree in a file that passes its check unless something wrote it other than
moleculo. If they do, trust the bits, and say that the two disagree. It matters
because moleculo's own reader takes the count from the header and the liveness
from the bits, so an inconsistent file makes it report a `deleted` that its own
answers do not match.

⚠ **The name promises a format it does not use.** `.roar` was chosen when a
roaring bitmap was planned; what is written is a dense bitmap, and the file's own
header is what tells you what it is. Do not infer the encoding from the
extension.

⚠ **Every shard has this file, and in most shards it is a bare header** — that
is the normal state, and it reads as *nothing is deleted*.

⚠ **A reader that ignores this file reports rows that no longer exist.** The
deletion is logical: the record, the molecule and the fingerprint all stay on
disk until the next rebuild, so a tool that walks the stores without consulting
the bitmap will hand back deleted molecules with a straight face.

---

## Files a v4 shard does not have

- **`screen.idx`.** Version 3's screening index is gone, and file kind 5 with
  it. The manifest still carries the feature statistics it was planned from.
- **A label column.** Planned for version 4 as kind 8, dropped before anything
  wrote one.

A file of either kind in a shard is not part of version 4.

---

## What a reader must do

1. **Refuse an unknown `format_version`**, in a file header or in a manifest,
   naming the version it found and the one it knows. Version 3 is refused *by
   name*, as the 1.x format. Refusing loudly is the whole reason the field
   exists: a v3 shard read with v4 rules returns molecules that are noise
   wearing a valid shape.
2. **Refuse the wrong magic or the wrong kind**, for the same reason.
3. **Refuse a `molecule_codec` other than `"lz4hc-9"`**, a codec name it does not
   know, and block sizes or a fingerprint width outside what the manifest
   section allows.
4. **Verify every checksum, before using what it covers**: the header CRC of
   every `*.off` and of `deleted.roar`; `mol.pack`'s CRC over each compressed
   block; `smi.sum` over each raw `smi.blk` block; `fp.sum` over each raw
   `fp.col` block; `popcount.sum` over each block of popcounts; and the content
   checksum of every zstd frame — refusing a frame that does not carry one. A
   missing `smi.sum` (raw `smi.blk`), `fp.sum` (raw `fp.col`) or `popcount.sum`
   is a refusal, not a reason to skip the check. Verify a block each time it is
   read, one record included — not only on a pass over the whole store. ⚠ **A
   reader that cannot verify a checksum must say so where its answer is**, not
   report the file as checked.
5. **Refuse a truncated or inconsistent file** — anything shorter than 64 bytes,
   an offsets file of anything but `blocks + 1` entries ending at the payload's
   length ([the block store](#the-block-store)), a block
   range outside the payload, a record whose length runs past the end of its
   block, a block holding the wrong number of records, a `mol.pack` block not
   consumed exactly, a zstd frame claiming more than its store's bound or not
   ending at its extent.
6. **Accept unknown `features` entries and unknown manifest fields.** They are
   additive by design.
7. **Consult `deleted.roar`** before reporting a row.
8. **Check the generation** of every shard against `database.json`, and refuse
   a `shards` list that is not strictly ascending and a shard id of 2³¹ or more.
9. **Report a degraded database in its own output.** When a shard named by
   `database.json` is absent or refused, the numbers are a floor and any digest
   covers only the shards that were read. Name each such shard and why. Say it
   where the numbers are, not only on stderr: a total piped into something else
   reads as final.
10. ⚠ **Never write into a shard.** An index is derived; if it is wrong, rebuild
    it from its input.

---

## A worked reading

To list every live row of a database as `row_id<TAB>atoms<TAB>bonds<TAB>record`:

```
read database.json           -> generation, shard ids   (or: one bare shard)
refuse a shard id listed twice, out of ascending order, or >= 2^31
for each shard id, ascending:
    read manifest.json       -> format_version, shard, generation, molecules,
                                the three block sizes, record_codec,
                                record_block_bytes, molecule_codec,
                                fingerprint_bits, fingerprint_codec
    refuse if format_version != 4, molecule_codec != "lz4hc-9",
           generation != database generation, shard >= 2^31,
           a block size outside 1..4096, or fingerprint_bits % 64 != 0
    open every file the shard has -> check magic, version 4, kind
    check the header CRC of mol.off, smi.off, fp.off and deleted.roar
    for each mol.pack block b:
        check its CRC, LZ4-decompress it to raw_len
        decode its min(block_molecules, molecules - b * block_molecules)
        molecules, holding every section to exact consumption
    for each smi.blk block (record_block_molecules records each):
        zstd: refuse a frame without its checksum flag, naming a
              dictionary, claiming more than record_block_bytes, or not
              ending at its extent; decompress, and check its content checksum
        raw:  check it against smi.sum (which must exist)
        split its records; check the count and that nothing trails them
    for each fp.col block (fingerprint_block_molecules rows each):
        zstd: as smi.blk, bounded by the fp.col figure
        raw:  check it against fp.sum (which must exist)
        split its rows; each must be fingerprint_bits / 8 bytes
    for each fingerprint block of popcount.col:
        check it against popcount.sum (which must exist)
    for local in 0 .. molecules:
        skip if the bitmap marks it
        emit ((shard << 32) | local), n, e of molecule local, record local
```

The count of what this emits is the database's live row count, and it is the
number moleculo reports as `recordsTotal`.

⚠ **If you are checking a reader against moleculo**, `moleculo index inspect
<shard>` prints `molecules`, `live` and `deleted` for one shard, and a served
database reports `recordsTotal` — three numbers that must agree with what a
reader of this document computes. Two shards built from the same input by the
same builder version are byte-identical, which is what makes such a comparison
meaningful at all. `tools/read-shard.py` in the repository is a reader written
from this document; it verifies every checksum above and prints each live
molecule's atom and bond counts.

---

## Version 3

**Version 3 is the format of every 1.x release, from 0.8.0 to the last, and it
is frozen.** moleculo 2.0 does not read it: a v3 shard is refused **by name** —
*format version 3 is the 1.x index format, which this build does not read; it
reads version 4 — convert the shard with \`moleculo index migrate\`* — rather than as
an unknown version, because the way forward is a conversion, not a guess.
`index inspect` prints the same words after `REFUSED`.

**`moleculo index migrate <database-dir>` converts a v3 database to version 4
in place, shard by shard, and needs no input file.** It reads only what this
section calls the container — the manifest's container fields, `smi.blk` with
`smi.off`, and `deleted.roar` — and never opens a v3 `mol.pack`, `fp.col` or
`screen.idx`: every row's record is parsed and perceived afresh by the version-4
builder. What it keeps:

- **every row id** — the shard id, the generation and each row's position;
- **every record**, byte for byte;
- **every deletion**;
- the shard's record and fingerprint codecs and fingerprint width.

So a migrated shard is the shard a fresh version-4 build of the same lines
writes, with its deletions. ⚠ **A row the version-4 builder refuses is deleted,
not dropped**: it keeps its id and its record, holds a molecule of zero atoms
and zero bonds in `mol.pack`, and is marked in `deleted.roar`, so no other
row's id moves and the generation does not change. Such rows are listed in
`<database-dir>.migrate-refused.tsv`. Each shard is built beside the old one
and swapped in when sealed, so an interrupted run is resumed by running it
again. A database served through other servers (`parts.json`) is refused:
migrate each part.

The full text of this document as it stood for version 3 is in the repository's
history: `dist/INDEX-FORMAT.md` at commit `a4e0279`, the last before this
rewrite.
**It is summarised here rather than only pointed to** because this file ships in
the release archive, which carries no history, and a v3 index is exactly the
thing a 2.0 user may hold. At the depth the version-3 text specified —
container and records — version 3 differs from everything above in these ways,
and in no others:

- **Header**: version `3`. Bytes 20..64 are all zero: **no file carries a CRC**,
  and there are no `*.sum` files.
- **Kinds**: 1–7 as above, with **5 = `screen.idx`**, the screening index. There
  is no kind 9.
- **Files**: `mol.pack`, `mol.off`, `smi.blk`, `smi.off`, `fp.col`, `fp.off`,
  `popcount.col`, `screen.idx`, `deleted.roar` and `manifest.json`.
- **Manifest**: `format_version` is `3`. **`block_molecules` sizes both
  `mol.pack` and `smi.blk`** (64 in every 1.x build), and there is no
  `record_block_molecules`, `record_block_bytes`, `molecule_codec` or
  `perception`. **`record_codec` is the codec of both `smi.blk` and `mol.pack`.**
  The `screening*` fields describe a `screen.idx` that exists.
- **`mol.pack`** is a block store like `smi.blk` — LEB128-framed records,
  `block_molecules` per block, compressed per block under `record_codec` — and
  **its record encoding was never specified**. Read a v3 shard's molecules from
  its `smi.blk` records.
- **zstd frames carry no content checksum**, so nothing but the record framing
  guards them: measured on frames written the same way, about half of all
  single-bit flips inside a frame decode to a different block that still frames
  correctly. ⚠ The version-3 text said the frames carry no *content size*
  either; a 1.0.0 build's frames do record one, with `Single_Segment_flag` set:
  descriptor `0x20` (a one-byte size) on a block under 256 bytes, `0x60` (two
  bytes) on the ordinary `smi.blk` and `mol.pack` frame and a short last
  `fp.col` block, `0xA0` (four bytes) on a full `fp.col` block. Do not rely on it either way — decode a v3 frame with a decoder that
  grows its buffer, and hold it to a bound of your own.
- **`deleted.roar`**: as above, without the CRC. The version-3 text read a
  payload shorter than `ceil(molecules / 8)` as leaving its missing rows live.
- **Offsets**: a v3 `smi.off` or `fp.off` may hold `blocks` entries and no
  trailing one — a store written before that entry existed — and its last block
  then runs to the end of the payload. `index migrate` reads it so.

Row identity, `database.json`, the rest of the block store's offsets, the
record store's records and the bitmap's bits are as described above.
