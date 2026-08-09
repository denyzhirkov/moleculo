# moleculo

Chemical structure search over large molecule collections — substructure,
SMARTS, similarity and exact formula — served over an HTTP API that is
wire-compatible with Arthor.

A single static binary. No runtime, no dependencies, no installer, no database
server. **Bring your own molecules**: nothing is bundled, nothing is uploaded,
and query structures never leave the process.

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
fixed corpus of 2 897 819 ChEMBL molecules: **99.984% agree on every field**,
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
| `--fp-codec`, `--screen-codec` | the same, for the fingerprint column and the screening index. |
| `--replace` | swap out a shard that is already there. Requires a higher `--generation`. |
| `--generation N` | index generation. Row identifiers are stable within one generation and only within one. |
| `--notify URL` | call a running server's reload endpoint once the shard is published. |

**Plan for the time.** Building is CPU-bound and single-threaded, and it is
hours rather than minutes at scale. Two measured points on one core of a laptop:

| molecules | wall clock | index size | bytes/molecule |
|---:|---:|---:|---:|
| 10 M PubChem | 50 minutes | 0.92 GB | 91.6 |
| 124.4 M PubChem | 10.7 hours | 10.7 GB | 86.4 |

Note which way the density moved. Bigger collections cost *less* per molecule,
because the screening index compresses better when each feature has more rows in
it, so sizing from a small trial run overestimates rather than under. On ChEMBL,
whose chemistry is heavier, the same build is 129.5 bytes per molecule — the
range across the corpora tried here is 86 to 130.

**Plan for the memory, and read the number carefully.** The build is out-of-core
and its own allocation stays small — the 124 M build peaked at 186 MB. What
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
| `--no-search-limit` | removes both. For offline work, not for a server anyone else can reach. |

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

---

## Known limitations

- **`fmt=sdf` is refused with a `400`.** It needs 2D coordinate generation,
  which is not implemented. Use `tsv` and generate coordinates with RDKit.
- **No graphical interface ships with this.** The API is wire-compatible with
  Arthor, including the `/config` blob its web UI reads on startup, but that
  combination has not been tested here. Today the client is a script or a
  spreadsheet.
- **No canonicalisation.** The same compound in two catalogues is two unrelated
  rows; there is no duplicate detection and no identity that spans databases.
- **No sharding.** One database is one index directory on one machine; several
  databases can be searched in one request, but a single database cannot be
  split across machines.
- **399 molecules in 2 897 819 read a different aromatic system from RDKit**,
  80 of them a single homologous series. They are catalogued rather than
  unknown, and the direction matters more than the count: a stricter reading
  loses a hit silently, a looser one returns an extra.
- **Building is single-threaded.** 124 M molecules is a working day on one core.
  The work parallelises and is not memory-bound; it simply has not been done.

---

## Security

- Query structures are treated as confidential intellectual property: never
  logged in full, never sent anywhere, never written outside the process.
- The software performs no outbound network communication, sends no telemetry
  and reports nothing to anyone.
- A malformed SMILES or SMARTS is a `400`, never a crash.
- Every search is bounded by wall clock, 30 seconds by default, and a search
  that hits the bound reports a partial count as partial. **There is still no
  admission control and no rate limiting**: a caller can keep every core busy by
  issuing searches faster than they complete. It listens on loopback by default;
  keep it there, or put it behind something that authenticates and rate limits.

---

## Licence

Proprietary — see `LICENSE`. Source code is not distributed.

Third-party open source components and their licences are listed in
`THIRD-PARTY-LICENSES`.
