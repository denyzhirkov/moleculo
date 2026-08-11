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
- [Operating it](#operating-it) — adding data, several sources, reloading
- [The search page](#the-search-page) — draw a query, read the results, keep what matters
- [What this is measured against](#what-this-is-measured-against) — RDKit, Arthor, and where this loses
- [Known limitations](#known-limitations)
- [Security](#security)

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
| Linux x86-64 | `x86_64-unknown-linux-musl` |
| Linux arm64 | `aarch64-unknown-linux-musl` |

The Linux builds are statically linked against musl: no glibc version
requirement, and they run on any distribution with a matching CPU.

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

**How correct is the reading?** Every molecule is diffed against RDKit over a
fixed corpus of 2 897 819 ChEMBL molecules: **99.991% agree on every field**,
and the remaining divergences are catalogued rather than unknown. Of that
corpus, 7 molecules are refused where RDKit accepts them.

Refusals are the other direction of the same question, and they are measured on
the whole of PubChem rather than a sample: of 124 469 489 rows, 42 142 are
refused (0.034%), every one of them for a valence no element allows —
`FBr(F)F`, `O=Cl(=O)(=O)F` — and RDKit refuses them too.

---

## Building an index

An index is a derived artifact: reproducible from the input and the builder
version, safe to delete and rebuild, and never edited in place.

```sh
moleculo index build molecules.smi ./indexes/mydb
moleculo index inspect ./indexes/mydb
```

| option | meaning |
|---|---|
| `--bits N` | similarity fingerprint width: 256, 512 (default), 1024 or 2048. Narrower costs recall, wider costs disk. Fixed at build time — it is a property of the index, not of a query. |
| `--codec none\|zstd` | compress the molecule and record stores. Default `zstd`. |
| `--fp-codec`, `--screen-codec` | the same, for the fingerprint column and the screening index. ⚠ `--fp-codec none` is **4.6x the similarity throughput for about 46% more index** — measured, see below. |
| `--replace` | swap out a shard that is already there. Requires a higher `--generation`. |
| `--generation N` | index generation. Row identifiers are stable within one generation and only within one. |
| `--notify URL` | call a running server's reload endpoint once the shard is published. |
| `--threads N` | workers for the two screening passes, which are three quarters of a large build. Default: the machine's parallelism. Does not change the shard; does raise memory. |
| `--shards N` | split the input into N shards and build them at once. This is the parallelism that matters at scale — see below. |
| `--build-workers N` | how many shard builds run concurrently. Default: the machine's parallelism. ⚠ A safety bound, not a speed knob: memory is per-build and multiplies. |

**Plan for the time.** Building is CPU-bound and it is hours rather than minutes
at scale. Measured on a ten-core laptop:

| molecules | one core | ten cores | index size | bytes/molecule |
|---:|---:|---:|---:|---:|
| 10 M PubChem | 50 minutes | — | 0.92 GB | 91.6 |
| 124.4 M PubChem | 10 h 43 | **3 h 58** | 10.7 GB | 86.4 |

Note which way the density moved. Bigger collections cost *less* per molecule,
because the screening index compresses better when each feature has more rows in
it, so sizing from a small trial run overestimates rather than under. On ChEMBL,
whose chemistry is heavier, the same build is 129.5 bytes per molecule — the
range across the corpora tried here is 86 to 130.

Three quarters of that is the two screening passes, and they run on as many
workers as `--threads` asks for. Those two passes go **6.5x faster** on ten
cores — 8 hours to 74 minutes at 124 M. The whole build gains **2.69x**, because
reading the input, merging the posting runs and writing the fingerprints are
sequential and stay so.

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
own screening threads, so the two kinds of parallelism multiply. If the machine
is smaller, turn `--threads` down as well as `--build-workers`.

```sh
moleculo index build big.smi /data/big --shards 8 --build-workers 8 --generation 1
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
| similarity, ten threads | 84 M molecules/s | **385 M/s** |
| similarity, one thread | 10 M/s | 53 M/s |
| substructure scan | unchanged | unchanged |
| fingerprint column, 2.9 M molecules | 91 MB | 180 MB |

The substructure scan does not move because it never reads that column, which is
the check that says the rest of the table is real. At 124 M the trade is a
fingerprint column of 2.98 GB against 7.96, taking the whole index from 86 bytes
per molecule to 126. **Compression stays the default**; if similarity is what
your deployment does all day, `--fp-codec none` is the one flag that matters.

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

## Running the server

```sh
moleculo serve ./indexes [more/roots ...] [--port 8080]
```

Every subdirectory of every root that holds a valid index is served as a
database, named after its directory. Several roots can be listed; two roots
offering the same database name is an error rather than a silent winner, because
the name is how the API identifies a collection.

An index that is missing, corrupt, or of an unknown format version takes **only
that database** out of service. The rest keep serving, and the reason is logged
by name.

The server listens on loopback only.

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
pool, a search waits for a slot, and the wait is charged against its own
deadline — one that queued away its thirty seconds is refused rather than
admitted to do a token amount of work and report a truncated count. `time` in
the response is the work; `requestTime` is the work plus the wait, so a client
can tell a busy server from a slow query. Past the queue depth a search is
refused on arrival, because waiting only to be refused helps nobody.

**Similarity is refused rather than truncated.** A partial substructure count is
honestly a floor and the hits in it are real; a partial similarity is not a
smaller answer but a wrong one, because `recordsFiltered` is defined as the
collection size and the histogram would become a sample of whichever molecules
happened to be scored first. If a similarity search does not fit in the limit,
the response is a `503` saying so — raise `--max-seconds` or search a smaller
database.

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

### Formula — "what do we have with this brute formula?"

Exact molecular formula, hydrogens included, order-independent: `C9H8O4` and
`O4C9H8` are the same query. The match is exact — `C9H9O4` finds nothing that
`C9H8O4` finds.

---

## Getting results out

### A search may take more than one request

This is the part that surprises everyone, including the people who wrote it.

A large scan returns **partial results** with `"hasMore": true`. Repeat the
identical request until it comes back `false`; only then have the counts
converged. Reading `recordsFiltered` from the first response gives you a partial
count that looks exactly like an answer.

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
`fmt=sdf` is **not** available — see below.

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

### Replacing a database

```sh
moleculo index build updated.smi ./indexes/mydb --replace --generation 2
```

A higher generation is required, not optional. Row identifiers are stable only
within one generation, so reusing the number would leave one identifier with two
meanings and no way for a client holding identifiers across the swap to notice.

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
**99.991% of 2 897 819 ChEMBL molecules agree on every field.** The rest are
catalogued, with a reason each, rather than left as a percentage.

That relationship is not a comparison, it is a dependency: where the two differ,
the default assumption is that this build is wrong. The exceptions are written
down, and the largest of them is not a disagreement at all. On a furan bridged
into a macrocycle, **RDKit's answer depends on the order it happens to walk the
rings** — the same molecule, written two ways, gets two answers from the same
build. That is 202 of the 211 remaining divergences. Arthor is stable there, and
this build follows it.

RDKit is also the better tool for most jobs that are not this one. It reads and
writes every format, generates coordinates, computes descriptors, does
conformers and reactions. This build does four kinds of search and nothing else.
If your collection fits in a PostgreSQL database, the RDKit cartridge is a more
complete answer than this is.

### Arthor — the wire-compatibility target and the speed yardstick

The HTTP surface is Arthor's, deliberately, so that clients written against it
work here unchanged. It is also the engine the performance bar is set by, and
that bar was measured rather than assumed:

| | Arthor | this build |
|---|---|---|
| similarity, resident database | 3 300 M molecules/s at 1.647 B, 256-bit | 84 M/s default, **385 M/s** with `--fp-codec none` — ten threads, 512-bit, 2.9 M |
| largest database served | 15.18 B molecules | 124.4 M verified end to end |
| substructure hit count | capped at 20 000 | exhaustive, or an honest floor |
| index size | not published | 86.4 bytes per molecule |

⚠ **Read that first row with its asymmetry showing.** Their figure is a scan of
1.647 billion fingerprints, ours of 2.9 million — a working set of tens of
gigabytes against one of tens of megabytes. The comparison flatters this build,
not Arthor, and the gap at equal scale is the wider one.

**Where this build is behind: throughput and scale.** Similarity is **8.6x
slower with the fingerprint column uncompressed and 39x with the default
`zstd`**, and the codec is therefore the first thing to change if similarity is
what a deployment does all day. Of the 8.6x, about 2x is fingerprint width — 512
bits against their 256 — and the rest is cores and memory bandwidth. Sharding is
within one machine, so Arthor still serves collections a hundred times larger
than anything verified here. Building a single shard is partly parallel — the
two screening passes go 6.5x on ten cores, the whole build 2.69x — and building
shards at once goes past that: **8.17x on eight**, which put 50 M molecules at
26 minutes 44.

**Where it is ahead: the count is real.** An Arthor substructure search stops at
20 000 hits. This one either returns the true count or says, in the response,
that what it returned is a floor and why. Nothing is truncated and presented as
final. Hit sets themselves agree where they can be compared — benzene on the
reference database returns 1973, and 1453 with ring systems locked, which are
the same two numbers and the same delta of 520.

And it runs on your machine. Queries never leave the process, no collection is
uploaded anywhere, and there is no service to depend on.

### Others in the same area

Not measured here, and listed so the map is honest rather than flattering.
**chemfp** is faster than this at Tanimoto and does only similarity — no
substructure search. **SmallWorld** answers a different question, nearest
neighbours by graph edit distance, which this cannot do at all. **Open Babel**
and its `fastsearch` index cover smaller collections with a broader toolbox.

The niche this aims at is narrow: exhaustive substructure and SMARTS search over
collections in the hundreds of millions, self-hosted, as one file.

---

## Known limitations

- **`fmt=sdf` is refused with a `400`.** It needs 2D coordinate generation,
  which is not implemented. Use `tsv` and generate coordinates with RDKit.
- **The interface is one page and does four things.** Query, results, a local
  collection you can export, and a structure editor. There is no dashboard, no
  saved searches and no user accounts, and the API is the supported way to do
  anything the page does not.
- **The Arthor web UI has not been tested against this.** `/config` serves the
  blob it reads on startup, but nobody has pointed it here.
- **No canonicalisation.** The same compound in two catalogues is two unrelated
  rows; there is no duplicate detection and no identity that spans databases.
- **Sharding is within one machine, not across machines.** A database may be
  built as many shards and is searched across all of them, but every shard has
  to be on the box serving it. Several databases can still be searched in one
  request.
- **211 molecules in 2 897 819 read a different aromatic system from RDKit**,
  and the composition matters more than the number. **202 of them are cases
  where RDKit's own answer is not a function of the molecule**: on a furan or
  thiophene bridged so that its oxygen also lies on a ring of nine atoms or
  more, its answer depends on which ring its walk reaches first, and the same
  graph read 5 aromatic atoms on 128 of 300 random writings of itself and 0 on
  the other 172. Arthor is stable there and agrees with this build. The residue
  that is genuinely ours is **nine molecules** — two aromatic-boron rings, two
  fullerene adducts at one atom each, five one-offs.
- **Only three quarters of a *single shard's* build is parallel.** The screening
  passes use every core; reading the input, merging the posting runs and writing
  the fingerprints do not. Measured on ten cores at 124 M: the parallel passes
  go 6.5x, one whole build 2.69x. Building several shards at once is what gets
  past that — 8.17x on eight — and it is the reason `--shards` exists. ⚠ The two
  multiply for memory: eight concurrent builds measured **3.94 GB** peak, not
  the 1.7 GB a per-build figure predicts, because each shard build runs its own
  screening threads.
- **No canonical identity across a rebuild of different data.** Row ids are
  stable for an index generation; a rebuild that reuses them for different
  molecules must bump the generation, and nothing checks that for you.

---

## Security

- Query structures are treated as confidential intellectual property: never
  logged in full, never sent anywhere, never written outside the process.
- The software performs no outbound network communication, sends no telemetry
  and reports nothing to anyone.
- A malformed SMILES or SMARTS is a `400`, never a crash.
- Every search is bounded by wall clock, 30 seconds by default, and a search
  that hits the bound reports a partial count as partial. Searches also queue
  behind a bounded pool, so a caller issuing them faster than they complete gets
  a `503` rather than turning one deadline into a queue of deadlines.
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

---

## Licence

Proprietary — see `LICENSE`. Source code is not distributed.

Third-party open source components and their licences are listed in
`THIRD-PARTY-LICENSES`.
