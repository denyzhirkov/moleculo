# The moleculo index format, version 3

This describes the **container**: how a shard's files are framed, how records
are stored and addressed, how row identity works, and which rows are live. It is
enough to write a program that lists a database's live molecules with their row
ids and their original input records, without moleculo.

It is deliberately not everything. The encodings of the packed molecule graph,
the fingerprints and the screening postings are **named and reserved** below,
not described — those are the search engine, and reading them is not what an
operator needs from an outside tool.

**Version 3 is frozen** until the next major release of moleculo (decision of
2026-09-16). A future version will change the header's version field, and a
reader written against this document must refuse what it does not recognise
rather than guess — see [What a reader must do](#what-a-reader-must-do).

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
- [Row identity](#row-identity)
- [The block store](#the-block-store)
- [The record store: `smi.blk` and `smi.off`](#the-record-store-smiblk-and-smioff)
- [Deletions: `deleted.roar`](#deletions-deletedroar)
- [Reserved files](#reserved-files)
- [What a reader must do](#what-a-reader-must-do)
- [A worked reading](#a-worked-reading)

---

## The shape on disk

A **database** is a directory holding `database.json` and one directory per
shard:

```
mydb/
  database.json
  shard-0000/
    manifest.json
    mol.pack      mol.off          the packed graphs, and block offsets
    smi.blk       smi.off          the input records, and block offsets
    fp.col        fp.off           fingerprints, and block offsets
    popcount.col                   per-row fingerprint popcounts
    screen.idx                     the screening index
    deleted.roar                   the deletion bitmap
  shard-0001/
    ...
```

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
shard's own.

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
| `shards` | shard ids expected to be present, ascending. |

⚠ **A shard whose manifest names a different generation is not a member of this
database**, however much it looks like one: its row ids mean something else. A
reader either refuses it or reports it as a fault; it must not merge its rows in.

⚠ **A shard named here whose directory is missing is a degraded database, not a
corrupt one.** moleculo serves the rest and says which shard is absent; a count
taken from a degraded database is a floor. Say so, rather than presenting the
total as final.

---

## `manifest.json`

One per shard, beside its files. The fields a container-level reader must
honour:

| field | type | meaning |
|---|---|---|
| `format_version` | `u16` | version of the files in this directory. `3` is this document. **Refuse anything else.** |
| `shard` | `u32` | this shard's id — the high half of every row id in it |
| `generation` | `u64` | see [Row identity](#row-identity) |
| `molecules` | `u64` | how many rows the shard **addresses**, deleted ones included |
| `block_molecules` | `u32` | records per block in `mol.pack` and `smi.blk` |
| `record_codec` | `"none"` \| `"zstd"` | codec of `smi.blk` |
| `builder` | string | what wrote the shard, e.g. `moleculo-index 0.23.0` |
| `features` | array of string | additive flags; see below |

Other fields describe the parts this document reserves — `fingerprint_bits`,
`fingerprint_codec`, `fingerprint_block_molecules`, `fingerprint_radius`,
`fingerprint_algorithm`, `screening`, `screening_dense`, `screening_codec`,
`screening_compress_min`, `screening_path_atoms`, `screening_vocabulary`. A
container-level reader ignores them; it must not fail because one is present,
absent, or a value it has not seen.

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
| 8 | 2 | format version, `u16` little-endian — `3` |
| 10 | 2 | file kind, `u16` little-endian — table below |
| 12 | 8 | a count, `u64` little-endian. **Only the deletion file uses it**; zero everywhere else |
| 20 | 44 | reserved, written as zero |

The payload begins at byte 64. The header is 64 rather than the 20 bytes the
fields need so that the payload starts on a 64-byte boundary.

| kind | value | file |
|---|---|---|
| molecule pack | 1 | `mol.pack` |
| block offsets | 2 | `mol.off`, `smi.off`, `fp.off` |
| record store | 3 | `smi.blk` |
| deletions | 4 | `deleted.roar` |
| screening index | 5 | `screen.idx` |
| fingerprints | 6 | `fp.col` |
| popcounts | 7 | `popcount.col` |

⚠ **The kind is checked, not assumed.** A file found under a name it does not
match — `smi.blk` carrying kind 1, say — is an error, not something to interpret
by filename. That check is what catches a file copied into the wrong place.

---

## Row identity

A row id is a `u64`:

```
row_id = (shard << 32) | local
```

where `shard` is the manifest's `shard` and `local` is the row's position in its
shard, counting from zero. The ordering that falls out — shard-major, position
within shard — is the order moleculo returns rows in.

⚠ **A row id means something only inside one generation.** A rebuild that
reuses ids for different molecules must raise `generation`, and anything that
stored row ids across such a rebuild is stale. Identify a molecule outside the
index by the identifier in its record, never by its row id.

⚠ **The `arthor.index` column on the HTTP wire is not this row id.** It carries
the row's position *inside its shard*, so on a database of more than one shard
two different rows in one answer can show the same value. Match a row you got
from the API to a row in the index by its **identifier**, not by that column.

⚠ **Bit 63 is reserved and is zero in every shard this format writes.** It is
kept for a future id that is *derived* rather than stored, so that a reader can
tell the two apart; treat a row id with the high bit set as one you do not
understand, and shard ids as bounded by 2³¹ rather than 2³².

---

## The block store

`mol.pack` + `mol.off`, `smi.blk` + `smi.off` and `fp.col` + `fp.off` are the
same structure: a **payload** of variable-length records and a side file of
**block offsets**.

**In the payload**, each record is written as its length, then its bytes:

- the length is an unsigned **LEB128** varint: seven bits of the value per byte,
  least-significant group first, with the high bit `0x80` set on every byte but
  the last;
- then exactly that many bytes of the record.

Records are grouped into **blocks** of `block_molecules` records (the last block
may be short). ⚠ **The block, not the record, is the compression unit.** When
`record_codec` is `"zstd"`, the whole block — length prefixes and all — is one
zstd frame; when it is `"none"` the block's bytes are written as they are. A
record never spans two blocks.

⚠ **The frames carry no content size.** They are produced without one, so a
decompressor cannot ask how large a block will be and must grow its buffer until
the frame fits. Guessing a multiple of the compressed size and giving up on
failure is how moleculo itself once lost 93% of a 124 M-row build while exiting
successfully.

**In the offsets file**, after its own 64-byte header, there is one `u64`
little-endian per block, and then **one more**:

- entry `b` is where block `b` starts, counted from the **first payload byte
  after the payload's header** — not from the start of the file;
- the final entry is where the last block ends, which is what gives the last
  block a length without consulting the file size.

So the byte range of block `b` is `[offsets[b], offsets[b + 1])`, and the number
of blocks is `ceil(molecules / block_molecules)`.

**To read row `local`:**

1. `block = local / block_molecules`, `within = local % block_molecules`;
2. take the payload bytes `[offsets[block], offsets[block + 1])`;
3. decompress them if the codec says so;
4. from the start of the block, read `within` records — each a LEB128 length and
   that many bytes — and skip them;
5. the next length and bytes are the record.

Sequential reading never needs the offsets at all: decode block by block.

**Corners, all of which occur:**

- a block holds exactly `min(block_molecules, molecules - block * block_molecules)`
  records — no more, no fewer. ⚠ **Check that count**, because it is the check
  that catches a damaged length prefix: a corrupted varint yields records that
  are individually well-formed and a block that holds the wrong number of them,
  and every other rule here passes;
- a shard of zero molecules has no blocks and an offsets file holding the single
  trailing entry;
- a LEB128 length longer than ten bytes, or one whose groups shift past 64 bits,
  is corrupt rather than large.

---

## The record store: `smi.blk` and `smi.off`

A record is **the input line the molecule was built from**, with trailing
whitespace removed, as bytes. moleculo's inputs are SMILES files, so in practice
a record is:

```
CC(=O)Oc1ccccc1C(=O)O<TAB>CHEMBL25
```

The text before the first tab is the SMILES; everything after it is whatever the
input carried — usually the identifier, and that identifier is what `index
delete` names a row by and what search results return.

⚠ **There is one record per addressed row, deleted rows included**, and records
are in row order. Row `local` of shard `s` is record `local` of that shard's
record store, and its row id is `(s << 32) | local`.

⚠ **A record is valid UTF-8, and that is a property of the builder rather than
of the format.** The ingest reads its input as UTF-8 lines and fails the build on
a byte sequence that is not, so no shard can hold a record that is not — decode
strictly and treat a failure as a corrupt file rather than as text to salvage.

⚠ **A line the SMILES parser rejected is not a row at all.** The builder skips
it, so input line numbers and row ids do not correspond; do not compute one from
the other.

---

## Deletions: `deleted.roar`

A header and then a plain **bitmap**, one bit per addressed row:

- row `local` is deleted when bit `local % 8` of payload byte `local / 8` is set
  (bit 0 is the least significant);
- the payload may be shorter than `ceil(molecules / 8)`, or absent entirely — a
  row past its end is **not** deleted;
- the header's count field (bytes 12..20) says how many rows are marked, so a
  live count costs an open rather than a popcount;
- only bits below `molecules` mean anything. A payload longer than
  `ceil(molecules / 8)`, or a bit set at or past `molecules`, addresses a row the
  shard does not have: ignore it rather than refusing — and never let it reach a
  count.

⚠ **The bitmap is the authority and the header count is a cached popcount of
it.** They are written together and cannot disagree unless something edited the
file, so if they do, the file is inconsistent: trust the bits, and say that the
two disagree. It matters because moleculo's own reader takes the count from the
header and the liveness from the bits, so an inconsistent file makes it report a
`deleted` that its own answers do not match.

⚠ **The name promises a format it does not use.** `.roar` was chosen when a
roaring bitmap was planned; what is written is a dense bitmap, and the file's own
header is what tells you what it is. Do not infer the encoding from the
extension.

⚠ **Every shard has this file, and in most shards it is a bare header** — that
is the normal state, and it reads as *nothing is deleted*.

⚠ **A reader that ignores this file reports rows that no longer exist.** The
deletion is logical: the record, the packed molecule and the fingerprint all
stay on disk until the next rebuild, so a tool that walks the record store
without consulting the bitmap will hand back deleted molecules with a straight
face.

---

## Reserved files

Named so that a reader knows what it is stepping over, and deliberately not
described here:

| file | holds |
|---|---|
| `mol.pack`, `mol.off` | the packed molecule graphs the matcher walks. A block store as above; the **record encoding** is not specified |
| `fp.col`, `fp.off` | fingerprints. A block store, blocked by `fingerprint_block_molecules`, which is **not** `block_molecules` |
| `popcount.col` | one popcount per row, for the similarity scan |
| `screen.idx` | the screening index: feature to the rows carrying it |

A container-level reader can skip them all and still list every live row with
its id and its original record.

---

## What a reader must do

1. **Refuse an unknown `format_version`**, in a file header or in a manifest,
   naming the version it found and the one it knows. Refusing loudly is the
   whole reason the field exists: a v2 shard read with v3 rules returns
   molecules that are noise wearing a valid shape.
2. **Refuse the wrong magic or the wrong kind**, for the same reason.
3. **Refuse a truncated file** — anything shorter than 64 bytes, an offsets
   table with fewer than `blocks + 1` entries, a block range outside the
   payload, a record whose length runs past the end of its block.
4. **Accept unknown `features` entries and unknown manifest fields.** They are
   additive by design.
5. **Consult `deleted.roar`** before reporting a row.
6. **Check the generation** of every shard against `database.json`.
7. **Report a degraded database in its own output.** When a shard named by
   `database.json` is absent, the numbers are a floor and any digest covers only
   the shards that were there. Say it where the numbers are, not only on stderr:
   a total piped into something else reads as final.
8. ⚠ **Never write into a shard.** An index is derived; if it is wrong, rebuild
   it from its input.

---

## A worked reading

To list every live row of a database as `row_id<TAB>record`:

```
read database.json           -> generation, shard ids   (or: one bare shard)
for each shard id, ascending:
    read manifest.json       -> format_version, shard, generation, molecules,
                                block_molecules, record_codec
    refuse if format_version != 3 or generation != database generation
    open smi.blk             -> check magic, version, kind 3
    open smi.off             -> check magic, version, kind 2
    open deleted.roar        -> check magic, version, kind 4
    for local in 0 .. molecules:
        skip if the bitmap marks it
        record = block store lookup as above
        emit ((shard << 32) | local), record
```

The count of what this emits is the database's live row count, and it is the
number moleculo reports as `recordsTotal`.

⚠ **If you are checking a reader against moleculo**, `moleculo index inspect
<shard>` prints `molecules`, `live` and `deleted` for one shard, and a served
database reports `recordsTotal` — three numbers that must agree with what a
reader of this document computes. Two shards built from the same input by the
same builder version are byte-identical, which is what makes such a comparison
meaningful at all.
