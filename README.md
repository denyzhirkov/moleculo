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
- [Operating it](#operating-it) — adding data, several sources, reloading, a missing shard
- [The search page](#the-search-page) — draw a query, read the results, keep what matters
- [What this is measured against](#what-this-is-measured-against) — RDKit, Arthor, where this loses and where the answer is better
- [Which of these you actually want](#which-of-these-you-actually-want)
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

**How correct is the reading?** Every molecule is diffed against RDKit, field by
field, over two corpora from different producers:

| corpus | molecules | agree on every field | refused where RDKit accepts |
|---|---:|---:|---:|
| ChEMBL | 2 897 799 | **99.991%** | 5 |
| ZINC20 | 5 119 973 | **99.9999%** | 14 |

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

The ZINC sample turned up fourteen refusals RDKit does *not* share, and they are
worth knowing about because the disagreement runs the other way. Each carries a
**triple bond between two aromatic atoms inside the ring** — `c1sc#cc1I`. A
triple bond is linear, so no five-membered ring can hold one, and a ring that
holds one cannot be planar or aromatic. RDKit reports it as five aromatic atoms
with a triple bond among them; this build says it cannot kekulize the ring,
which is precisely what is wrong with it.

ChEMBL's five are small enough to name, and four of them are one thing: an
**aromatic ring containing boron** — `Cc1b[n+](…)cc(C)c1` and three others.
Boron in an aromatic ring is a documented divergence rather than an oversight;
the fifth is a `[c-][n+]` ylide. ⚠ Two acridinium-type `[nH+]` rings and a zinc
complex used to be in this list and are not any more — the acridiniums because
0.6.0 refuses them *and so does RDKit*, which moves them out of the disagreement
rather than fixing them.

---

## Building an index

An index is a derived artifact: reproducible from the input and the builder
version, safe to delete and rebuild, and never edited in place.

⚠ **Indexes built before this release must be rebuilt.** The packed molecule
changed shape — an atom with no unusual property now costs one byte instead of
two, and a bond that continues the chain costs its flags byte and nothing else —
so the index format moved from version 2 to 3. A shard of the older version is
**refused by name**, loudly, rather than read with the wrong rules: this is
exactly what the version field is for, and it is why the older shards are not
silently misread into molecules that are garbage wearing a valid shape.

What it buys, measured on 400 000 PubChem molecules: `mol.pack` **−8.1%** and a
scan **5.2% faster** (30.4 against 28.9 million atoms a second). Across the whole
index the size figure is about **−2.6%**, because the packed molecule is a third
of it.

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
| `--bits N` | similarity fingerprint width: 256, 512 (default), 1024 or 2048. Narrower costs recall, wider costs disk. Fixed at build time — it is a property of the index, not of a query. |
| `--codec none\|zstd` | compress the molecule and record stores. Default `zstd`. |
| `--fp-codec`, `--screen-codec` | the same, for the fingerprint column and the screening index. ⚠ `--fp-codec none` is **6.6x the similarity throughput for about 26% more index** — measured, see below. |
| `--replace` | swap out a shard that is already there. Requires a higher `--generation`. |
| `--resume` | continue an interrupted sharded build: a shard already sealed is skipped rather than refused. ⚠ Only correct if the input file has not changed — a shard records how many molecules it holds, not which bytes it came from |
| `--generation N` | index generation. Row identifiers are stable within one generation and only within one. |
| `--notify URL` | call a running server's reload endpoint once the shard is published. |
| `--threads N` | workers for the two screening passes, which are three quarters of a large build. Default: the machine's parallelism. Does not change the shard; does raise memory. |
| `--shards N` | split the input into N shards. **Size the shard, not the machine** — see below. |
| `--build-workers N` | how many shard builds run concurrently. Default: the machine's parallelism. ⚠ **Lower this on slow or networked storage**; it is a bound on concurrent I/O as much as on memory. |

**Plan for the time.** Building is CPU-bound and it is hours rather than minutes
at scale. Measured on a ten-core laptop:

| molecules | one core | ten cores | index size | bytes/molecule |
|---:|---:|---:|---:|---:|
| 10 M PubChem | 50 minutes | — | 0.89 GB | **89.25** |
| 124.4 M PubChem, one shard | 10 h 43 | **3 h 58** | 10.7 GB | 86.4 ⚠ |
| 124.4 M PubChem, 16 shards | — | ~18 h at two threads ⚠ | 10.49 GB | **84.34** |

Note which way the density moved. Bigger collections cost *less* per molecule,
because the screening index compresses better when each feature has more rows in
it, so sizing from a small trial run overestimates rather than under. On ChEMBL,
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

### Choosing the shard count, and why eight is not always the answer

**Size the shard, not the machine.** A shard's postings are spilled to sorted
runs and then merged, and the merge reads every run at once — so the number of
open streams during a build is *runs per shard × concurrent builds*. Runs scale
with shard size, at roughly one 64 MB run per 570 000 molecules:

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
| similarity, ten threads | 88 M molecules/s | **580 M/s** |
| similarity, one thread | 10 M/s ⚠ | 53 M/s ⚠ |
| substructure scan | unchanged | unchanged |
| fingerprint column, 2.9 M molecules | 95 MB | 188 MB |
| whole index, 2.9 M molecules | 0.39 GB, 133 B/mol | 0.49 GB, 168 B/mol |

⚠ The two single-thread figures are marked because they were measured on an
earlier build and have not been retaken; every other number in the table is from
this one. The ten-thread row is the operative one either way.

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
molecules      10000000
sampled        49994 built, 6 refused by the parser
index size     933.7 MB
peak disk      1.5 GB  (the sort spills beside the index)
build time     7 min  ⚠ reads low, see below
```

**It measures your data rather than quoting ours.** Fifty thousand of your
molecules are built into a real index and scaled up, so the numbers come from
your chemistry on your machine. ⚠ A density constant would have been a guess
wearing a number — the collections measured here differ by half, from 83 bytes a
molecule to 129.5, and that spread is exactly what decides whether a disk is big
enough.

⚠ **Two numbers, two directions, both checked against a real 10 M build:**

- **Size reads high** — 933.7 MB estimated against 882.0 actual, **+6%**. A
  feature's postings compress better the more rows hold it, so a full collection
  costs less per molecule than a sample of it. Provision for this and you will
  have room.
- **Time reads low** — 7 minutes estimated against 9 min 20 s actual, **−25%**.
  A sample small enough to estimate quickly never spills its sort to disk, so
  the merge a real build pays for is missing from the figure. Allow margin.

Neither is corrected by a fudge factor. One ratio from one comparison is a
guess, and a guess dressed as a correction is worse than a number with its
direction attached.

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
**four** passes over every molecule and they run at different rates — ingest,
then the two screening passes near 67 000 molecules a second on a ten-core
laptop, then the fingerprint pass near 126 000 — so a build-wide estimate would
be a number this cannot honestly produce.

⚠ **Ingest reports in bytes, and it is the only pass that can.** The other three
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

**Size the disk for roughly twice the finished index** while a build runs. The
sort spills into the shard's staging directory before the merge reads it back,
and at 124 M molecules those runs were 6.3 GB beside a 10.7 GB index.

### In a container

The image carries **the binary from the release**, unpacked — it does not
compile its own. What runs in your cluster is byte-identical to what you would
download, including the leak checking that keeps a builder's home directory out
of it.

```sh
docker build --build-arg VERSION=0.7.0 \
             --build-arg TARGET=x86_64-unknown-linux-musl -t moleculo:0.7.0 .

docker run --rm -p 8080:8080 \
  -v /srv/indexes:/databases:ro \
  moleculo:0.7.0
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

`complete: false` means the wall clock ran out and `matches` is a **floor**.
`key` is hex because a 128-bit key does not survive JSON's number type; `null`
means your query has no key, which is not a key of its own.

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

⚠ **Without `G`, a query that draws a stereocentre matches both enantiomers**,
which is what both Arthor and `RDKit` do by default. Asked on a 3 004-molecule
collection, Arthor answers 212 for a chiral query and 212 for its mirror;
`RDKit`'s default answers 207 and 207. `G` is how you say you meant the wedge.

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

⚠ `"decisive": false` means the database was built before 0.6.0. Older shards
open and search exactly as before, but they did not record which fragments are
merely *common*, so "absent from the index" and "in almost every molecule" look
alike to them. **`absentFeatures` is then empty and the summary says why** — a
field named for absence does not get to carry a guess. Rebuilding that database
is what makes the answer decisive; nothing else about it is affected, and the
locks are reported either way because they are a fact about the query rather
than about the collection.

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
from where the atoms sit. So the coordinates are checked against `RDKit` over a
corpus rather than assumed: **78 935** molecules carrying stereochemistry, of
which **one** comes back as a different isomer. Its cause is named under
[Known limitations](#known-limitations).

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
one; nothing logs a request, and the databases involved are named where the load
matters. That is checked rather than intended — a run driving searches against a
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
**99.991% of 2 897 799 ChEMBL molecules agree on every field, and 99.9999% of
5 119 973 from ZINC20** — two producers, different chemistry. The rest are
catalogued, with a reason each, rather than left as a percentage. On the ZINC
sample the catalogue is empty of anything this build owns: every aromatic
disagreement there is one where RDKit's answer depends on how the molecule was
written.

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

| | Arthor demo (4.2.4) | this build |
|---|---|---|
| similarity, resident database | 3 300 M molecules/s at 1.647 B, 256-bit | 88 M/s default, **580 M/s** with `--fp-codec none` — ten threads, 512-bit, 2.9 M |
| largest database served | 15.18 B molecules | 124.4 M verified end to end |
| substructure hit count | capped at 20 000 | exhaustive, or an honest floor |
| index size | not published | **84.3 bytes per molecule** — 124 427 347 molecules in 16 shards, 10.49 GB |

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
| a molecule is too branched to decide | counted as a non-match | counted apart, in `undecided` |
| you negate a search whose count is a floor | answers | **refused** — the complement of a floor contains matching rows permanently |
| a similarity search is cut short | answers | **refused with `503`** rather than returning a histogram biased by what it missed |
| nothing matched | empty | `explain` says what was absent, and how many tautomeric forms your query has |
| you ask "do we already hold this?" | ⚠ **no such endpoint** | `/dt/{db}/identity`, saying how each row merged and what it set aside |
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
cores and memory bandwidth. Sharding is
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

- **One molecule in 78 935 leaves `fmt=sdf` as a different isomer.** Measured
  against `RDKit` over world-drugs, real-space, mcule and two disjoint
  100 000-molecule ChEMBL samples: **78 770 agree, 164 state *less* than they
  could, and one states something false.** That one is a 29-membered macrolactam
  whose ring shares 27 atoms with its neighbour — the layout refuses a fused
  macrocycle by construction, and no amount of redrawing gets round it.
  ⚠ **The 164 are losses, not lies**: a reader sees no configuration where it
  expected one, which is visible, rather than the wrong one, which is not.
- **A hydrogen on a charged sulfur is dropped from an SD file.** A sulfoximine
  written `N[SH+]([O-])c1ccccc1` comes back as `N[S+]([O-])c1ccccc1` — one
  hydrogen short, which is a different compound. It is the last of its kind:
  eight records in 100 000, all one motif, and it is a limit of what this writer
  puts on the atom line rather than of the model. It affects the SD file only;
  search sees the hydrogen.

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

  ⚠ **And that last sentence was itself too strong until this release, so it is
  corrected rather than quietly repaired.** The *model* always kept the
  deuterium — nothing was lost on the way in, which is what the paragraph above
  says and it is true. But **searching** for one did not work: inside a SMARTS
  bracket, `[2H]` compiled to *"isotope 2, carrying one hydrogen"* with the
  element dropped entirely, so a deuterium — which carries no hydrogens — failed
  a query drawn from itself, while the same pattern matched unrelated atoms like
  `[NH+]`. Seventeen two-letter element symbols went the same way: `[Rh]` was
  read as *"in a ring, with one implicit hydrogen"* and `[Rb]` as a
  contradiction that matched nothing. Fixed in this release, measured at
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
  substances.

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
- **The identity key is not yet invariant to how a molecule was written, on two
  molecules in forty thousand.** Hand `/dt/{db}/identity` the same compound
  spelled two different ways and **2 in 40 000 give two different keys**, which
  means the endpoint can answer *"we do not hold this"* about a compound the
  collection holds. Measured over 40 000 PubChem molecules in 958 143 spellings,
  24 apiece.

  ⚠ **This entry said 17 in 100 000 until 0.13.0, and both halves of that were
  wrong.** The number came from a check that drew **six** spellings per molecule
  and reported a *sampled lower bound* — a figure that cannot be compared
  between releases, because which molecules surface depends on the draw. Five of
  the causes it pointed at were one defect and are fixed; what is left is two
  fullerene cages.

  ⚠ **Those two are not a bug so much as a price.** On a fused cage the ring
  count is stable but the *choice* of rings is not — SSSR is famously non-unique
  there — and this build deliberately takes the cyclomatic number rather than
  RDKit's symmetrised SSSR. RDKit symmetrises precisely to make the choice
  canonical. Changing that would change what a ring means everywhere, including
  under the aromatic divergences listed below, so it is a decision rather than a
  fix.

  ⚠ **A further 40 in 100 000 are refused in one spelling and accepted in
  another** — the same molecule, a `400` or a `200` depending on where the author
  started the string. That one is untouched. Neither affects substructure,
  SMARTS, similarity or formula search, whose hit sets do not depend on the key.
- **Some rows cannot be decided, and now they say so instead of counting as
  misses.** Subgraph isomorphism is NP-complete, so the matcher abandons a
  molecule after a fixed amount of work rather than hanging your build on one
  row — a symmetric giant with a dozen interchangeable arms can outrun any
  budget. Those rows arrive in the response as **`undecided`**, counted apart
  from the misses, and the warning says the count is a floor. ⚠ **Earlier
  releases reported them silently as "your query does not occur here."** For
  calibration: matching every molecule against **itself** — the easiest query
  there is — 455 of 497 693 PubChem molecules exceed the budget. Raising it is
  not offered as a fix, because it trades a silent wrong answer for a slow one.
- **Some structures we hand back as SMILES are read by other tools as a
  different molecule.** The `SMILES` column every result carries — in `json`,
  `tsv` and `csv` alike — is consumed by whatever you pipe it into, and over
  497 877 PubChem molecules **10 come back
  through RDKit as a different graph and 12 it will not read at all** — 0.004%.
  ⚠ **This said 22 until 0.13.0**: twelve of them were a ring-fusion bond marked
  aromatic where both RDKit and Arthor mark it single, and that is fixed. The ten
  left are porphyrin-type macrocycles, a different mechanism. ⚠ **The molecule stored and
  searched here is correct**; it is the written form that is not portable. If
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
- **Sharding is within one machine, not across machines.** A database may be
  built as many shards and is searched across all of them, but every shard has
  to be on the box serving it. Several databases can still be searched in one
  request.
- **211 molecules in 2 897 799 of ChEMBL read a different aromatic system from
  RDKit** — three in 5 119 973 of ZINC20, where none of them is ours —
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
