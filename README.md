# moleculo

Chemical structure search over large molecule collections — substructure,
SMARTS, similarity and exact formula — served over an HTTP API that is
wire-compatible with Arthor.

A single static binary with a search page inside it. No runtime, no
dependencies, no installer, no database server, and **no request to anything but
this process** — no CDN, no font service, no structure resolver. It runs on a
network with no route to the internet, which is where confidential chemistry
usually lives.

**Bring your own molecules**: nothing is bundled, nothing is uploaded, and query
structures never leave the process.

---

## Contents

- [Install](#install)
- [Quick start](#quick-start) — five minutes to a working search
- [Preparing your molecules](#preparing-your-molecules)
- [Building an index](#building-an-index)
- [Running the server](#running-the-server)
- [Searching](#searching) — the four query types
- [Getting results out](#getting-results-out)
- [Operating it](#operating-it) — adding data, appending and deleting without a rebuild, reloading, a missing shard, stopping, probes, metrics
- [The search page](#the-search-page) — draw a query, read the results, keep what matters
- [What this is measured against](#what-this-is-measured-against) — RDKit, Arthor, where this loses and where the answer is better
- [Which of these you actually want](#which-of-these-you-actually-want)
- [Known limitations](#known-limitations)
- [Security](#security)

The index format is specified separately, in
[`INDEX-FORMAT.md`](INDEX-FORMAT.md): enough to write a program that lists what
a database holds — row ids, original records, which rows are live, the stored
molecules — and checks every checksum, without this software. It describes
format version 4, which 2.0 writes; version 3, the frozen format of 1.x that
`index migrate` reads, is summarised at its end.

What 2.x promises, and what it deliberately does not, is
[`COMPATIBILITY.md`](COMPATIBILITY.md): the wire, the command line, the
configuration, index format 4 and the metric names are stable. An answer
moves only in a minor release, and only under a heading that says so. What
2.0 changed against 1.x is listed there too, under *Upgrading from 1.x to 2.0*.

⚠ **This is a release candidate, 2.0.0-rc.2.** Its search speed against 1.1.0
was measured on 200 000 rows, where the bar names 1 000 000, and it has not
yet been compared with Arthor; 2.0.0 follows both.
[Known limitations](#known-limitations) says what else is unmeasured.

---

## Install

Download the archive for your platform from
[Releases](https://github.com/denyzhirkov/moleculo/releases), verify it, unpack
it, and put the binary on your `PATH`.

```sh
tar xzf moleculo-<version>-<platform>.tar.gz
cd moleculo-<version>-<platform>
shasum -a 256 -c ../SHA256SUMS      # optional, SHA256SUMS is in the same release
./moleculo --help
```

| platform | archive |
|---|---|
| macOS, Apple Silicon | `aarch64-apple-darwin` |
| Linux x86-64 (x86-64-v2: Intel since 2008, AMD since 2011) | `x86_64-unknown-linux-musl` |
| Linux arm64 | `aarch64-unknown-linux-musl` |

The Linux builds are statically linked against musl: no glibc version
requirement, and they run on any distribution with a matching CPU.

⚠ **From 1.1.0 the x86-64 build needs an `x86-64-v2` CPU**, with `POPCNT` and
SSE4.2: any Intel processor since Nehalem (2008) and any AMD since Bulldozer
(2011), which is the baseline RHEL 9 already requires. Older processors stop
at the first instruction they lack (`Illegal instruction`), not at a wrong
answer. Similarity scoring is popcounts, and baseline x86-64 has no popcount
instruction. The server names what its build uses at startup:
`similarity kernel: popcnt`.

---

## Quick start

```sh
mkdir -p sources indexes

cat > sources/demo.smi <<'EOF'
CC(=O)Oc1ccccc1C(=O)O	aspirin
CN1CCC[C@H]1c1cccnc1	nicotine
CC(C)Cc1ccc(cc1)C(C)C(=O)O	ibuprofen
Cn1cnc2c1c(=O)n(C)c(=O)n2C	caffeine
EOF

moleculo serve ./sources --build-dir ./indexes
```

The server builds `demo.smi` into a database called `demo` and starts serving
it. Then, from another terminal:

```sh
curl -G 'http://localhost:8080/dt/demo/search' \
  --data-urlencode 'type=Substructure' \
  --data-urlencode 'query=c1ccccc1' \
  --data-urlencode 'fmt=tsv'
```

Everything containing a benzene ring comes back as tab-separated SMILES and
identifiers.

---

## Preparing your molecules

One molecule per line: **SMILES, a tab, an identifier.** Anything after a second
tab is ignored.

```
CC(=O)Oc1ccccc1C(=O)O	CHEMBL25
CN1CCC[C@H]1c1cccnc1	CHEMBL3
```

This is the shape ChEMBL, PubChem, Enamine and Mcule exports already come in.
PubChem's `CID-SMILES` needs its two columns swapped, since it ships
identifier-first.

A line the parser cannot read is counted and skipped, not fatal — one bad row in
a hundred million must not fail a build. The build reports how many it skipped
and shows the first few reasons.

**How correct is the reading?** Every molecule is diffed against RDKit, field by
field, over two corpora from different producers:

| corpus | molecules | agree on every field | refused where RDKit accepts |
|---|---:|---:|---:|
| ChEMBL | 2 897 819 | **99.992%** | **0** |
| ZINC20 | 5 200 000 | **99.9999%** | **0** |

The remaining divergences are catalogued rather than unknown, and on the ZINC
sample **none of them is ours**: all three aromatic disagreements are cases
where RDKit's own answer changes with how the molecule is written.

⚠ **Read the two numbers together.** The difference between them is the
chemistry, not the engine. ZINC is enumerated combinatorial space assembled from
validated building blocks; ChEMBL is forty years of medicinal chemistry
literature, with the metal complexes and drawing conventions that come with it.
Which figure your collection resembles depends on where your molecules came
from.

Refusals are the other direction of the same question, and they are measured on
the whole of PubChem rather than a sample: of 124 469 489 rows, 42 142 are
refused (0.034%), every one of them for a valence no element allows —
`FBr(F)F`, `O=Cl(=O)(=O)F` — and RDKit refuses them too.

⚠ **Both refusal columns read zero since 0.19.0, and until then this section
argued for the refusals.** It said the ZINC sample turned up fourteen molecules
RDKit reads and this build refused — each with a **triple bond between two
aromatic atoms inside a ring**, `c1sc#cc1I` — and that refusing them was being
right, because a linear triple bond cannot sit in a planar aromatic ring. The
chemistry in that sentence stands; the conclusion did not. **Reading a string
and deciding what it means are different jobs**, and the reader was letting the
perceiver decide whether a string could be read at all. Arthor reads every one
of those strings and calls the ring *not aromatic* — tolerance in the reader,
strictness in the perceiver, at the same time. So does this build now:
`c1sc#cc1I` reads, and reads as 0 aromatic atoms, which is what Arthor answers
and RDKit does not. On a rebuilt every-thirtieth-line sample of 5 200 000 ZINC20
molecules the old reader refused **7** that RDKit reads; the new one refuses
**none**.

ChEMBL's were four **aromatic rings containing boron** — `Cc1b[n+](…)cc(C)c1`
and three others — and they read now too, for the same reason: `[b-]` is
isoelectronic with carbon and takes a ring double bond, borabenzene takes one,
and only borole with a hydrogen on the boron does not. Every cell of that table
was run against RDKit rather than reasoned. ⚠ **The number in the table read
"5" for a day on a corpus that could not have contained a refusal** — the
oracle's ChEMBL file had been recovered from a shard, so it held only rows the
build already accepted. Refusals are measured on the re-fetched file now, all
2 897 819 rows, and the 13 this build still refuses are refused by RDKit too:
`[PH]F6` and `[AsH]F6` written with a hydrogen, a valence no element allows.

---

## Building an index

An index is a derived artifact: reproducible from the input and the builder
version, and safe to delete and rebuild. **A shard is never edited in place.** A
database can gain a shard or a set of deleted rows without a rebuild — see
[Adding to a database](#adding-to-a-database-without-rebuilding-it) and
[Deleting rows](#deleting-rows) — and both write a new file beside the old and
swap it in, so a reader never sees half of either.

⚠ **This build reads index format 4 and nothing older.** A shard of format 3,
which every 1.x release wrote, is **refused by name**, loudly, with the command
that converts it: see [Upgrading a 1.x index to
2.0](#upgrading-a-1x-index-to-20). A shard of format 2, built before 0.8.0, is
refused as a format version this build does not read, and it cannot be
converted: rebuild it from its input. Neither is read with the wrong rules: this
is exactly what the version field is for, and it is why an older shard is not
silently misread into molecules that are garbage wearing a valid shape.

```sh
moleculo index build molecules.smi ./indexes/mydb
moleculo index inspect ./indexes/mydb
```

Above a few tens of millions of molecules, build it as shards — they are built
concurrently and searched as one database:

```sh
moleculo index build big.smi ./indexes/big --shards 8 --generation 1
moleculo index inspect ./indexes/big/shard-0000
```

Past a few hundred million, the shard count matters more than that and eight is
the wrong answer — see [choosing the shard
count](#choosing-the-shard-count-and-why-eight-is-not-always-the-answer).

| option | meaning |
|---|---|
| `--bits N` | similarity fingerprint width: any positive multiple of 64, 512 by default. Narrower costs recall, wider costs disk. Fixed at build time — it is a property of the index, not of a query. |
| `--codec none\|zstd` | compress the record store, which holds your input lines. Default `zstd`. The molecule store is always LZ4: index format 4 fixes it. |
| `--fp-codec` | the same, for the fingerprint column. ⚠ `--fp-codec none` is **6.6x the similarity throughput for about 26% more index** — measured, see below. |
| `--replace` | swap out a shard that is already there. Requires a higher `--generation`. |
| `--resume` | continue an interrupted sharded build: a shard already sealed is skipped rather than refused. ⚠ Only correct if the input file has not changed — a shard records how many molecules it holds, not which bytes it came from |
| `--generation N` | index generation. Row identifiers are stable within one generation and only within one. |
| `--shard N` | the id of the shard this build writes, the high half of every row id in it. Default 0. Refused beside `--shards` of more than 1, which assigns the ids itself, and refused at 2³¹ or more. |
| `--notify URL` | call a running server's reload endpoint once the shard is published. |
| `--threads N` | workers for the pass that counts the screening vocabulary. Default: the machine's parallelism. Does not change the shard; does raise memory. |
| `--shards N` | split the input into N shards. **Size the shard, not the machine** — see below. |
| `--build-workers N` | how many shard builds run concurrently. Default: the machine's parallelism. ⚠ **Lower this on slow or networked storage**; it is a bound on concurrent I/O as much as on memory. |

**Plan for the time.** Building is CPU-bound and it is hours rather than minutes
at scale.

⚠ **Everything measured below is index format 3 or earlier, which built a
screening index; format 4 builds none.** Format 4's time at scale has not been
measured. Its size has, on one sample: the first 1 000 000 rows of PubChem
(999 972 molecules built) make one shard of **84.9 bytes per molecule**, of
which the molecule store is 40.1 — named so it is not read as a
collection-wide figure.

Measured on a ten-core laptop, formats 2 and 3:

| molecules | one core | ten cores | index size | bytes/molecule |
|---:|---:|---:|---:|---:|
| 10 M PubChem | 50 minutes | — | 0.89 GB | **89.25** |
| 124.4 M PubChem, one shard | 10 h 43 | **3 h 58** | 10.7 GB | 86.4 ⚠ |
| 124.4 M PubChem, 16 shards | — | ~18 h at two threads ⚠ | 10.49 GB | **84.34** |

Note which way the density moved. Bigger collections cost *less* per molecule
on these formats, because the screening index compressed better when each
feature had more rows in it, so sizing from a small trial run overestimated
rather than under. ⚠ Format 4 has no screening index, so that mechanism is gone
from it and the direction is unmeasured. On ChEMBL,
whose chemistry is heavier, the same build is **133.2** bytes per molecule — the
range across the corpora tried here is 86 to 133.

⚠ **The one-shard 124.4 M row is marked because it was measured on the previous
index format**, and it is kept rather than replaced. The row under it is the
same collection rebuilt for 0.10.0 — but on **format 3 and in 16 shards, at two
threads on a laptop somebody was using**, so its size and its time answer
different questions from the row above and neither is a correction of the other.
Density falls with shard size as well as with format, so 86.4 → 84.34 is two
causes and this document will not pretend to have separated them. ⚠ The **time**
in that row is not a measurement of this software at all: it is what a
deliberately gentle build costs on a busy machine, recorded so nobody reads the
blank in the "one core" column as missing data.

The 10 M row above **was** rebuilt on this format for 0.9.0 and moved 91.6 →
89.25, which is −2.6% and exactly what the format change predicts;
the ChEMBL figure beside it was measured on this build. The two moved in opposite
directions — ChEMBL grew from 129.5 despite the smaller molecules, because other
slices have been added since — which is why neither number is scaled from the
other.

On format 3, three quarters of that was the two screening passes, which ran on
as many workers as `--threads` asks for and went **6.5x faster** on ten cores —
8 hours to 74 minutes at 124 M — while the whole build gained **2.69x**, because
reading the input, merging the posting runs and writing the fingerprints were
sequential. Format 4 keeps one of those passes, the one that counts the
vocabulary, on `--threads` workers, and has no posting runs to merge; reading
the input, encoding the molecules and writing the fingerprints stay sequential.

Parallel work is not free work: ten threads spend **14% more processor time in
total** than one, since every worker keeps its own copy of the feature counts.

**Shards are the parallelism that gets past that.** Shards are independent by
construction, so building them at once has none of the sequential tail one build
has. Measured on the same laptop, 50 M PubChem as eight shards:

| | |
|---|---|
| wall clock | **26 min 44** |
| processor time | 3 h 38 |
| speed-up | **8.17x** |
| peak memory | **3.94 GB** |
| index | 3.9 GB, 83.2 bytes/molecule |

⚠ **Budget the memory from that figure, not from one build's.** A single build
peaks around 211 MB, and eight of those is not 1.7 GB: each shard build runs its
own counting threads, so the two kinds of parallelism multiply. If the machine
is smaller, turn `--threads` down as well as `--build-workers`.

### Choosing the shard count, and why eight is not always the answer

**Size the shard, not the machine.** ⚠ **What follows is a format-3 build's
problem, kept because that is where it was measured.** Format 4 writes no
postings and spills nothing, so it has no merge to thrash; a shard's size still
bounds how long its build takes and how much a failed build loses.

On format 3, a shard's postings were spilled to sorted runs and then merged, and
the merge read every run at once — so the number of open streams during a build
was *runs per shard × concurrent builds*. Runs scaled with shard size, at
roughly one 64 MB run per 570 000 molecules:

| molecules per shard | runs | 8 builds at once | 3 builds at once |
|---:|---:|---:|---:|
| 6 M | ~11 | 88 streams | 33 |
| 15 M | ~26 | 210 | **79** |
| 60 M | ~106 | **845** | 317 |
| 240 M | ~420 | 3 400 | 1 260 |

On storage that handles concurrent random reads well, the high numbers are fine.
On anything else they are not, and the failure does not look like a failure:
**the build keeps running, writes files, and reports nothing wrong.**

⚠ **The diagnostic is the pair of numbers, not either alone.** If the processor
is mostly idle *and* the disk is delivering single-digit MB/s, the merge is
thrashing — the same enclosure that reads at 663 MB/s sequentially was measured
at **4 MB/s** with 845 streams open, and a build projected at 17 hours was on
course for 50. Reducing the shard size and the concurrency fixed it; nothing
else had to change.

A reasonable default for a large collection on ordinary storage:

```sh
# ~15 M molecules per shard, three builds at a time
moleculo index build big.smi /data/big --shards 128 --build-workers 3 --generation 1
```

And for a few tens of millions, where the merge is short either way:

```sh
moleculo index build mid.smi /data/mid --shards 8 --build-workers 8 --generation 1
```

The input is divided by byte range rather than copied, so a 50 GB input does not
need 50 GB of scratch to be split. A shard that fails does not abandon the
others: the database serves what sealed and reports the rest as missing.
The wall clock is what improves.

Two builds of the same input are byte-identical whatever `--threads` says. That
is a property the test suite asserts rather than a claim: an index is a function
of its input and the builder version, and a thread count is neither.

**The fingerprint codec is the similarity lever, and it is a build-time choice.**
Measured on one corpus, one machine and one commit, differing only in
`--fp-codec`:

| | `zstd` (default) | `none` |
|---|---:|---:|
| similarity, ten threads | 88 M molecules/s | **553 M/s** |
| similarity, one thread | 14 M/s | 116 M/s |
| substructure scan | unchanged | unchanged |
| fingerprint column, 2.9 M molecules | 95 MB | 188 MB |
| whole index, 2.9 M molecules | 0.39 GB, 133 B/mol | 0.49 GB, 168 B/mol |

Measured at 1.1.0 on ChEMBL (2 897 806 molecules), built twice from one input
with only `--fp-codec` changed. The figures are medians of three rounds, each
alternating with 1.0.0 on the same index, which read 84 and 327 M/s on ten
threads and 13 and 89 on one. ⚠ **The ten-thread `none` figure is the noisiest
in the table**: its three rounds ran from 415 to 756 M/s on a laptop whose load
average stood near 20. The previous release's 580 sits inside that range.
Read it as "several hundred million per second". On a quiet machine, measure
it yourself.

The substructure scan does not move because it never reads that column, which is
the check that says the rest of the table is real. **Compression stays the
default**; if similarity is what your deployment does all day, `--fp-codec none`
is the one flag that matters — and on this corpus it costs a quarter more disk,
not the half it costs on a lighter one, because the fingerprint column is a
smaller share of a heavier index.

**Plan for the memory, and read the number carefully.** The build is out-of-core
and its own allocation stays small — 186 MB on one thread at 124 M, and 849 MB
on ten, because each worker holds its own feature counts. What
`/usr/bin/time -l` reports as maximum resident set is much larger, 3.6 GB on that
build, because each pass reads the shard back through a memory map and the
kernel counts those file pages as resident. They are evictable: the build does
not need that memory and will not fail without it. A machine with far less RAM
than the dataset can build it.

A build writes into `<name>.building` and renames it into place when the shard
is sealed, so a running server never sees a half-built index, and a build that
dies leaves nothing behind but the staging directory.

---

### Upgrading a 1.x index to 2.0

2.0 reads index format 4. An index built by any 1.x release is format 3, and 2.0
**will not serve it until it is converted**. The server says so at startup and
names the command:

```
3 of the 3 shards of ./indexes/mydb are the 1.x index format; convert them with `moleculo index migrate`
```

Convert it in place:

```sh
moleculo index migrate ./indexes/mydb
```

```
  shard 0    migrated: 163 rows, 4 deleted
  shard 1    migrated: 137 rows, 2 deleted
./indexes/mydb is format 4; a server running with --enable-reload picks it up at its next reload, any other at its next start
```

**You do not need the original input file.** Every 1.x index keeps the input
line each molecule came from, and the conversion rebuilds the index from those
lines. What you get is the index a fresh 2.0 build of the same lines would
write, byte for byte, with three things kept from the old one:

- **every row identifier**, so a client that stored them keeps valid ones;
- **every record**, exactly as stored: the `SMILES` and identifier columns do
  not change;
- **every deletion** made with `index delete`.

⚠ **A 2.0 database can answer a query differently from its 1.x self.** The
conversion re-reads the chemistry with 2.0's rules, so where 2.0 changed an
answer (see the changelog's *Answers moved*), the converted database gives the
new answer. That is the point of converting rather than copying.

⚠ **A molecule 2.0 cannot store is deleted, not dropped.** Dropping it would
renumber every row after it. Every one is listed in a file beside the database,
`./indexes/mydb.migrate-refused.tsv` here, which the command names when it
writes to it: shard, row id, identifier and the reason, one row per line. The
structure is deliberately not in it — the identifier finds the line in your own
input. A run that resumes adds to the file rather than replacing it. `index
inspect` counts these rows among the deleted ones.

⚠ **A line 1.x refused was never in the index**, so it is not in the converted
one either, even if 2.0 would accept it. Only a rebuild from your input file
adds it.

**It is safe to interrupt.** Each shard is built beside the old one and swapped
in once it is complete and on disk; the old shard is removed only once the new
one opens. If the run stops — a crash, Ctrl-C, a full disk — run the same
command again. Shards already converted are skipped, and a shard that was
half-built is started over. A shard that fails keeps its 1.x files untouched; no
further shard is started, the ones under way finish, and the command says which
failed and why:

```
  shard 1    FAILED: No space left on device (os error 28) — its 1.x files are untouched
moleculo: a shard failed; run the command again to continue — converted shards are skipped
```

The path may be relative, `.`, or a symbolic link, to the database or to any of
its shards: each shard is built beside the directory it really is, so a link
still points at the converted shard afterwards.

**Plan for**:
- **time**: converting is a build, and it runs like a sharded one. Shards are
  converted `--build-workers` at a time, the machine's parallelism by default,
  and `--threads` sets the workers of the one parallel pass inside each, as for
  `index build`. A database of one shard converts on one worker, as it would
  build; see the figures above;
- **disk**: the old database, plus one new shard for each shard being converted
  at once — up to `--build-workers` of them — since each old shard is removed
  only after its replacement is complete. The two formats have not been sized
  on the same collection: format 4 read 84.9 bytes a molecule on the first
  1 000 000 rows of PubChem, and format 3 read 89.25 on the 10 M PubChem
  build in the table above, which is a different population. `index inspect` on the first converted shard,
  beside an old one, tells you for your collection. On a tight disk, lower
  `--build-workers`;
- **memory**: as for a sharded build, it multiplies with both settings.

**Stop the 1.x server first.** It keeps the old files open until it exits, so
their disk space is not given back while it runs, and at its next reload it
cannot read a converted shard. A 2.0 server pointed at a database part-way
through serves the converted shards and warns that the rest need converting; it
picks each one up at its next reload if it runs with `--enable-reload`, and at
its next start otherwise. Run one conversion per database at a time.

The generation does not change: it marks when a row identifier starts meaning a
different molecule, and here none does.

⚠ **A 2.0 shard built before a later chemistry change keeps its old answers,
and says so.** A 2.0 shard stores the chemistry it perceived at build —
hydrogens, rings, aromaticity — and the version of the rules that produced it.
After an upgrade that changes those rules, the shard is served as built: the
server logs one warning per such shard when it opens the database, and
`/dt/data` adds a `perception` field with the stored version to that shard's
index entries. Rebuild the shard from its input to take the new rules;
`index migrate` converts only 1.x indexes.

⚠ **A database answered through other servers** (a coordinator with
`parts.json`) holds no shards of its own, so the command refuses it. Run it on
each part's machine, against that part's own copy.

⚠ **A format-4 shard built by a 2.0 pre-release** that this build refuses as
"must be rebuilt" is not converted: rebuild it from its input. So is a
converted shard this build cannot open, which should not happen; its 1.x copy is
then kept beside it, as `<shard>.v3.building`, and the command names it.

## Running the server

```sh
moleculo serve ./indexes [more/roots ...] [--port 8080]
```

### Configuration, and what it refuses

Every setting is reachable as a flag **or** as an environment variable, because
a container is configured by its environment and a shell is not. **The flag
wins** — an operator debugging a running system overrides the image, not the
other way round.

| flag | environment |
|---|---|
| `<databases-dir>...` | `MOLECULO_DATABASES` (colon separated, like `PATH`) |
| `--bind` | `MOLECULO_BIND` |
| `--port` | `MOLECULO_PORT` |
| `--build-dir` | `MOLECULO_BUILD_DIR` |
| `--max-seconds` | `MOLECULO_MAX_SECONDS` |
| `--max-candidates` | `MOLECULO_MAX_CANDIDATES` |
| `--max-concurrent-searches` | `MOLECULO_MAX_CONCURRENT_SEARCHES` |
| `--max-queued-searches` | `MOLECULO_MAX_QUEUED_SEARCHES` |
| `--max-upload-bytes` | `MOLECULO_MAX_UPLOAD_BYTES` |
| `--enable-reload` | `MOLECULO_ENABLE_RELOAD=1` |
| `--enable-index-control` | `MOLECULO_ENABLE_INDEX_CONTROL=1` |
| `--enable-upload` | `MOLECULO_ENABLE_UPLOAD=1` |
| `--no-search-limit` | `MOLECULO_NO_SEARCH_LIMIT=1` |
| `--no-tautomer-report` | `MOLECULO_NO_TAUTOMER_REPORT=1` |
| `--part` | `MOLECULO_PARTS` (comma separated; the flag repeats, once per part) |
| `--shutdown-grace-seconds` | `MOLECULO_SHUTDOWN_GRACE_SECONDS` (default 30) |
| `--shutdown-delay-seconds` | `MOLECULO_SHUTDOWN_DELAY_SECONDS` (default 0) |
| `--access-log` | `MOLECULO_ACCESS_LOG=1` |

⚠ **It refuses rather than guesses, at startup rather than at the first
request.** Three cases are worth knowing because each is a mistake that used to
be silent:

- **an unknown flag is an error**, not something ignored — a typo that is
  ignored is a setting you believe you applied;
- **a value that does not parse names the field and quotes back what it got**;
- **a contradiction is refused.** `--no-search-limit` removes every bound, so
  giving it alongside `--max-seconds` means you believe something untrue about
  your server, and picking a winner would leave you believing it. The same rule
  refuses `--enable-upload` without `--build-dir`: uploading with nowhere to
  write is a failure found by whoever tries it first rather than by you.

### The three switches that grant a power

⚠ **Everything else this server does is read-only. These are not, they are all
off by default, and each is its own flag on purpose** — an operator who enabled
one in an earlier version did not ask for the next.

| switch | what it permits |
|---|---|
| `--enable-reload` | `POST /admin/reload` — rescan the roots, build anything new, swap the catalog |
| `--enable-index-control` | `PUT /dt/{db}/data` — warm or evict an index; `POST /dt/setpriority` — the order databases are listed in |
| `--enable-upload` | `POST /dt/upload` — write a collection into the build directory |

⚠ **`--enable-upload` is the one to think about twice.** It lets an
unauthenticated caller write a file to your disk. It is bounded — 2 GB by
default, `--max-upload-bytes` to change it, and an oversized body is refused
before it is read into memory — the name is checked against a strict allow-list
because it becomes a filename, and a name already in use is **refused rather
than overwritten**. But there is no authentication in front of it, so switch it
on only where you would also be comfortable with an open port.

⚠ **`--bind` defaults to `127.0.0.1`, deliberately.** The port is the only
boundary this product has, so loopback is the default everywhere except where
something opts in — which the container image does, because inside a container
loopback reaches nobody.

### Before you start: what will it cost?

```sh
moleculo index estimate molecules.smi
```

```
molecules      1000000
sampled        49994 built, 6 refused by the parser
index size     94.2 MB
peak disk      94.2 MB  (the index itself: nothing temporary is written; --replace also holds the old shard until the swap)
build time     55 s  ⚠ direction of error unmeasured, see below
```

**It measures your data rather than quoting ours.** Fifty thousand of your
molecules are built into a real index and scaled up, so the numbers come from
your chemistry on your machine. ⚠ A density constant would have been a guess
wearing a number — the collections measured here differ by half, from 83 bytes a
molecule to 129.5, and that spread is exactly what decides whether a disk is big
enough.

**Peak disk is the finished index.** A build writes each file straight to its
final name in `<name>.building` and publishes it by rename, so nothing
temporary sits beside it. Measured on the first 1 000 000 PubChem rows with
2.0.0-rc.1, reading the staging directory every 0.2 s with `du -A` (the bytes
in its files, in KiB): its largest reading was 82 323 KiB against a finished
shard of 82 916 KiB, and it held only the shard's own ten files throughout.
The disk blocks allocated to it, read with plain `du`, peaked at 84 800 KiB
against the finished shard's 84 812. ⚠ `--replace` keeps the old shard until the new one is swapped in,
so a replacement needs room for both.

⚠ **Size reads high, and the one measurement of it says why.** The example
above is that 1 000 000-row input, which built to **81.0 MB** — 84 902 882
bytes by `index inspect`, in the estimate's unit of 1 024² bytes, the same
shard as the 82 916 KiB above — so the estimate is **+16%**. Most of the gap is the
manifest's feature vocabulary, which is close to a fixed cost per shard rather
than a cost per molecule: 19.6 bytes a molecule in the 50 000-row sample
against 3.2 in the built shard, and scaling the sample multiplies it. Provision
from the estimate and you will have room.

⚠ **Time has not been checked against a build on this index format.** One
estimate and one build, on a machine somebody was using, is no comparison. On
the previous format time read **−25%** against a real 10 M build, because a
sample sorted its screening postings in memory where a real build spilled and
merged them; this format has neither. Allow margin.

Neither is corrected by a fudge factor. One ratio from one comparison is a
guess, and a guess dressed as a correction is worse than a number with its
provenance attached.

The refusal count is worth reading too: molecules this build cannot parse are
cheaper to find out about now than at hour three.

### Watching a build

⚠ **A build used to print nothing until it finished**, and at 124 M molecules
that is four hours. There is no way to tell that from a hang, and killing it is
what people do. So it now says where it is — and since 0.10.0 killing it costs
only the shard in flight, because `--resume` continues the rest:

```
INFO starting          pass="screening vocabulary" molecules=9999789
INFO working           pass="screening vocabulary" done=3973120 total=9999789
                       percent=40 per_second=66193 pass_seconds_left=91
INFO pass complete     pass="screening vocabulary" molecules=9999789
                       seconds=150 per_second=66588
```

**One line every thirty seconds**, per pass, on stderr — `RUST_LOG=info`, which
is the default. A pass adds two more lines of its own, one when it starts and
one when it finishes, so a build costs roughly `duration / 30s` lines per pass
plus eight. A 7 420-molecule build costs **9 lines** and finishes inside one
interval; a build that runs for an hour costs a few hundred.

⚠ **That is a rule rather than a number on purpose.** This section used to quote
"24 lines for a 10 M build", which was true of one machine at one thread count
and of nothing else: the count is a function of how long the build takes, and
that is exactly the figure the rest of this document says depends on your
hardware. Measured both ways on the same 10 M collection and the same code:
**24 lines at ten threads, 59 at two.**

⚠ **`pass_seconds_left` is of the current pass, not of the build.** A build has
**three** passes over every molecule and they run at different rates — ingest,
then the screening vocabulary, then the fingerprints (on format 3, which had a
fourth, the two screening passes ran near 67 000 molecules a second on a
ten-core laptop and the fingerprint pass near 126 000) — so a build-wide
estimate would be a number this cannot honestly produce.

⚠ **Ingest reports in bytes, and it is the only pass that can.** The other two
take their denominator from the shard's manifest, which already knows how many
molecules it holds. Ingest is the pass that *discovers* that number, so it has
none to report against; what it has is the length of the input it was handed.
Each line names its `unit`, so a rate is never ambiguous about what it counts. What you get is a measurement of the pass you are
watching.

### When a build runs out of disk

⚠ Measured, not asserted — on a volume deliberately 72 KB too small for the
index it was asked to write:

```
moleculo: build failed: No space left on device (os error 28)
```

One line, exit status 1, **and nothing left behind**: the half-written staging
directory is removed, so the space it was holding comes back and a retry fails
or succeeds on its own merits rather than on the wreckage of the last attempt.

⚠ Both halves of that were defects until this was measured. The staging
directory used to survive — on the disk that had just filled, it kept every byte
it had written, leaving zero free and a retry that failed for a *different*
reason. And the error used to be followed by fifty lines of usage text, which
answers "what should I have typed" and is noise against "your disk is full".

⚠ **A build you kill is a different case, and it does leave its staging behind.**
That cleanup runs when a build *fails*; a process stopped by a signal does not
get to run it. So `Ctrl-C` on a long build leaves a `<name>.building` directory
holding whatever it had written — gigabytes, at the scales where killing a build
is tempting. Nothing is lost and nothing needs deleting by hand: **the next build
to the same destination removes it**, so a retry reclaims the space. Measured,
because "it cleans up" and "it cleans up when killed" are different claims.

**Size the disk for the finished index** while a build runs, plus the shard it
replaces if you pass `--replace`: a build writes nothing temporary, and the
staging directory is the shard being built. ⚠ The previous index format needed
about 1.6 times that, because its screening postings were spilled as sorted
runs beside the index — 6.3 GB beside a 10.7 GB index at 124 M molecules.


### In a container

The image carries **the binary from the release**, unpacked — it does not
compile its own. What runs in your cluster is byte-identical to what you would
download, including the leak checking that keeps a builder's home directory out
of it.

```sh
docker build --build-arg VERSION=<version> \
             --build-arg TARGET=x86_64-unknown-linux-musl -t moleculo:<version> .

docker run --rm -p 8080:8080 \
  -v /srv/indexes:/databases:ro \
  moleculo:<version>
```

- **Databases are mounted, never baked in.** A molecule collection is your
  confidential property and has no business inside an image that gets copied
  between registries. Mount it read-only; the server never writes to it.
- **The process runs unprivileged** as uid 10001. It reads mounted indexes and
  writes nothing outside a build directory, so root buys nothing and costs you
  an argument with your security team.
- ⚠ **Nothing is fetched at build time or at run time.** Verified by running the
  image with `--network none` and searching inside it.

Every subdirectory of every root that holds a valid index is served as a
database, named after its directory. Several roots can be listed; two roots
offering the same database name is an error rather than a silent winner, because
the name is how the API identifies a collection.

An index that is missing, corrupt, or of an unknown format version takes **only
that database** out of service. The rest keep serving, and the reason is logged
by name. A database made of several shards goes further: losing one shard leaves
the others serving, with every affected search saying so — see
[when a shard is missing](#when-a-shard-is-missing).

The server listens on **loopback by default** and there is no authentication of
any kind. ⚠ `--bind` (or `MOLECULO_BIND`) will put it on another address, which
the container image does because inside a container loopback reaches nobody —
**that is the one place the port stops being the boundary**, so read
[Security](#security) before setting it anywhere else.

### Search limits

Every search runs under a wall-clock limit, **30 seconds by default**. A search
that reaches it stops, returns what it found, and says so: the response carries
a `warning` and its `recordsFiltered` is a floor rather than a total presented
as final. Nothing is silently truncated and called complete.

| option | meaning |
|---|---|
| `--max-seconds N` | wall clock per search. `0` removes it. |
| `--max-candidates N` | molecules per search. `0` removes it. Time is the bound that protects a server; a molecule count bounds work whose duration depends on the corpus. |
| `--max-concurrent-searches N` | searches running at once. Default one per core, since a scan already claims every core. |
| `--max-queued-searches N` | arrivals that may wait for a slot. Default matches the pool. |
| `--no-search-limit` | removes all of them. For offline work, not for a server anyone else can reach. |

**Searches queue rather than being refused for arriving together.** Past the
pool, a search waits for a slot for at most `--max-seconds`; one still waiting
then is refused with a `503` rather than left in the queue. Its own
`--max-seconds` starts when it is admitted, so waiting and working together can
take up to twice the limit. `time` in the response is the work; `requestTime`
is the work plus the wait, so a client can tell a busy server from a slow query. Past the queue depth a search is
refused on arrival, because waiting only to be refused helps nobody.

**Similarity is refused rather than truncated.** A partial substructure count is
honestly a floor and the hits in it are real; a partial similarity is not a
smaller answer but a wrong one, because `recordsFiltered` is defined as the
collection size and the histogram would become a sample of whichever molecules
happened to be scored first. If a similarity search does not fit in the limit,
the response is a `503` saying so — raise `--max-seconds` or search a smaller
database. Similarity searches that arrive together share one scan; each still
answers by its own limit, with its result or the `503`.

---

## Searching

```
GET|POST /dt/{database}/search
```

| parameter | meaning |
|---|---|
| `type` | `Substructure`, `SMARTS`, `Similarity` or `Formula` |
| `query` | the structure or formula |
| `qopts` | per-type options, below |
| `start`, `length` | paging |
| `fmt` | `json` (default), `tsv`, `csv` |

Several databases at once: `/dt/db1,db2/search`. Counts sum, every hit carries
the database it came from, and `partStats` breaks the total down per database.

### Query limits

A structure query is held to five limits, wherever it is read — substructure,
SMARTS, similarity and `identity`. A query past any of them is a
`400 Invalid Query` naming the limit, while it is read and before anything is
searched:

| limit | applies to | why |
|---|---|---|
| **4 096 atoms** | SMILES and SMARTS | ring perception and fragment ordering grow with the square of the size, before any deadline is read |
| **512 aromatic atoms**, counted as written and again after aromaticity is perceived | SMILES | finding a Kekulé form recurses once per pair of aromatic atoms |
| **256 levels** of nesting in one SMARTS expression (`;`, `,`, `&`, `!` and juxtaposition) | SMARTS | every walk over a query recurses once per level |
| **two levels of `$()`**: `$($(C))` is read, `$($($(C)))` is not | SMARTS | each level multiplies the work of the levels below it |
| **128 atoms** inside all of a query's `$()` together | SMARTS | a recursive search runs on the stack of the search that asked for it |

⚠ **These bound the server, not your chemistry.** Past them, one query held a
core for seconds to minutes before any time limit applied — a 4 002-atom
aromatic ring took 57 s — or overflowed a thread's stack, which stops the whole
server rather than failing one request. Of 2.9 M ChEMBL molecules, the largest
holds 855 atoms and the most aromatic 400 aromatic atoms. A collection is not
held to these limits: they apply to queries only.

### Do you already hold this compound?

```
GET|POST /dt/{database}/identity?query={smiles}
```

⚠ **The one endpoint here that is ours rather than Arthor's.** It returns the
rows holding the *same compound* as your query — the free acid, its sodium salt,
its lysine salt — and each says **how** it merged and what it set aside.

```json
{ "key": "43c6119de2e0654c…", "parent": "CC(=O)Oc(c1)c(ccc1)C(=O)O",
  "matches": [
    { "id": 12, "record": "CC(=O)Oc1ccccc1C(=O)O\tCHEMBL25",
      "mergedBy": "whole-molecule", "setAside": [] },
    { "id": 4471, "record": "CC(=O)Oc1ccccc1C(=O)O.NCCCC[C@H](N)C(=O)O\tCHEMBL22",
      "mergedBy": "largest-fragment", "setAside": ["NCCCC[C@H](N)C(=O)O"] }
  ],
  "complete": true, "examined": 5, "skipped": 442, "time": 7620 }
```

`complete: false` means `matches` is a **floor**, for one of two reasons. The
wall clock ran out, or a canonical search reached its work budget, either on
your query or on a candidate row. In the second case a `warning` field says
which. `key` is hex because a 128-bit key does not survive JSON's number type.
`null` means your query has no key, which is not a key of its own. Such an
answer always comes with `complete: false` and a `warning`, and an empty
`matches`.

⚠ **Do not retry a `key: null` answer as if it had timed out.** The budget is
counted, not timed, so the same query gives the same answer on any machine and
under any load. A floor with no `warning` ran out of time, and a longer
`--max-seconds` can change it. A `warning` names something that no amount of
time will change.

⚠ **It answers and never acts** — nothing is deduplicated, merged or deleted.
See [Known limitations](#known-limitations) for the two things to know before
relying on it, both of which matter.

### Substructure — "which of our compounds contain this fragment?"

The query is SMILES. The count is **exhaustive**: if a fragment occurs in
700 000 compounds, that is the number you get, not "more than 20 000".

```sh
curl -G 'http://localhost:8080/dt/mydb/search' \
  --data-urlencode 'type=Substructure' --data-urlencode 'query=c1ccccc1'
```

`qopts` narrows it, in any combination:

| letter | effect |
|---|---|
| `R` | lock ring systems — benzene stops matching naphthalene |
| `C` | lock chains — hexane stops matching cyclohexane |
| `Q` | lock formal charge |
| `I` | lock isotope |
| `S` | lock stereochemistry **out** — every atom the query did not already speak about must *not* be a stereocentre. ⚠ The opposite of what the letter suggests |
| `N` | **negate**: return the molecules that do *not* match. Not a lock — it does not rewrite the query |
| `G` | **ours, not Arthor's**: enforce the configurations the query draws, so a chiral query stops matching its mirror |

⚠ **`R`, `C`, `Q`, `I`, `S` and `N` are Arthor's**; `G` is an extension of ours
and Arthor answers `400` to it, which is why it is a letter rather than a query
parameter — an unknown parameter it *ignores*, so a client would have got a
different answer with no way to know.

**`geometry=1` is the same lock under a name of ours** — `&geometry=1` beside
`query=`, values `1` and `0`. Both spellings resolve to one lock before anything
searches. ⚠ **Reach for `qopts=G`**: point a client that writes `geometry=1` at
Arthor and it is quietly ignored, where `qopts=G` is refused with a `400`, and a
refusal is the answer you want there. The parameter exists because `G` is a
letter in *their* vocabulary and we guessed it was free: on the day they spell
`G` as something of their own, this is where the lock lives and nothing else
about your client changes. Asking for both at once in opposite directions —
`qopts=G&geometry=0` — is a `400` rather than a winner.

⚠ **Without `G`, a query that draws a stereocentre matches both enantiomers**,
which is what both Arthor and `RDKit` do by default. Asked on a 3 004-molecule
collection, Arthor answers 212 for a chiral query and 212 for its mirror;
`RDKit`'s default answers 207 and 207. `G` is how you say you meant the wedge.

⚠ **Every substructure search scans: index format 4 has no screening index.**
A 1.x shard carried a small index of the path features its molecules hold, and
since 0.16.0 a substructure search could use it to skip every row without a
rare feature the query had to contain. On an 8 476 919-row PubChem shard it planned **one of the 56 corpus
queries**, because the features ordinary queries need sit above the density
ceiling on a real collection, and it cost 12–17 bytes a molecule on the three
shards measured (two PubChem, one ZINC20), so format 4 drops it. ⚠ **A fused
ring was unscreenable by construction anyway** — naphthalene's short paths are
the same runs of aromatic carbon as benzene's.
Each shard still counts its features into its manifest: that is what explains
an empty answer.

### SMARTS — "this nitrogen, in this environment"

The query is SMARTS, for when SMILES is not specific enough: an aromatic
nitrogen in a six-membered ring, an atom with exactly two ring bonds, a halogen
that is not fluorine.

### Similarity — "find me analogues of this"

The query is SMILES; `qopts` names the measure instead of locks: `Tanimoto`
(default), `Dice`, `Dissimilarity`, `Manhattan`, or `Tversky(α,β)` with your own
parameters.

Every molecule in the database is scored — the response's `recordsFiltered` is
the collection size — and the response carries a 256-bucket histogram of the
score distribution, which answers "how similar is this collection to my query at
all", not only "what are the top hits".

That pass scores the fingerprint folded to the index's width (`--bits`), and
folding lets unrelated features share a bit — enough, for a query of a few atoms,
to rank first a molecule that shares nothing with it. So the first 1 000 rows
of the ranking are scored again from their unfolded features and ordered by
that score: what you read at the top is exact. A search over several databases
(`/dt/db1,db2/search`) is one ranking over all of them, with one such head —
the answer one database holding them all would give. Past row 1 000 the
order and the scores are the folded ones, on their own scale, and a page that
reaches there says so in `warning`. The histogram and the count are the folded
pass's. `Manhattan` is normalised by the width and stays folded throughout.

### Formula — "what do we have with this brute formula?"

Exact molecular formula, hydrogens included, order-independent: `C9H8O4` and
`O4C9H8` are the same query. The match is exact — `C9H9O4` finds nothing that
`C9H8O4` finds.

### When nothing matches, the answer says why

New in 0.6.0, and as far as we can tell nobody else does it: a structural search
that finds **nothing** comes back with an `explain` object instead of a bare
zero.

```json
"explain": {
  "summary": "no molecule in this database contains 1 of the fragments this query requires",
  "absentFeatures": ["Pu"],
  "activeLocks": [],
  "decisive": true,
  "tautomerForms": 2,
  "tautomerFormsComplete": true
}
```

Three things it tells you, and all three are things you cannot otherwise find
out without redrawing the query several ways:

- **which fragment of the query the collection simply does not hold** — often
  the whole answer;
- **which `qopts` locks were in force.** ⚠ This is the commonest invisible cause
  of a surprising empty result: `R` stops benzene matching naphthalene and `C`
  stops hexane matching cyclohexane, and a caller who left one on has no way to
  see that from a zero;
- **how many tautomeric forms your query has.** A substructure search is exact:
  draw the enol, and a collection that stores the keto form matches nothing. The
  count includes the form you drew, so `1` means there is no other and rules the
  question out. ⚠ `"tautomerFormsComplete": false` means enumeration hit its
  bound and the count is a **floor**, not a total.

⚠ **The tautomer count is a fact about your query and never about the
collection.** "This query has four forms" is arithmetic on the string you sent;
"another of its forms is in here" would be a statement about the molecules, and
this does not make it. The enumeration runs the reference's own 37 transforms
and agrees with it on **94.5%** of a ChEMBL sample, compared set against set.

It costs about 20 ms on an answer that already found nothing, and
`--no-tautomer-report` (or `MOLECULO_NO_TAUTOMER_REPORT=1`) switches it off
entirely — the catalogue is then never compiled and the two fields never
appear.

⚠ `"decisive": false` means a shard of the database cannot tell a fragment that
is *absent* from one that is merely *common*: its manifest does not record
which fragments were too common to count, or its feature vocabulary is not this
build's, or it records no features at all, as a shard with no molecules does.
Every shard 2.0 builds or converts records which fragments were too common, so
outside an empty shard this comes from a shard written some other way. To such
a shard "absent from the index" and "in almost every molecule" look alike.
**`absentFeatures` is then empty and the summary says why** — a field named for
absence does not get to carry a guess. Rebuilding that shard is what makes the
answer decisive; nothing else about it is affected, and the locks are reported
either way because they are a fact about the query rather than about the
collection.

⚠ It costs an index lookup, not a second search — and a search that **found**
something never computes it at all. The field is absent from every other
response, so a client written against Arthor's shape never meets it.

---

## Getting results out

### A search may take more than one request

This is the part that surprises everyone, including the people who wrote it.

A large scan returns **partial results** with `"hasMore": true`. Repeat the
identical request until it comes back `false`; only then have the counts
converged. Reading `recordsFiltered` from the first response gives you a partial
count that looks exactly like an answer.

The second column is the **row id**: `(shard << 32) | local`, unique within a
database and stable for an index generation. On a single-shard database — the
default — it is simply the row's position, so nothing about it looks new. On a
database of several shards the rows of the second shard start at 4 294 967 296,
which is that arithmetic rather than a large number for its own sake.

⚠ **It is not a key across rebuilds**, and it is not what to write down: a row id
means something only inside one index generation, so anything you keep — a
selection, an export, a list to delete — should carry the `Identifier`. The
`index delete` command takes identifiers for the same reason.

⚠ **This diverges from the reference deliberately.** Arthor puts a position
*within an index part* there, which repeats across the parts of one database —
measured on its own `REAL-Database-22Q1`, where index 3 appears twice in a
single answer. Our page keys a marked molecule by that column, and while it
repeated, marking a row could silently unmark a different molecule.

A complete client is a dozen lines:

```python
import json, time, urllib.parse, urllib.request

def search(base, database, query, kind="Substructure", qopts=None, length=1000):
    params = {"type": kind, "query": query, "length": length, "draw": 1}
    if qopts:
        params["qopts"] = qopts
    url = f"{base}/dt/{database}/search?" + urllib.parse.urlencode(params)
    while True:
        body = json.loads(urllib.request.urlopen(url, timeout=300).read())
        if not body.get("hasMore"):
            return body
        time.sleep(0.5)

hits = search("http://localhost:8080", "mydb", "c1ccccc1")
print(hits["recordsFiltered"], "of", hits["recordsTotal"])
for _rank, _row, smiles, identifier, *_ in hits["data"]:
    print(identifier, smiles)
```

### Into a spreadsheet

Add `fmt=tsv` (or `csv`) and the response is a table of SMILES and identifiers
that opens directly in Excel or pandas.

### Into chemistry software

Results are SMILES, which RDKit, Open Babel and most toolkits read directly.

Add `fmt=sdf` and the response is an MDL V2000 SD file with generated 2D
coordinates — the format every drawing program and registration system opens.
Stereochemistry is expressed the way a molfile expresses it: wedge and hash
bonds for stereocentres, and the geometry itself for double bonds.

The program line of each record says `moleculo` and the time it was written.
Arthor stamps its own name there; putting *that* name on a file this program
wrote would be a claim about its origin that is not true, and nothing in the
format reads the line.

⚠ **A molfile is a drawing, and that is what makes this exact rather than
approximate.** Nothing in the file *says* cis or trans; a reader works it out
from where the atoms sit. So the coordinates are checked against `RDKit` over
five populations rather than assumed: of **76 094** molecules carrying
stereochemistry, **76 086** agree, **8** state less than they could, and **0**
state a different isomer. Which populations, the 8, and the one class that can
still state a wrong isomer are under [Known limitations](#known-limitations).

---

## Operating it

### Adding data without a manual build

Point the server at a folder of `.smi` files and give it somewhere to write:

```sh
moleculo serve ./sources --build-dir ./indexes --enable-reload
```

A `chembl.smi` in any root with no `chembl` index yet is built into
`./indexes/chembl` and served. The build directory joins the roots
automatically, so the sources themselves may be read-only — a mounted share
usually is.

Building is **off unless `--build-dir` is given**: it saturates a core for as
long as it takes, and a serving process should not be made to do that by
accident. On a busy server, build elsewhere and copy the finished index in.

### Picking up new data without a restart

`--enable-reload` adds `POST /admin/reload`. It rescans every root, builds any
new sources, and swaps the catalog. Searches already running finish against the
data they started with.

```sh
cp new-catalogue.smi ./sources/
curl -X POST http://localhost:8080/admin/reload
```

Or let the builder do it, from anywhere:

```sh
moleculo index build vendor.smi ./indexes/vendor \
  --notify http://localhost:8080/admin/reload
```

Reload is off by default: it can put a different database behind a name a client
is already using, which is an operator's decision rather than a caller's.

### Sending a collection over HTTP

With `--enable-upload` and `--build-dir`, a collection can arrive as a request
body instead of a file copy:

```sh
curl -X POST --data-binary @new-catalogue.smi \
  "http://localhost:8080/dt/upload?name=vendor"
```

```json
{"name":"vendor","bytes":184320,"built":false,
 "message":"stored; it becomes searchable after the next reload"}
```

⚠ **`"built": false` is the part to read.** The file is stored where a
hand-copied one goes; it does **not** build here. A 124 M build is hours, and a
request that runs for hours looks like a hung server to everything watching it —
so it becomes searchable at the next reload, exactly as a copied file does.

⚠ **A name already in use is refused, never replaced.** Overwriting a collection
somebody is searching, from a request with no authentication in front of it, is
not an upload.

### Warming an index before you need it

The first search against a cold index pays for reading it off disk — an hour, at
124 M. With `--enable-index-control` you can ask for that in advance:

```sh
curl -X PUT "http://localhost:8080/dt/mydb/data?idxtouch=SUB"   # substructure
curl -X PUT "http://localhost:8080/dt/mydb/data?idxtouch=SIM"   # similarity
```

The response is the database's description, including `memInfo`, so you can see
how much arrived.

⚠ **Warming is advice, not a command** — the kernel may honour it now, later, or
never, and on a machine short of memory "never" is the usual answer. ⚠ And it is
one of the few things here by which a careless client hurts *its neighbours*
rather than itself: warming a 124 M index pulls 10.5 GB into the page cache and
can push everything else on the machine out of it.

⚠ **`idxevict` works on Linux and cannot work on macOS**, which the server says
rather than pretending. Dropping a file's pages needs `posix_fadvise`, which
macOS does not have; the call that looks like it should work, `madvise`, returns
success and leaves every page exactly where it was. Measured, both platforms.

`?idxmigrate=`, `?name=` and `?resolver=` are refused with a reason: what
migrating an index means is undocumented, and a name or resolver changed at
runtime would not survive a restart.

### Replacing a database

```sh
moleculo index build updated.smi ./indexes/mydb --replace --generation 2
```

A higher generation is required, not optional. Row identifiers are stable only
within one generation, so reusing the number would leave one identifier with two
meanings and no way for a client holding identifiers across the swap to notice.

### Adding to a database without rebuilding it

```sh
moleculo index append ./indexes/mydb new-compounds.smi
```

The new molecules become one more shard of the database. **Nothing already
there is touched**, and the result is the index a full rebuild of both inputs
would have written — **byte for byte, all ten files of every shard**, measured
on the first 200 000 lines of ChEMBL cut in two, and served side by side the two
answer 56 of 56 corpus queries the same. Row identifiers of existing molecules
do not move, so a client holding them across the append keeps valid ones.

⚠ **A database built as a single shard is converted, without copying it.** That
is the form `index build` writes by default, so it is probably the form yours is
in. `append` gives it the several-shard layout by *hard-linking* the existing
files into place — they are not duplicated, which matters at eleven gigabytes —
and swaps the result in with one rename. At every instant a reader could look,
the directory is a valid database.

A running server sees the new shard at its next reload; pass
`--notify http://localhost:8080/admin/reload` to ask for one when the append
finishes.

**Run one append per database at a time.** It takes no lock, as `index migrate`
does not either; a second append started before the first finishes can report a
failure that in fact succeeded, and a retry on that report duplicates a shard.

**If an append of a single-shard database is killed mid-swap**, the next append
finishes it — but only when what it finds is exactly what an interrupted append
leaves: `mydb` absent, `mydb.appending.building` the complete new layout, and
`mydb.old.building` the original shard, the two still hard-linked file for file.
Anything else — a stale staging beside something a later build parked under the
same name, a shard that was deleted from while the append ran, a layout that
does not open — makes it **refuse, name what each directory holds, and move
nothing**; you decide which copy is the database and `mv` it to `mydb`. An
append whose result does not open complete puts the original shard back and
fails, keeping the new layout at `mydb.appending.building` for inspection; the
next append discards it and builds again. Converting a single shard needs file
identity (device and inode) to prove the swap, so on a platform without it the
append refuses before touching anything: rebuild with `--shards` instead.

### Deleting rows

```sh
moleculo index delete ./indexes/mydb --id CHEMBL25 --id CHEMBL1201585
moleculo index delete ./indexes/mydb --ids-file withdrawn.txt
```

Name molecules **by the identifier in your input** — the second column. ⚠ **Not
by row id**: a row id means something only until the next rebuild, so a list of
them that outlives one would delete different molecules and say nothing.

```
  shard 0    5025 newly deleted, 5025 deleted in all
  shard 1    4976 newly deleted, 4976 deleted in all
10001 rows deleted across 2 shards
a running server picks this up at its next reload
```

An identifier that matches nothing is **named** in the output rather than
counted, because it is usually a typo or the wrong database. Deleting a row that
is already deleted changes nothing and says **0**. ⚠ An identifier your
collection holds twice deletes both, and the count says so.

**A deleted row is gone from every answer**: searches, similarity, formula,
identity and every export. Over the first 200 000 lines of ChEMBL — 199 997
molecules — with 10 001 rows deleted, the database answers **56 of 56 corpus queries identically to the
same corpus rebuilt without them — in every field**, hits, identifiers and
counts alike.

⚠ **`recordsTotal` counts the rows a client can still retrieve**, not the rows
the index was built from. It has to: for a similarity search Arthor promises
`recordsFiltered` equals `recordsTotal` and the histogram sums to it, and
counting built rows would break both at once. `moleculo index inspect` shows the
built count beside the live one, and the difference is what a rebuild would
reclaim — **deletion frees no disk space until then**.

⚠ **There is no HTTP route for this, deliberately.** The server has no
authentication (see [Security](#security)), and a destructive route in front of
it would be a delete button for anyone who can reach the port. Delete on the
machine that holds the index; the server picks it up at its next reload.

### When a shard is missing

A sharded database whose shard directory has gone — a disk that failed, a copy
that did not finish — **still serves the rest**, and says what it lost. The
server logs it at startup, `/dt/data` drops the missing part from `partCnt`, and
**every search over that database carries a warning naming the shard**:

```json
"warning": "mydb: incomplete: 1 of 8 shards unavailable — shard 3 (missing);
            counts and hits cover only the rest"
```

Read that warning. `recordsTotal` falls *with* the missing shard, so a degraded
answer is internally consistent and looks complete — the warning is the only
thing that tells you the collection searched is not the collection you asked
about. A database whose every shard is gone is refused outright rather than
served as an empty one.

**A shard found damaged while searching is treated as a missing one.** An
index opens without reading its stores end to end, so a damaged block in the
middle of `fp.col` or `mol.pack` is found by the first search that walks over
it. That search does not return what it had read up to that point. The server
logs the file, the block and what was wrong, once, at error level. It serves
the shard as missing from then on and answers the same request from the rest,
with the warning naming the shard (`shard 3 (corrupt: fp.col block 41: …)`).
**After you rebuild the shard**, it serves again at the next reload
(`POST /admin/reload`, with `--enable-reload`) or the next restart. The server
recognises the new files because a rebuild gives them a new identity on disk, so
there is nothing to clear by hand. A file restored from a backup counts as new
if the restore puts a new file in place: removing the damaged one first,
renaming into place, or `rsync`'s default. ⚠ Copying over the existing file
with `cp -p` keeps its inode and its timestamps, so it looks unchanged, and only
a restart serves it again.

⚠ **Until 1.1.0 a damaged block made a search stop and answer as though
complete**: one overwritten byte made a similarity search score 2 003 of
3 004 rows with no warning. **Since index format 4 (2.0), damage that leaves a
record's length intact is caught too**, such as a flipped bit in a fingerprint,
which in 1.x read as a different fingerprint and moved the scores. Every store
carries a checksum that is verified as it is read: `mol.pack` a CRC-32C per
block, `fp.col` its zstd frame checksum or, under `--fp-codec none`, `fp.sum`,
`popcount.col` its `popcount.sum`, `smi.blk` its zstd frame checksum or, under
`--codec none`, `smi.sum`, and the offset tables and `deleted.roar` a CRC-32C
each, checked when the index opens. A record's block is checked every time it
is read — for a hit on a page, an identity lookup, or a pass over the whole
store — so a damaged record is never rendered as a different one. A raw
`smi.blk` from a 2.0 pre-release, built before `smi.sum`, is refused as "must
be rebuilt". [`INDEX-FORMAT.md`](INDEX-FORMAT.md) lists every checksum.

### A database on more than one machine

A collection that does not fit one box is served as **parts**: every part is an
ordinary `moleculo serve` holding some of the database's shards under the same
name, and one server — the coordinator — names them and answers for the whole.
A client cannot tell, except that `partStats` is keyed by part:

```sh
# on each part, its own shards under the database's name
moleculo serve /indexes                                  # holds big/shard-0000..0007
moleculo serve /indexes                                  # holds big/shard-0008..0015

# on the coordinator, no shards of its own, the parts in shard order
moleculo serve /empty --part big=one@http://one:8080 --part big=two@http://two:8080
```

The same list can live in `<root>/big/parts.json` as
`{"parts":[{"name":"one","url":"http://one:8080"},…]}`; a flag wins over the
file. A shard directory is a database of one shard and keeps its shard id, so
copying `big/shard-0009` to a part as `big/` keeps every row id global and the
hits in the order one server would return them. **List the parts in shard
order; that promise cannot be checked from the coordinator**, because the wire
carries a row's index within its shard and not the shard.

What the coordinator does: forwards the request to every part at once over the
same protocol you use, sums the counts, concatenates the rows in part order —
or merges a similarity by score — and pages the result. Polling works as it
does on one server: `hasMore` holds until every part has settled. A part's own
`--max-seconds` bounds its search; the coordinator adds a connect timeout of
five seconds and a total timeout of its own deadline plus five.

⚠ **A similarity's ties can sit in a different order than one server gives
them.** The wire carries a score to three decimals and a single server ranks by
the exact one, so among rows the coordinator cannot tell apart it orders by row
index. The rows on either side of a tie, the set inside it, the scores and the
histogram are the same; a `length=100` ranking on world-drugs showed 22 rows in
seven ties swapped inside their tie and nowhere else — and a page that ends
inside a tie can show different members of it, for the same reason.

**A part that does not answer** — down, timing out, refusing — makes the answer
*incomplete* in exactly the shape a missing shard does: the count falls with
the part, `verified` says how much was covered, and the warning names it:

```json
"warning": "big: incomplete: 1 of 2 parts unavailable — two (http://two:8080): timeout: global"
```

Two answers cannot be partial and are refused instead: a **similarity** (a
ranking over some of the collection is not a ranking, `503`) and a **negated
search** (the complement of a floor holds matching rows permanently, `400`).
Warming, evicting, uploading and reload are done on each part, and the
coordinator says so when asked. A request naming a database of parts beside
another database is refused: name it alone.

⚠⚠ **A PART HAS NO AUTHENTICATION, EXACTLY LIKE THIS SERVER, AND ANSWERS TO
WHOEVER CAN REACH IT. THE PARTS MUST BE REACHABLE ONLY FROM THE COORDINATOR.**
That is a network boundary you draw, not something this software enforces, and
the coordinator prints the same sentence at startup for every database of
parts it serves. Query strings travel to the parts over plain HTTP, as they
travel from your client to the coordinator.

⚠ **Measured on one laptop, as two processes; not on two machines.** The
property that was tested is that the answer does not depend on where the
shards live — every corpus query, a page across the seam between two parts, a
similarity's ranking and histogram, an identity, and the three text formats,
all identical to one server holding both shards. Nothing about throughput on
two machines has been measured, and this README does not claim a number for it.


⚠ **Run one build everywhere, and the coordinator tells you when you have not.**
Every server stamps its version on each response (`x-moleculo-version`), and a
coordinator that receives an answer from a part on another build says so in the
`warning` of every result it merges, naming the part and both versions. It
**warns rather than refuses**, so upgrading machines one at a time keeps the
database serving — but read that warning: an older part answers by older rules,
and the one that bites is silent. A 0.23.0 part ignores `geometry=1` entirely
and returns the unlocked answer, which is 207 rows where this build returns 101.
A part older than 0.24.0 stamps nothing at all, and the coordinator names it for
that too.
### Stopping it

`SIGTERM` or `SIGINT` (Ctrl-C) **drains** the server rather than killing it:

1. **announce**: for `--shutdown-delay-seconds` (default **0**), `/readyz`
   answers `503` and `moleculo_draining` reads `1`, while connections are still
   accepted and served;
2. **close**: no new connection is accepted;
3. **finish**: what is in flight completes, for at most
   `--shutdown-grace-seconds` (**30 by default**, a search's own deadline);
4. the process exits `0` and logs either that every connection closed on its
   own, or that the grace ran out and how many requests were still being
   answered. ⚠ Read that count as a floor: a client that has sent half a
   request, or is still downloading a large result, holds a connection with
   nothing being answered, and it is closed too.

A **second** signal exits at once with status `130`, for when you meant it.

⚠ **With no announce phase, readiness never fails where anything can see it.**
The listener closes the moment the drain begins, so a probe gets *connection
refused* rather than a `503`. That still takes the server out of rotation, and
under Kubernetes it is all you need (below). A load balancer that learns only
by polling `/readyz`, such as an HAProxy or nginx health check, should get
`--shutdown-delay-seconds` set to its check interval times the failures it
waits for. It then stops sending work before anything is refused.

⚠ **Your supervisor must wait longer than delay plus grace, or it kills the
drain.** Docker waits 10 seconds by default, so use `docker stop -t 40` or
`stop_grace_period: 40s` in Compose. Kubernetes waits 30, so set
`terminationGracePeriodSeconds: 40`. systemd's default of 90 is already enough.

⚠ **What a stop does not keep**: a structure search that is still converging
across polls (`hasMore: true`) lives in this process's memory, so a client
polling across a restart starts again from the first poll. That is the
protocol working as designed, not data loss, because nothing is stored.

### Probes: `/healthz` and `/readyz`

| route | answers `200` when | use it for |
|---|---|---|
| `GET /healthz` | the process is up and its runtime answers | liveness: restart if this fails |
| `GET /readyz` | at least one database is served and no stop has begun | readiness: send traffic only when this passes |

During a stop's announce phase `/readyz` answers `503` with `"status":
"draining"`, and `/healthz` still answers `200`: stopping is not dying, and
a liveness failure then would only get the drain killed.

Both are always on, cheap, and outside `/dt/`, so they cannot collide with the
Arthor API. ⚠ **A degraded database does not fail readiness.** It is listed in
the body (`"degraded": ["mydb"]`) and the rest keep serving, because one
missing shard must not take every other database out of rotation. Alert on
`moleculo_database_degraded` instead.

⚠ **Neither probe waits behind a search.** Searches, reloads and uploads run on
threads of their own, so a server whose every search slot is busy still
answers its probes in milliseconds. Through 0.24.0 it did not: a full pool held
every thread the runtime had, and a liveness probe would have failed on a
busy, healthy server. ⚠ One limit remains, and only under `--no-search-limit`:
with no admission pool, more than 512 searches at once fill the runtime's
blocking threads, the rest wait where no metric can see them, and a reload or
an upload waits behind them. The pool is the bound, which is one more reason
not to remove it on a server anyone else can reach.

The container image has a `HEALTHCHECK` on `/readyz`. A server started with
`--build-dir` builds any pending sources **before** it listens, which can take
hours on a large collection, and it reads *unhealthy* until the build is done.
In Kubernetes, cover that with a `startupProbe` rather than a longer liveness
timeout:

```yaml
containers:
  - name: moleculo
    image: moleculo:<version>
    ports: [{ containerPort: 8080 }]
    startupProbe:
      httpGet: { path: /healthz, port: 8080 }
      periodSeconds: 10
      failureThreshold: 360        # an hour to open, or to build at startup
    livenessProbe:
      httpGet: { path: /healthz, port: 8080 }
      periodSeconds: 10
    readinessProbe:
      httpGet: { path: /readyz, port: 8080 }
      periodSeconds: 5
    lifecycle:
      # Runs before SIGTERM. Kubernetes starts removing a terminating pod from
      # its Service at the same moment, and the pause keeps this server
      # answering until every proxy has heard. Use this or
      # --shutdown-delay-seconds, not both.
      preStop: { exec: { command: ["sleep", "5"] } }
terminationGracePeriodSeconds: 40
```

A systemd unit that drains on `systemctl stop`:

```ini
[Service]
ExecStart=/usr/local/bin/moleculo serve /srv/indexes --port 8080
KillSignal=SIGTERM
TimeoutStopSec=40
Restart=on-failure
User=moleculo
```

### Metrics: `/metrics`

Prometheus text format, always on, on the same port as everything else:

| metric | type | labels |
|---|---|---|
| `moleculo_build_info` | gauge | `version` |
| `moleculo_start_time_seconds` | gauge | |
| `moleculo_draining` | gauge | |
| `moleculo_http_requests_in_flight` | gauge | |
| `moleculo_http_requests_total` | counter | `route`, `method`, `status` |
| `moleculo_http_request_duration_seconds` | histogram | `route` |
| `moleculo_searches_total` | counter | `type` (`substructure`, `smarts`, `similarity`, `formula`, `identity`, or `unknown` for a `type=` that is not one of them), `status` |
| `moleculo_search_slots` | gauge | `state` (`running`, `capacity`) |
| `moleculo_search_queue` | gauge | `state` (`waiting`, `capacity`) |
| `moleculo_database_degraded` | gauge | `database` |
| `moleculo_database_rows` | gauge | `database` |
| `moleculo_parts_databases` | gauge | |

⚠ **`route` is the template, never the path**: `/dt/:database/search`, not
`/dt/chembl/search`. A request naming a database nobody holds cannot create a
label, and neither can an invented HTTP method: `method` is one of the seven
standard ones or `other`. `database` appears only where the catalog holds that name, so the label
set is bounded by what you serve, not by what clients send. **No metric carries
a query**, and the names are part of the 2.x promise.

What to alert on: `moleculo_search_queue{state="waiting"}` near its capacity (a
`503` is next), `rate(moleculo_searches_total{status="503"}[5m])` above zero,
and `moleculo_database_degraded == 1`.

### Logging every request

`--access-log` (`MOLECULO_ACCESS_LOG=1`) writes one line per request under the
target `moleculo::access`:

```
INFO moleculo::access: request method=GET route=/dt/:database/search database=chembl status=200 elapsed_ms=41
```

⚠ **It never carries the query.** It logs the route template, the database
segment of the path as the client sent it (still percent-encoded, so a newline
cannot forge a log line), the method, the status and the time. It reads nothing
else: not the query string, not a header, not the body. That is checked by a
test that drives real searches and finds the query in the log **zero** times.
Off by default, because the default is one event per *state*, and a line per
request is a volume you opt into. `RUST_LOG=error` silences it along with
everything else.

### Watching it from the outside

New in 0.6.0. Operator events go to **stderr** as structured lines, so stdout
stays pipeable:

```
WARN moleculo_server::routes: refusing a search on arrival: the pool is full
     databases=mydb refusal=every search slot is busy…
WARN moleculo_engine::catalog: database is serving degraded
     databases=mydb warning=incomplete: 1 of 8 shards unavailable — shard 3
```

`RUST_LOG` takes the usual filter grammar — `RUST_LOG=moleculo_engine=debug` for
one component, `RUST_LOG=error` to quieten it. The default is `info`, which is
every operator message and none of the noise.

⚠ **Query structures never appear in a log, at any level.** They are treated as
confidential intellectual property and a log file is the easiest place to leak
one. Only `--access-log` logs a request, and it never logs the query; the
databases involved are named where the load matters. That is checked rather than intended — a run driving searches against a
full pool was grepped for the query that produced them, and it appears zero
times.

⚠ Events fire **once per state, not once per request**. A degraded database
warns when it mounts, not on every search it answers; per-request detail is
there at `debug` if you want it. A warning per query on a large collection is
not a signal, it is a flood that hides the ones that fire once.

### How much of a database is in memory

`GET /dt/{database}/data` reports page residency per shard and per index, which
is what decides whether a query takes seconds or minutes:

```json
"memInfo": [{"index": {"type": "SUB", "partNum": 1, "partCnt": 8},
             "nodes": [{"incore": 10682, "total": 16366,
                        "aprx": true, "chart": "#######..."}]}]
```

Per shard rather than averaged, because a database is warm or cold in parts and
one figure would hide the single shard that is paging. `aprx` is always true and
honestly so: it is a snapshot of a page cache every process on the machine is
competing for.

---

## The search page

`moleculo serve` puts a page on the port it listens to. Open it and you get the
whole engine: a query field, the four search types, the collections you have
mounted, the `qopts` locks, and results.

**Draw, or type.** The `draw` button opens a structure editor. Drawing writes
into the query field, the field can be read back into the editor, and query
features drawn in it come out as SMARTS. The field is the source of truth, so
pasting a SMILES and drawing one end at the same place. The editor loads the
first time you ask for it and not before.

**Hits are drawn, not just spelled.** A query is usually written with aromatic
lowercase and a catalogue often stores Kekulé uppercase, so a perfectly correct
hit can look like nothing you recognise — `c2ccc(c1ccccc1)cc2` and
`C1=CC=C(C=C1)C2=C(C(=CC=C2)O)O` share a biphenyl that no one sees by reading.
Each row therefore carries a structure with its notation beneath it. Drawings
are painted as rows come into view, so a page of 250 costs what you actually
look at, and **settings** turns them off.

**Click a structure to see it properly.** The table draws small because it has
to fit hundreds of rows; a dialog opens the one you clicked at readable size,
with its identifier, its notation, a copy button, and arrows — or the left and
right keys — to walk the hits without going back to the table. Escape closes it.

⚠ **Stereochemistry in that drawing was wrong before 0.6.0**, and it was wrong
in a way nothing noticed for the life of the code: the mark travels inside a
notation this build writes, and the writer was reordering an atom's neighbours
without flipping the descriptor that is a parity over that order. The result was
**the other enantiomer** — same atoms, same bonds, same formula, same hit set,
so every test about constitution passed on both sides of it. Round-trip fidelity
on ChEMBL went from **73.82% to 99.990%**. Hit sets never depended on it, so no
index needs rebuilding for this; only what you were shown does.

**And the part that matched is marked on the drawing.** One occurrence per hit,
not all of them — the engine asks whether a query occurs, which is a cheaper
question than how many times. The mark travels inside the notation rather than
as a list of atom numbers, so nothing has to agree with anything else about how
a molecule is numbered.

⚠ The editor and the renderer are stored compressed, so a client that does not
send `Accept-Encoding: gzip` gets a `406` saying so rather than bytes it did not
ask for. Every browser sends it; `curl` needs `--compressed`.

**Settings**, top right, holds the three preferences that are yours rather than
the engine's: whether structures are drawn, how many rows a request asks for
(50, 250 or 1000 — each one is a real query against the collection), and whether
the page follows the system's light or dark setting or one you pick. All three
are remembered in your browser.

**The count says what it is.** While a search runs the page shows a running
total and says so; when a bound stops it, the number sits on a broken line and
is labelled a floor, with the reason; and a rail shows how much of the
collection was actually examined. A number without that state attached would be
a lie in one of three ways, and this page is careful not to tell it.

**A collection.** Star any hit and it is kept in your browser — structure and
identifier both, so it survives an index rebuild that changes row numbers — and
exports as TSV. It never leaves your machine; the server is not told.

**Sorting sorts what is loaded.** The page says so, in as many words. Sorting
250 loaded rows of ten million is not sorting the result, and pretending
otherwise is how a chemist comes to believe they are looking at the best hits.

The page needs no flag and no separate download. Everything it uses — including
the editor, all 65 files of it — is inside the binary.

---

## What this is measured against

No tool here is a straw man. Two of them are load-bearing — one decides whether
this is correct, the other decides whether it is fast enough — and it is worth
saying plainly what each is for and where this build comes off worse.

### RDKit — the correctness oracle

RDKit reads the corpus and moleculo reads the corpus and the two are diffed
field by field: formula, atom and bond counts, ring count, aromatic atoms,
aromatic bonds, and the per-atom ring-bond counts that ring locks compile to.
**99.992% of 2 897 819 ChEMBL molecules agree on every field, and 99.9999% of
5 200 000 from ZINC20** — two producers, different chemistry. The rest are
catalogued, with a reason each, rather than left as a percentage. On the ZINC
sample the catalogue is empty of anything this build owns: every aromatic
disagreement there is one where RDKit's answer depends on how the molecule was
written.

That relationship is not a comparison, it is a dependency: where the two differ,
the default assumption is that this build is wrong. The exceptions are written
down, and the largest of them is not a disagreement at all. On a furan bridged
into a macrocycle, **RDKit's answer depends on the order it happens to walk the
rings** — the same molecule, written two ways, gets two answers from the same
build. That is 186 of the 210 remaining divergences, and most of the other 24
are the same class caught by a stricter test. Arthor is stable there, and this
build follows it. ⚠ **What is left after that is not ours**: the two
aromatic-boron rings that used to head this list are read by Arthor exactly as
this build reads them — 0 aromatic, borazine itself included — so on those two
RDKit stands alone; and of the two fullerene adducts that read one atom short of
*both* references, one is fixed in 0.19.0 and the other waits on a ring-basis
decision the owner has taken the other way, under *Known limitations*.

RDKit is also the better tool for most jobs that are not this one. It reads and
writes every format, generates coordinates, computes descriptors, does
conformers and reactions. This build does four kinds of search and nothing else.
If your collection fits in a PostgreSQL database, the RDKit cartridge is a more
complete answer than this is.

### Arthor — the wire-compatibility target and the speed yardstick

**Why this API and not one of our own.** Not because anything is wrong with
Arthor — it is a current product, version 4.3.4 as of May 2026. Because its HTTP
surface is already wired into other people's tools and scripts, and that wiring
is worth more than any interface this project could design. A client written
against Arthor works here unchanged. Compatibility is not deference and it is
not a bid to replace anything; it is one concrete property — you do not have to
rewrite what already works.

⚠ **Everything measured below comes from the public demo instance at
`arthor.docking.org`, which serves 4.2.4 against the product's current 4.3.4.**
That is a demonstration deployment a minor version behind, not the shipping
engine, and the numbers should be read that way. It also explains two things
found while building against it: the swagger it publishes does not describe what
it does, and its answers have drifted from what the same queries returned in an
earlier session. Neither is evidence about Arthor as a product.

It is also the engine the performance bar is set by, and that bar was measured
rather than assumed:

| | Arthor demo (4.2.4) | moleculo 1.x, index format 3 |
|---|---|---|
| similarity, resident database | 3 300 M molecules/s at 1.647 B, 256-bit | 88 M/s default, **553 M/s** with `--fp-codec none` — 1.1.0, ten threads, 512-bit, 2.9 M, a noisy median (see above) |
| largest database served | 15.18 B molecules | 124.4 M verified end to end |
| substructure hit count | capped at 20 000 | exhaustive, or an honest floor |
| index size | not published | **92.5 bytes per molecule** — 124 427 347 molecules in 16 shards, 11.5 GB, built by 0.17.0 (⚠ this row read 84.3 for three releases, a figure no other document carried; the 0.16.0 screen costs +7.1% and the rest is the corpus) |

⚠ **That column has not been measured on 2.0.** Every figure in it was taken
from a 1.x release on index format 3. Format 4 has not been built at
124 M, so its size and build time at that scale are unknown; on the first
1 000 000 PubChem rows it reads 84.9 bytes a molecule (see [Building an
index](#building-an-index)). Similarity in 2.0 also re-scores the first 1 000
rows of each ranking exactly, which adds about 8 ms a search whatever the
database size: 54 → 120 ms for 8 queries over 500 000 PubChem rows.

⚠ **The index-size row is the one number a buyer asks first and it cannot be
compared**: Arthor publishes no such figure, so ours stands alone rather than
beside anything. It also replaces an earlier 86.4, which was **one shard on the
previous format** — a different measurement, not a correction.

**And a second table, because the first one is only about speed and size.** What
follows is not a benchmark. It is what the *answer* tells you, which is where
this build has spent its effort:

| when | Arthor demo | this build |
|---|---|---|
| a substructure count runs past 20 000 | caps and reports `hasMore: false` | converges, however long it takes |
| a limit cut the search short | no signal in the response | says so in `warning`, **and** gives the number in `verified` |
| a molecule is too branched to decide | counted as a non-match | counted apart, in `undecided` — and it now reaches `identity` and the tautomer count too, not just search |
| you negate a search whose count is a floor | answers | **refused** — the complement of a floor contains matching rows permanently |
| you negate a search over a row nobody could decide | counted as a non-match, so it is **returned to you as a row your query is absent from** | **refused**, naming how many |
| a similarity search is cut short | answers | **refused with `503`** rather than returning a histogram biased by what it missed |
| nothing matched | empty | `explain` says what was absent, and how many tautomeric forms your query has |
| you ask "do we already hold this?" | ⚠ **no such endpoint** | `/dt/{db}/identity`, saying how each row merged and what it set aside |
| your collection holds a wildcard atom and you search by formula | the row is unreachable — the wildcard is not in the formula | the formula carries it, `C7H6*NO`, and `*3` asks for exactly three |
| your query drew stereochemistry | ignored | ignored by default, honoured on `qopts=G` |

⚠ **Every row of that table is a *reported fact*, not a benchmark**, and several
of them exist because the honest answer was slower or emptier than the
convenient one. The line this build will not cross is a count that is a floor
presented as final.

⚠ **Read that first row with its asymmetry showing.** Their figure is a scan of
1.647 billion fingerprints, ours of 2.9 million — a working set of tens of
gigabytes against one of tens of megabytes. The comparison flatters this build,
not Arthor, and the gap at equal scale is the wider one.

**Where this build is behind: throughput and scale.** Similarity is **5.7x
slower with the fingerprint column uncompressed and 37x with the default
`zstd`**, and the codec is therefore the first thing to change if similarity is
what a deployment does all day. Of the 5.7x, about 2x is fingerprint width — 512
bits against their 256, chosen on recall rather than on speed — and the rest is
cores and memory bandwidth. A database can span machines as
[parts](#a-database-on-more-than-one-machine), but serving across two machines
has not been measured, so Arthor still serves collections a hundred times
larger than anything verified here. Building a single shard is partly parallel — on
index format 3 the two screening passes went 6.5x on ten cores and the whole
build 2.69x — and building shards at once goes past that: **8.17x on eight**,
which put 50 M molecules at 26 minutes 44 on that format.

**Where it is ahead: the count is real.** An Arthor substructure search stops at
20 000 hits. This one either returns the true count or says, in the response,
that what it returned is a floor and why. Nothing is truncated and presented as
final. Hit sets themselves agree where they can be compared — benzene on the
reference database returns 1973, and 1453 with ring systems locked, which are
the same two numbers and the same delta of 520.

**Three more things measured on that demo instance, offered as calibration
rather than as points scored.** Their engine is faster and serves collections
this one has not attempted; none of that is in question. But a public demo under
load is not a specification, and it behaves like one:

- **Concurrency costs more than the work does.** Three simultaneous searches,
  none of them expensive, produced two waits of **10.7 seconds against 6 and 19
  milliseconds of actual search**. Their contract exposes this honestly —
  `requestTime` includes queue time and `time` does not — and this build now
  reports the same split, because it is genuinely useful.
- **The demo instance has drifted from its own recorded behaviour.** A query
  verified at 135 hits against 135 in an earlier session now answers
  `recordsFiltered` 0 while returning five rows.
- **The specification that instance publishes is incomplete.** The compatibility
  contract behind this build was rebuilt from live probing and their own web
  bundle, because the swagger the demo serves does not describe what it does.
  The prepared-query echo is an undocumented private SMARTS dialect.

Where they are unambiguously better than the reference oracle: on a furan
bridged into a macrocycle, **Arthor gives the same answer every time and RDKit
does not.** That stability is why this build follows Arthor there.

And it runs on your machine. Queries never leave the process, no collection is
uploaded anywhere, and there is no service to depend on.

### Others in the same area

Not measured here, and listed so the map is honest rather than flattering.
**chemfp** is faster than this at Tanimoto and does only similarity — no
substructure search. **SmallWorld** answers a different question, nearest
neighbours by graph edit distance, which this cannot do at all. **Open Babel**
and its `fastsearch` index cover smaller collections with a broader toolbox.

### Which of these you actually want

No ranking is offered, because the honest answer differs by axis and a project
that ranks itself has chosen the axis. What follows is the question each tool
answers best.

| if the job is | use |
|---|---|
| similarity, as fast as it can be done | **chemfp** |
| reading, writing, descriptors, conformers, coordinates — chemistry generally | **RDKit** |
| nearest neighbours by graph edit distance | **SmallWorld** |
| billions of molecules, as a service someone else operates | **Arthor** |
| exhaustive substructure and SMARTS over hundreds of millions, self-hosted, where the count has to be trustworthy | **this** |

That last row is the narrow thing this is first at, and it is worth stating
plainly rather than leaving implied: a count here is either exact or labelled a
floor with the reason attached. It is one binary, it holds your collection on
your disk, and nothing it does reaches the network.

Everything else on the list does something this does not.

---

## Known limitations

- **This is a release candidate, and four things are not yet measured at the
  scale or against the reference they need.**
  - **Format 4's build time** has not been measured at scale — not at 124 M,
    let alone at a billion.
  - **Format 4's size** has been measured only on 1 000 000-row samples.
  - **The search speed 2.0 claims against 1.1.0** — 2.12× summed over the 56
    corpus queries, at least 10.83× on each of the ring queries `[R2]`, `[r5]`
    and `[r6]`, one thread — was measured on the first 200 000 rows of
    PubChem, on 2026-10-31, on the development tree of that day; the scan path
    has changed since. The bar names the first 1 000 000 rows, and that run
    is owed, on the release binary. Until then the figures are an indication.
  - **Arthor has not been compared with 2.0**, on time to the first page or
    on exhaustive throughput: the table under [Arthor](#arthor--the-wire-compatibility-target-and-the-speed-yardstick)
    is 1.x on format 3, and the public instance did not answer when last
    tried, on 2026-11-09.

  2.0.0 follows the 1 000 000-row run and the Arthor comparison.
- **Below row 1 000 of a similarity ranking, the order and the scores are the
  folded fingerprint's**, and they can read higher than the exact scores above
  them: folding lets unrelated features share a bit, and mostly inflates a
  score. On 500 000 PubChem rows, `COC`'s exact head ended at 0.000 with
  405 472 rows below it printing more. An answer whose page reaches past row
  1 000 says so in `warning`. The exact top 100 and top 250 lay wholly inside
  the head for each of the 8 queries measured there.
- **A query past the size and depth limits is refused**, with a `400` naming
  the limit — see [Query limits](#query-limits). The largest of 2.9 M ChEMBL
  molecules sits far below them, and they bound queries only, never a
  collection.
- **`fmt=sdf` can state less stereochemistry than it could, and on a fused
  macrocycle it may state a wrong isomer.** Measured against `RDKit` over five
  populations anyone can rebuild — world-drugs, real-space, mcule, and ChEMBL
  lines 1–100 000 and 100 001–200 000 (2026-09-08): **76 094 molecules with
  stereochemistry, 76 086 agree, 8 state *less* than they could, 0 state
  something false.** The 8 are bridged polycyclic cages — six daphnane-type
  orthoesters, one bicyclo[2.2.2] — and one crowded peptide, where the layout
  cannot spread a centre's neighbours far enough to read. ⚠ This read **134**
  until 2026-09-08, and the two mechanisms behind 126 of them were the
  writer's: a three-coordinate centre with a lone pair — a sulfoxide's sulfur,
  a bridgehead nitrogen — was never marked at all (78), and a cyclopropane
  fused on a ring was drawn with its apex *on top of* a ring atom (46). ⚠ The
  entry read *one in 78 935* from 2026-08-21, over
  two ChEMBL samples nobody recorded how to draw; the one was a 29-membered
  macrolactam whose ring shares 27 atoms with its neighbour — the layout refuses
  a fused macrocycle by construction — and it is not in the five populations
  above, so the class is real and its rate is not known. ⚠ **The 134 are
  losses, not lies**: a reader sees no configuration where it expected one,
  which is visible, rather than the wrong one, which is not.
- ~~**A hydrogen on a charged sulfur is dropped from an SD file.**~~ ⚠ **Fixed in
  0.9.0 and listed here as live for eleven releases.** A sulfoximine written
  `N[SH+]([O-])c1ccccc1` came back as `N[S+]([O-])c1ccccc1` until the atom
  line's valence field was written for a hydrogen the model cannot derive; it
  reads back as itself now, and has since 2026-08-24. This entry was found
  stale by re-checking every README figure whose population was never recorded
  (2026-09-06) — the same pass that found the bare-metal defect above. What the
  entry said about the mechanism was right; what it said about the state of the
  product was not.

  ⚠ **This entry replaces a wrong one, and the correction is worth more than
  the entry.** Until this entry replaced it, this section said that deuterium written as its own
  atom and radical electrons were *"lost when the SMILES is read, so they affect
  search as much as export"*. **That was false in its mechanism and therefore in
  its scope.** `[2H]C([2H])([2H])Oc1ccccc1` and `[O]N1C(C)(C)CC(C)(C)N1[O]` both
  survive a SMILES round trip untouched; nothing was ever lost on the way in.
  Both were defects in the *molfile writer*, which had no line for a mass number
  and none for an unpaired electron — `M  ISO`, shipped in 0.8.0, and `M  RAD`.
  If you read the earlier text and concluded that a deuterated compound could
  not be searched here, **it could, and it always could.**

  ⚠ **And that last sentence was itself too strong until 0.11.0, so it is
  corrected rather than quietly repaired.** The *model* always kept the
  deuterium — nothing was lost on the way in, which is what the paragraph above
  says and it is true. But **searching** for one did not work: inside a SMARTS
  bracket, `[2H]` compiled to *"isotope 2, carrying one hydrogen"* with the
  element dropped entirely, so a deuterium — which carries no hydrogens — failed
  a query drawn from itself, while the same pattern matched unrelated atoms like
  `[NH+]`. Seventeen two-letter element symbols went the same way: `[Rh]` was
  read as *"in a ring, with one implicit hydrogen"* and `[Rb]` as a
  contradiction that matched nothing. Fixed in 0.11.0, measured at
  **3 144 molecules in 497 335** of `PubChem` that could not be found by a query
  written from themselves.
- **A hit whose structure will not fit V2000 is left out of the SD file** rather
  than truncated: over 999 atoms or bonds needs V3000, which is not implemented.
  The omission is logged with the identifier.
- **The interface is one page and does four things.** Query, results, a local
  collection you can export, and a structure editor. There is no dashboard, no
  saved searches and no user accounts, and the API is the supported way to do
  anything the page does not.
- **The Arthor web UI has not been tested against this.** `/config` serves the
  blob it reads on startup, but nobody has pointed it here.
- **Duplicates are answered, never removed.** `/dt/{db}/identity?query=...`
  returns the rows holding the same compound as your query — the free acid, its
  sodium salt, its lysine salt — each saying **how** it merged and what it set
  aside. Nothing is deleted, deduplicated or rewritten, and that is a decision
  rather than an unfinished feature: a key merges 4.12% of ChEMBL and **99.88%
  of those merges go through the "largest fragment is the compound" rule**,
  which cannot tell a counter-ion from a second active ingredient. Shown to a
  chemist that population is informative; acted on, it silently deletes distinct
  substances. When *you* have decided a row should go, that is
  [`index delete`](#deleting-rows), by identifier — `identity` will tell you
  which rows share a compound and never act on it.

  ⚠ **A combination product answers for its larger drug and is missing from the
  smaller one's answer.** A row holding two actives files under whichever
  fragment is bigger. Ask about the bigger one and the row appears with the
  other named; ask about the smaller and **the row is not there**. In ChEMBL
  that is twenty-five combination antibiotics filing under the antibiotic and
  discarding tazobactam and sulbactam, which are drugs, plus twenty-four ionic
  liquids under a shared anion. Inherited from the reference's own rule; no
  endpoint design fixes it.

  ⚠ **Ask it about compounds, not about general structures.** Its per-row work
  is a standardisation rather than a scan step, so candidates are filtered by
  heavy-atom count first — asked for benzene against 2.9 M rows it rejects
  **2 440 588 candidates and standardises one**. Rejecting still costs a parse,
  so the loop carries the same wall clock a search does and benzene comes back
  at the deadline with `"complete": false`. Use `/search` for that question.
- **The identity key is not yet invariant to how a molecule was written, and on
  fullerene cages it essentially never is.** Hand `/dt/{db}/identity` the same compound
  spelled two different ways and **2 in 40 000 give two different keys** (the
  2026-08-28 draw; a 2026-09-06 draw at 25 spellings reads **0**, after one
  molecule that was not a cage — a nitronate whose spelling carried marks on a
  bond that cannot have a configuration — taught the key to ignore such marks), which
  means the endpoint can answer *"we do not hold this"* about a compound the
  collection holds.

  ⚠⚠ **That number is how often the affected molecules turn up in a random
  sample, not how often the defect fires.** Measured on the molecules it is
  actually about — every bare carbon cage in a 10 million-molecule slice of
  PubChem, 22 of them, at 24 spellings each — **21 of 22 give more than one key,
  and one gives thirteen.** The single exception is C60, whose symmetry makes
  the choice below irrelevant; break that symmetry, as C70 and every
  functionalised fullerene do, and the key moves. If your collection is
  fullerene chemistry, treat `identity` as unavailable rather than rare.

  ⚠ **This entry said 17 in 100 000 until 0.13.0, and both halves of that were
  wrong.** The number came from a check that drew **six** spellings per molecule
  and reported a *sampled lower bound* — a figure that cannot be compared
  between releases, because which molecules surface depends on the draw. Five of
  the causes it pointed at were one defect and are fixed; what is left is
  **fullerene chemistry as a class**, which a random sample of ordinary
  molecules will almost never contain.

  ⚠ **This is a price rather than a bug, both sides of it are measured, and
  the decision has been taken: the price is paid.** On a fused cage the ring
  *count* is stable — always the cyclomatic number, and always right — but the
  *choice* of rings is not, which is what SSSR's non-uniqueness means there.
  Mapping the rings of one spelling onto another atom by atom, with every
  automorphism tried, over the 22 cages at 24 spellings each: **this build's
  ring set depends on the spelling on 21 of 22, and keeping every shortest cycle
  through every ring bond — what RDKit's symmetrisation is built on — on 0 of
  22.** So the stable definition exists and we do not use it.

  ⚠ **This paragraph said "moves on 22 of 22" until 0.19.0, by an instrument
  that compared ring *shapes*, and 0.19.0 found the ring basis had been holding
  things that were not rings at all** — a walk with its root dropped from both
  ends, recorded as a ring because nothing checked that its last atom was bonded
  to its first. That was noise on top of the real instability, not the
  instability: fixed, the shape instrument reads 4 of 22 and the honest one
  reads 21 of 22. One cause remains on cages, and it is the tie.

  ⚠ **And it costs something on ordinary chemistry too, measured after the
  ruling rather than before it.** Over 40 000 PubChem molecules at 25 spellings
  each, **385 — about one in a hundred, a lower bound — have a ring set that
  depends on how they were written**: bridged bicyclics, adamantanes,
  quinuclidines, morphinans, anything where two equal-sized cycles tie. What a
  query could see of that through 1.x: **`[R3]` answered differently on 48 of
  the 385 depending on the spelling the *collection* stored**, `[N;R2]` on 15
  and `[R2][R2]` on 14. ✅ **Since 2.0, `R<n>` and `r<n>` no longer read the
  basis.** They count every relevant cycle (one that is not a sum of shorter ones;
  a family of exponentially many equal rings, as in a cyclodextrin, counts
  once), a set fixed
  by the structure: 0 of the 40 000 and 0 of the 22 cages move with the
  spelling. Against RDKit, the molecules where any atom's `R` or `r` differs fell
  from 255 to 5 of 22 423 across the three test corpora. In each of the five,
  RDKit keeps only one of two equal rings, so two symmetric atoms, or one ring
  system with and without a substituent, get different answers. What still reads the basis is
  aromaticity, the identity key and the 2D layout of an SD file, and the first
  two are what the rest of this entry is about.

  ⚠ **What using the stable definition would cost is why the owner decided
  against it (2026-09-03).** The ring count would change on **138 503 of
  9 999 789 PubChem molecules — 1.39%** — and with it the aromatic divergences
  listed below. (It would have moved every `R` and `r` query too; 2.0 moved
  those separately, as above.) On C60 the two definitions read 31
  rings and 32: the cyclomatic number against the 32 faces, which is Euler's
  relation and not a defect in either. **The ruling is to keep the minimal
  basis and declare this**: `identity` on fullerene chemistry is unreliable as a
  class, one fullerene adduct reads one aromatic atom fewer than both references
  because the ring its atom needs is a tie ring a minimal basis cannot hold,
  and neither changes until the ruling does.

  ⚠ **A further 45 in 500 000 were refused in one spelling and accepted in
  another; 39 were fixed in 0.17.0 and the last six in 0.19.0.** The class was
  aryl carbanions — phenyl anions, Grignards, aryllithiums — which PubChem writes
  Kekulé (`C1=CC=[C-]C=C1`), where we read them, and which RDKit writes aromatic
  (`[c-]1ccccc1`), where we returned a `400`. One structure, two answers, decided
  by how it was drawn. The six that remained were an aromatic ring holding a
  triple bond, aromatic silicon and an aromatic borate — three more cells of the
  same table, each run against RDKit. **The count is 0 of 499 971 now**, and the
  measurement is six seconds rather than a sample: every molecule we accept,
  rewritten by RDKit's writer, read back.

  Neither affects substructure, SMARTS, similarity or formula search, whose hit
  sets do not depend on the key.
- **Some rows cannot be decided, and now they say so instead of counting as
  misses.** Subgraph isomorphism is NP-complete, so the matcher abandons a
  molecule after a fixed amount of work rather than hanging your build on one
  row — a symmetric giant with a dozen interchangeable arms can outrun any
  budget. Those rows arrive in the response as **`undecided`**, counted apart
  from the misses, and the warning says the count is a floor. ⚠ **Earlier
  releases reported them silently as "your query does not occur here."** For
  calibration: matching every molecule against **itself** — the easiest query
  there is — **30 of 9 999 789** PubChem molecules exceed the budget. ⚠ This
  entry said "455 of 497 693" until 0.19.0; that was a sample nobody recorded
  and 150× out, and the seed fix in 0.15.0 had already taken the real figure
  from 59 to 30. Raising the budget is not offered as a fix, because it trades a
  silent wrong answer for a slow one.
- **Some structures we hand back as SMILES are read by other tools as a
  different molecule.** The `SMILES` column every result carries — in `json`,
  `tsv` and `csv` alike — is consumed by whatever you pipe it into, and over
  the first 500 000 lines of PubChem — 499 971 round-trip **exactly**, 27 are
  molecules this build refuses to read at all, and **not one comes back as a
  different molecule.**

  ⚠⚠ **This entry said "10 as a different graph and 12 unreadable" until
  0.14.2, and that figure cannot be rebuilt.** It was measured over "a 497 877-
  molecule PubChem sample" and nothing recorded how that sample was drawn; no
  prefix of the corpus reproduces it, and the same measurement on the 0.13.0
  build gives the same zero. The defect may well have been real when it was
  written down — what is certain is that **a number whose population nobody
  recorded cannot be checked, kept or retired.** Samples are now prefixes,
  built by `tools/oracle/sample.sh`, and every figure names its own. ⚠ **The
  molecule stored and searched here is correct**; it is the written form that is
  at issue. If
  you are round-tripping results through another toolkit, `fmt=sdf` carries the
  structure faithfully — see the next entry for what that took.
- **`fmt=sdf` had three defects and all three are now closed.** ⚠ **Until
  0.14.0 this README said it "does not have this problem", and it had them.**

  **A record no longer asserts a double-bond geometry your input never stated.**
  A molfile has no "unspecified" in its coordinates — a reader derives cis or
  trans from where the atoms sit — so the drawing answered a question nobody
  asked, on **9 751 records in 100 000**. The format's own way of saying
  "either" is the crossed double bond, and it is now written wherever the
  structure leaves the geometry open, matching RDKit bond for bond. Fixed in
  0.14.0; the count is **0**.

  **A record now states the valence of an atom whose default a reader cannot
  guess.** 15 molecules in 100 000 came back as a *different molecule* without
  it, and the reader's guess was wrong in both directions — `C[SnH3]` came back
  with one hydrogen instead of three, `[Te][Te]` with one each where it had none.
  Fixed in 0.14.0 and 0.14.1; **structures that come back as a different molecule
  are now 0 in 100 000.**

  ⚠ **That sentence was true of the PubChem sample it was measured on and false
  on ChEMBL until 0.20.1.** A bare metal — `[Na]`, `[Li]`, `[Cs]`, `[Ra]`, a
  lithium citrate written as three lithium atoms — has a total valence of
  zero, and the writer put `0` in the field. In a V2000 atom line a written
  zero means *not stated*, so a reader derives the valence from its own default
  table and RDKit's says sodium takes one bond: `[Na]` came back as `[NaH]`,
  `[Ra]` as `[RaH2]`, lithium citrate as three lithium hydrides. **Three records
  in a 100 000-line ChEMBL prefix, a different compound each.** V2000's code
  for zero valence is **15**, and it is written now for every atom outside the
  organic subset whose total is zero, charged or not. Measured after the fix
  over six populations — the three corpora, two ChEMBL prefixes and the PubChem
  sample — **0 records come back as a different graph**. ⚠ The PubChem sample
  holds no bare metal, which is why the sentence above stood for three
  releases; if you exported SD files with 0.20.0 or earlier and your collection
  holds bare metals, export them again.

  **A record no longer claims unpaired electrons an atom does not have.** This
  build expected a negatively charged phosphorus, sulfur or iodine to reach a
  *higher* valence than it does, so the shortfall was written as a radical:
  **every iodide counter-ion we have ever exported carried one — 1 189 records
  in 100 000.** Fixed in 0.14.1. Per atom against RDKit, claims no atom supports
  go from **1 189 to 0**, and nothing that was right became wrong.

  ⚠ **If you hold SDF files from 0.14.0 or earlier, export them again.** The
  coordinates never changed; what a reader takes from them did, three times.

  ⚠ One record RDKit still refuses to read is `F[Kr]F`, which we write correctly
  and its own valence model declines. Radical state on a *metal* is still not
  modelled here at all — 601 records in 100 000 state fewer unpaired electrons
  than RDKit does, every one of them on an atom outside the organic subset.
- ~~**Sharding is within one machine, not across machines.**~~ ⚠ **Closed in
  0.22.0** — a database may be served as *parts* on other machines, see
  [A database on more than one machine](#a-database-on-more-than-one-machine).
  What remains true: it has been demonstrated with several servers on one
  machine and never measured across two, and the parts must sit in shard order
  by the operator's hand — the coordinator cannot check it.
- **210 molecules in 2 897 819 of ChEMBL read a different aromatic system from
  RDKit** — three in 5 119 973 of ZINC20, where none of them is ours —
  and the composition matters more than the number. **186 of them are cases
  where RDKit's own answer is not a function of the molecule**: on a furan or
  thiophene bridged so that its oxygen also lies on a ring of nine atoms or
  more, its answer depends on which ring its walk reaches first, and the same
  graph read 5 aromatic atoms on 128 of 300 random writings of itself and 0 on
  the other 172. Arthor is stable there and agrees with this build; most of the
  other 24 are that class caught by a stricter test. ⚠ **The residue that used
  to be called "genuinely ours" — nine molecules — was asked of both references
  in 0.19.0 and is not**: the two aromatic-boron rings are read by Arthor as
  this build reads them, 0 aromatic, borazine itself included, so RDKit stands
  alone there; of the two fullerene adducts one is fixed and the other is the
  ring-basis ruling above; the five one-offs were the fusion bonds below.

  ⚠ **That entry counts aromatic *atoms*. Bonds are counted separately and were
  worse until 0.15.0.** Over 398 040 ring-fusion bonds in 500 000 PubChem
  molecules, **4 now disagree with RDKit where 145 did** — a bond can lie inside
  one aromatic ring combination and on the edge of another, and this build used
  to decide from the first combination it found and never revisit. 0.19.0
  closed the last five: the rescue skipped any ring combination whose rings
  were all already settled, and that is exactly the combination whose edge
  carries a fusion bond when two rings fail alone and a third passes. **The
  four that remain all sit in one tetra-azo molecule where Arthor reads every
  bond as this build does and RDKit alone differs.** 185 further disagreements
  are the same furan-in-a-macrocycle cases described above, where RDKit's
  answer is not a function of the molecule, and are left alone deliberately.
- **Only part of a *single shard's* build is parallel.** The pass that counts
  the screening vocabulary uses every core; reading the input, encoding the
  molecules and writing the fingerprints do not. Measured on index format 3, at
  ten cores and 124 M: the parallel passes went 6.5x, one whole build 2.69x.
  Building several shards at once is what gets past that — 8.17x on eight — and
  it is the reason `--shards` exists. ⚠ The two multiply for memory: eight
  concurrent format-3 builds measured **3.94 GB** peak, not the 1.7 GB a
  per-build figure predicts, because each shard build runs its own threads.

- **No canonical identity across a rebuild of different data.** Row ids are
  stable for an index generation; a rebuild that reuses them for different
  molecules must bump the generation, and nothing checks that for you.

---

## Security

- Query structures are treated as confidential intellectual property: never
  logged in full, never written outside the process, and sent to nobody —
  **except the parts a coordinator was told to serve**, over plain HTTP, which
  is why they must be reachable only from it.
- The software performs no outbound network communication other than to those
  parts, sends no telemetry and reports nothing to anyone.
- A malformed SMILES or SMARTS is a `400`, never a crash, and so is a query
  past the size and depth limits in [Query limits](#query-limits).
- Every search is bounded by wall clock, 30 seconds by default, and a search
  that hits the bound reports a partial count as partial. Searches also queue
  behind a bounded pool, so a caller issuing them faster than they complete gets
  a `503` rather than turning one deadline into a queue of deadlines.
- `/metrics`, `/healthz` and `/readyz` answer on the same port as searches.
  They name databases and count requests, which is no more than `/dt/data`
  already tells anyone who can reach the port, and **no metric or probe
  carries a query**. If your monitoring lives outside the perimeter, scrape
  through the same proxy that guards the rest.
- ⚠ **There is no authentication, and no per-client rate limiting.** The pool
  bounds the server's own workload; it does not bound *who* may send work. There
  is no notion of a user, a key or an allowed address anywhere in this build.
- ⚠ **Anyone who can reach the port can export the whole collection.** A SMARTS
  of `[*]` with `fmt=tsv` matches every molecule and streams it. That is not a
  defect in the search — it is what a search engine does — but it means the port
  is the security boundary, and there is nothing behind it.
- It listens on loopback by default. **Keep it there**, or put it behind
  something that authenticates and rate limits. That is the supported way to
  expose this to more than one trusted user, and it is not optional advice.
- ⚠ **The container image sets `MOLECULO_BIND=0.0.0.0`, and it has to.** Inside
  a container loopback reaches nobody, so an image that kept the default would
  publish a port that answers nothing. What that means for you is that **the
  container's boundary is the port you map and the network you attach it to**,
  not the process — `-p 127.0.0.1:8080:8080` keeps it on your host's loopback,
  and `-p 8080:8080` does not.

---

## Licence

Proprietary — see `LICENSE`. Source code is not distributed.

Third-party open source components and their licences are listed in
`THIRD-PARTY-LICENSES`.
