# Changelog

## 0.7.0 — 2026-08-17

A release about running this somewhere other than the machine that built it: a
container image, configuration that refuses rather than guesses, and a build
that gives the disk back when it fails.

⚠ **Nothing about search or chemistry changed.** Answers, indexes and hit sets
are identical to 0.6.0, and no index needs rebuilding.

**Configuration is one validated thing, and it refuses at startup**

Every setting is now reachable as a flag *or* an environment variable — a
container is configured by its environment, not by a command line — with the
flag winning, so an operator overriding a running system beats the image rather
than the reverse. Every flag that worked before still works.

It stops guessing. An unknown flag is an error rather than something ignored,
because a typo that is ignored is a setting you believe you applied. A value
that does not parse names the field and quotes back what it got. And ⚠ **a
contradiction is refused**: `--no-search-limit` removes every bound, so giving
it alongside `--max-seconds` means you believe something untrue about your
server, and picking a winner would leave you believing it.

**`--bind`, and why the default did not change**

The server listened on `127.0.0.1`, hard-coded. That default is the security
posture rather than an accident — the port is the only boundary this product
has — so it stays, and `--bind` / `MOLECULO_BIND` is how anything opts out.

⚠ **The container image sets `0.0.0.0` and has to**: inside a container loopback
reaches nobody, and an image that kept the default would publish a port that
answers nothing. That is exactly how this was found. What it means for you is in
the README's security section: **the container's boundary is the port you map**,
so `-p 127.0.0.1:8080:8080` and `-p 8080:8080` are not the same decision.

**A container image carrying the binary from the release**

Unpacked, not compiled. An image that builds from source is a *different
artifact* from the one the release ritual verified, without the path-leak
checking that keeps a builder's home directory out of a public binary. 15 MB,
unprivileged as uid 10001, databases mounted read-only and never baked in —
a molecule collection is your property and has no business inside an image that
gets copied between registries.

```sh
docker build --build-arg VERSION=0.7.0 \
             --build-arg TARGET=x86_64-unknown-linux-musl -t moleculo:0.7.0 .
docker run --rm -p 127.0.0.1:8080:8080 -v /srv/indexes:/databases:ro moleculo:0.7.0
```

**A failed build gives the disk back**

⚠ Measured on a volume deliberately 72 KB too small, and both halves were
defects. The staging directory used to survive the failure holding everything it
had written — on a 1 020 KB volume that left **zero free**, so the retry failed
for a different reason than the first attempt. And the one useful line,
`No space left on device`, was followed by fifty lines of usage text that pushed
it off the screen. Now: one line, exit 1, nothing left behind, space returned.

Provisioning number that follows from it: **size the disk for roughly twice the
finished index** while a build runs. The sort spills beside it — 6.3 GB of runs
next to a 10.7 GB index at 124 M molecules.

**Still not here**

⚠ `fmt=sdf` is still refused with a `400` naming what is missing. The generator
exists and is close — the format is valid, RDKit reads 3 004 records with zero
warnings, and tetrahedral stereochemistry survives — but **53 molecules of
2 027 still come back with the wrong double-bond geometry**, and shipping an
export that hands a chemist a different isomer is worse than refusing. Use `tsv`
and generate coordinates with RDKit until it is right.

## 0.6.0 — 2026-08-15

An empty answer now tells you why it is empty, the drawing of a hit is no longer
the wrong enantiomer, and the server says something to its operator.

⚠ **One reading change, and it is narrow.** An aromatic ring nitrogen carrying a
hydrogen and a positive charge — `[nH+]` — was read wrongly in *both*
directions. Pyridinium `c1cc[nH+]cc1` was refused where RDKit accepts it, and
the five-membered `c1cc[nH+]c1` and acridinium were accepted where RDKit refuses
them. A cation has no lone pair left to donate, and the rule demanded no explicit
hydrogen rather than counting it. **If your collection contains aromatic
`[nH+]`, rebuild for identical answers** — ChEMBL holds none in the rows this
build accepts, so its agreement figure is unchanged at 99.991%, which also means
ChEMBL cannot tell you whether your collection does.

What did move is the refusal count: re-measured against a fresh ChEMBL download
rather than carried over, **rows this build refuses and RDKit accepts went from
7 to 5**. Four are an aromatic ring containing boron and one is a `[c-][n+]`
ylide. The two acridinium-type rings left that list because 0.6.0 refuses them
*and so does RDKit* — moved out of disagreement rather than fixed.

**An empty structural search explains itself**

A `Substructure` or `SMARTS` search that finds nothing comes back with an
`explain` object instead of a bare zero: which fragment of the query the
collection does not hold, and which `qopts` locks were in force. The second is
the commonest invisible cause of a surprising empty result — `R` stops benzene
matching naphthalene, and nothing in a zero says so.

It costs an index lookup rather than a second search, and a search that *found*
something never computes it. The field is absent from every other response, so a
client written against Arthor's shape never meets it.

⚠ **On a database built before 0.6.0 it answers "I cannot tell", and that is the
honest answer.** The screening index drops a fragment both when nothing carries
it and when more than one row in sixteen does — opposite facts with the same
symptom. 0.6.0 records the second case as well, so absence becomes decisive;
older shards did not, so they report `"decisive": false`, name no fragment, and
say rebuilding is what settles it. Everything else about them works unchanged.

**The mark on a drawn hit was the wrong enantiomer**

The highlight travels inside a notation this build writes, and the writer
reordered an atom's neighbours without flipping the descriptor that is a parity
over that order. Same atoms, same bonds, same formula, same hit set — the other
enantiomer. Every test about constitution passed on both sides of it for the
life of the code. Round-trip fidelity on ChEMBL: **73.82% to 99.990%**.

Hit sets never depended on chirality, so **no index needs rebuilding for this**.
Only what you were shown was wrong.

⚠ Residue, named rather than hidden: **271 molecules of 2.9 M** carry a
three-coordinate chiral nitrogen, sulfur or phosphorus that still comes out
mirrored. Two candidate rules were implemented and measured against RDKit during
this cycle and both were refused — one broke 3 996 molecules of a 4 000 sample,
and the one that fixed 119 while breaking 8 was rejected for scoring by accident
rather than by mechanism.

**The server talks to its operator**

Structured events on stderr, `RUST_LOG` for filtering, `info` by default. A
refused admission is reported — a stream of `503`s could not previously be told
from a broken server. A degraded database warns when it mounts.

⚠ **Query structures are never logged, at any level.** Checked rather than
intended: a run driving searches against a full pool was grepped for the query
that produced them, and it appears zero times.

**Smaller things**

- `index build` tells three different truths where it used to tell one: a
  destination holding a shard, a destination holding something else (where
  `--replace` is *not* the answer and used to be suggested), and an empty
  directory — which now simply builds instead of being refused for holding a
  shard it does not hold.
- The standardiser's choice of which negative keeps its charge no longer depends
  on how the record was spelled. 35 ChEMBL molecules change; 31 move toward
  RDKit's answer and 4 away, because a canonical ranking is not *their* canonical
  ranking.

**Under the hood, and deliberately not wired up**

There is now a canonical form, a 128-bit key over it, and RDKit's standardisation
chain — normalisation catalogue, fragment parent, uncharger, reionizer — ported
and diffed against the original. None of it is reachable from the binary.

⚠ That is a decision, not an omission. Measuring the key over ChEMBL showed 4.12%
of records merging, **99.88% of it through the "largest fragment is the compound"
rule** — which cannot tell a counter-ion from a second active ingredient. Twenty
four ionic liquids file under their shared anion; twenty five combination
antibiotics file under the bigger drug, discarding tazobactam and sulbactam,
which are drugs. Duplicate detection needs a second signal, and shipping it
without one would quietly delete distinct substances.

**Index format**

Additive. A 0.6.0 build records which fragments are too common to index; older
shards lack that and open exactly as before. **No rebuild is required by this
release** — only the `[nH+]` reading above can change answers, and only on
collections that contain it.

## 0.5.0 — 2026-08-11

A database can be more than one shard, the aromatic reading got substantially
closer to the reference, and searches queue instead of trampling each other.

⚠ **Aromaticity changed, so hit sets changed.** 188 molecules of the ChEMBL
corpus read differently than they did in 0.4.0, and the half that matters most
is where this build used to see *less* aromaticity than RDKit — it lost hits a
chemist was looking for. Indexes built by earlier versions still open and still
work; but for identical answers, rebuild.

**A database can be split into shards, and built eight at a time**

`--shards N` cuts an input into N shards and builds them concurrently. Shards
are independent, so this has none of the sequential tail one build has: 50 M
PubChem as eight shards took **26 minutes 44 against 3 h 38 of processor time —
8.17x**. Searches fan out across shards and merge; hit sets are identical to
searching the same molecules as one shard, which is asserted over the query
corpus rather than assumed.

⚠ Budget **3.94 GB** of memory for eight concurrent builds, not the 1.7 GB a
single build's 211 MB would suggest. Each shard build runs its own screening
threads, so the two kinds of parallelism multiply. Turn `--threads` down as well
as `--build-workers` on a smaller machine.

A directory holding an index the old way is still a database and still opens.

**A missing shard makes a database incomplete, not unavailable**

If a shard's directory is gone, the rest still serve. `/dt/data` drops the
missing part from `partCnt`, and **every search over that database carries a
warning naming the shard**. Read it: `recordsTotal` falls *with* the missing
shard, so a degraded answer is internally consistent and looks complete. The
warning is the only thing distinguishing it. A database whose every shard is
gone is refused rather than served as an empty one.

**You can see how much of a database is in memory**

`GET /dt/{db}/data` now reports real page residency per shard and per index —
`incore`, `total`, and a coarse chart — which is what decides whether a query
takes seconds or minutes. Per shard rather than averaged, because a database is
warm or cold in parts and one figure hides the shard that is paging.

**Searches queue rather than fighting**

A wall-clock bound limits one search; it never limited a caller issuing them
faster than they finish, which turned one deadline into a queue of deadlines
where everything times out. Searches now wait behind a bounded pool — one per
core by default — and the wait is charged against the search's own deadline, so
one that queued away its thirty seconds is refused rather than admitted to do
token work. Past the queue depth, a search is refused on arrival.

`time` in a response is now the work and `requestTime` the work plus the wait;
they used to be the same number, reporting a queue wait of zero forever. This is
the split Arthor's own contract defines, and measuring their instance showed
10.7 seconds of queueing against 20 milliseconds of work.

New: `--max-concurrent-searches`, `--max-queued-searches`.

**An exact-formula count converges, and 0.4.0's warning about it is withdrawn**

0.4.0 said a formula count was still a floor because a formula search answers
once. It no longer answers once — it is a job like substructure, publishes as it
goes, and converges across polls. On 124 M it is exhaustive in 16 seconds where
0.4.0 answered `C9H8O4` with 463 and called it final. It is 1246.

**The aromatic reading, and an admission about the reference**

Divergence from RDKit on the ChEMBL corpus went from **399 molecules to 211**,
and full-field agreement from 99.984% to **99.991%**. Four changes did it: the
reference's own fused-ring algorithm replaced four home-grown rules; benzenoid
polycyclics may enclose their own interior carbons; an atom now lends the ring
only what its bonds have left it; and a guard requiring a π bond somewhere was
retired after the reason it existed had been covered elsewhere for two releases.

⚠ **202 of the 211 that remain are cases where RDKit disagrees with itself.** On
a furan or thiophene bridged so its oxygen also sits on a ring of nine atoms or
more, its answer depends on which ring its walk reaches first: the same molecule
read 5 aromatic atoms on 128 of 300 random writings of itself, and 0 on the
other 172. Arthor is stable there and agrees with this build. **The residue that
is genuinely ours is nine molecules.**

**What has not changed**

No authentication, no per-client rate limiting, no notion of a user. Anyone who
can reach the port can search — and a SMARTS of `[*]` with `fmt=tsv` exports the
whole collection. The port is the security boundary and there is nothing behind
it. It listens on loopback by default; keep it there or put it behind something
that authenticates.

Still no canonicalisation, so the same compound from two catalogues is two
unrelated rows. Still no `fmt=sdf`. Sharding is within one machine.

## 0.4.0 — 2026-08-10

The page learned to show why a molecule matched, and two counts learned to stop
lying.

**You can see why a row is a hit**

A structure search now returns each hit with the matched atoms marked, and the
page draws them marked. No other client of this API can do that, because the
match is expressed in the notation itself rather than as a list of atom numbers
that a renderer would have to agree with.

**Click a structure to read it**

The table draws small because it has to fit hundreds of rows. Clicking a drawing
opens it at readable size, with its identifier, its notation, a copy button, and
arrows — or the left and right keys — to walk the hits without going back to the
table.

**An exact-formula count says when it is a floor**

A formula search applies the same wall-clock bound as every other search. When
that bound stopped it, the response used to report a partial count as final: the
same 124 M collection answered one query with 378 hits one minute and 377 the
next, depending on what else the machine was doing. The response now carries how
many molecules were actually examined, and says the count is a floor.

⚠ It is still a floor. Substructure and SMARTS run in the background and their
counts converge as you poll; a formula search answers once. On 124 million rows
it examines about a fifth of the collection in thirty seconds.

**The fingerprint codec is measured, and it is the similarity lever**

`--fp-codec none` is **4.6x the similarity throughput** of the default, for
about 46% more index — 84 M molecules/second against 385 on ten threads, with
the fingerprint column going from 2.98 GB to 7.96 at 124 M. The substructure
scan does not change, because it never reads that column. Compression stays the
default; the flag is there for a deployment where similarity is the whole job.

**Build numbers, measured rather than projected**

0.3.0 said a parallel 124 M build should take about 3.5 hours. Run: **3 h 58**,
a 2.69x gain on 10 h 43. The two screening passes go 6.5x; everything else is
sequential. Ten threads also spend 14% more processor time in total than one and
peak at 849 MB rather than 186 — the wall clock is what improves.

**Also**

- A mark: the page and its browser tab now have one.
- A refused query already named the atom in 0.3.0; the page now shows that
  reason rather than only the reference's wording.

**Known limitations**

- `fmt=sdf` is refused with a `400`; it needs 2D coordinate generation.
- A formula count over a large collection is a floor, as above.
- No canonicalisation, so no duplicate detection and no identity across
  collections.
- No sharding: one collection is one index directory on one machine.
- Only three quarters of a build is parallel, and after that the posting merge
  is the largest single phase; more threads will not touch it.
- 399 molecules in 2 897 819 read a different aromatic system from RDKit, 80 of
  them one homologous series.
- The structure editor draws its own interface and cannot be themed with the
  page.


## 0.3.0 — 2026-08-10

The engine grew an interface, and the build stopped taking a working day.

**A search page, in the binary**

`moleculo serve` now puts a page on the port it listens to. Query field, the
four search types, collections, `qopts` locks, results, export. **Every byte of
it comes from this process** — no CDN, no font service, no depiction service, no
structure resolver — because the deployment this is built for is a corporate
network with no route to the internet. A page that fetches a font from a third
party fails exactly there, and it fails silently: the page renders and the font
never arrives. There is a test that scans the embedded page for external URLs.

- **Draw a query.** A structure editor opens on demand. Drawing writes into the
  query field, the field reads back into the editor, and query features drawn in
  it come out as SMARTS. The field is the source of truth, so typing, pasting
  and drawing all end at the same string.
- **Hits are drawn, not just spelled.** A query is usually written aromatic and
  a catalogue often stores Kekulé, so a correct hit can look like nothing you
  recognise. Structures are painted as rows come into view.
- **A collection.** Star a hit and it is kept in your browser — structure and
  identifier both, so it survives a rebuild that renumbers rows — and exports as
  TSV. It never reaches the server.
- **The count says what it is**: still counting, exact, or a floor with the
  reason and how much of the collection was examined. A number without that
  state is wrong in one of three ways.
- **Sorting sorts the loaded rows** and says so. Sorting 250 of ten million is
  not sorting the result.
- **Settings**: draw structures or not, rows per request, appearance.

**The build is parallel**

The two screening passes are three quarters of a large build and now run on
every core. Measured on 1 M PubChem molecules: 193.6 s on one thread, 45.8 s on
eight, 44.8 s on ten. `--threads` sets it; the machine is asked by default.

**Two builds of the same input remain byte-identical whatever the thread count**,
which the test suite asserts rather than assumes. Memory scales with workers —
111 MB at one, 515 MB at eight — so a build-memory figure now names a thread
count.

**Correctness**

Full-field agreement with RDKit over 2 897 819 ChEMBL molecules: **99.984%**,
from 99.840% in 0.2.0. The aromatic model now applies Hückel's rule only to atom
sets that trace a ring — a fused ring system's union frequently is not one — and
aromatic divergences fell from 4 526 to 399.

Refusals are measured over the whole of PubChem rather than a sample: 42 142 of
124 469 489 rows (0.034%), every one for a valence no element allows, and RDKit
refuses all of them too.

**Fixed**

- A refused query now says which atom was wrong, in a `detail` field beside the
  reference's own wording, instead of only "Invalid SMILES".
- A truncated search reports how many molecules it actually verified, in
  `verified`, instead of leaving a client to infer it from silence.

**Known limitations**

- `fmt=sdf` is refused with a `400`; it needs 2D coordinate generation.
- No canonicalisation, so no duplicate detection and no identity across
  collections.
- No sharding: one collection is one index directory on one machine.
- Only three quarters of a build is parallel; the rest is sequential, which puts
  the ceiling near 4x on a ten-core machine.
- 399 molecules in 2 897 819 still read a different aromatic system from RDKit,
  80 of them one homologous series. Catalogued, not unknown.
- The structure editor draws its own interface and cannot be themed with the
  page.


## 0.2.0 — 2026-08-10

The release 0.1.0 said was in progress. Every limitation it declared except two
is closed, and the index it could not build is built.

**The 12 M ceiling is gone**

0.1.0 shipped with a warning that indexes above roughly 12 M molecules did not
carry a complete similarity index. The cause was a decompression buffer sized by
guesswork — eight times the compressed block — which simply ran out and stopped
writing without an error. It now grows until the block fits, and a build that
cannot complete a pass says which pass and how far it got.

**124 427 347 PubChem molecules are now indexed and searched end to end**, every
slice verified complete. That build took 10.7 hours on one core and 10.7 GB of
disk — 86.4 bytes per molecule, *less* per molecule than the 91.6 a 10 M build
gives, because the screening index compresses better the more rows each feature
holds.

**Search**

- **Formula search works.** In 0.1.0 `type=Formula` was accepted and returned
  nothing, which is indistinguishable from "nothing matched". It is a scan, it is
  order-independent, and `C9H8O4` finds 378 molecules in a 124 M index.
- **Every search carries a wall-clock bound**, 30 seconds by default. 0.1.0
  documented one; no search actually had it. A search that reaches the bound
  returns what it found, marks `recordsFiltered` as a floor, and says so in a
  `warning`.
- **Similarity is refused rather than truncated.** A partial substructure count
  is honestly a floor; a partial similarity is a wrong answer, because
  `recordsFiltered` is the collection size by definition and the histogram would
  become a sample. Over the limit, it is a `503` that says so.
- **Several databases in one request, correctly.** 0.1.0 accepted the request
  and could return molecules that do not match the query: row lists were
  overwritten per database instead of accumulated, and then resolved against the
  wrong collection's records. A benzamide query could return sulfur hexafluoride.
  Fixed, with `partStats` per database, and the test asserts that every returned
  molecule actually matches.
- Similarity scanning is parallel across cores.

**Correctness**

Full-field agreement with RDKit over 2 897 819 ChEMBL molecules: **99.984%**, up
from 99.840%. 477 molecules disagree on any field, against 4 635 before.

The aromatic model now applies Hückel's rule only to atom sets that trace a
ring. A fused ring system's union frequently is not one — perimidine's three
rings enclose naphthalene's junction carbon, a porphyrin's five enclose four
nitrogens — and counting those as circuits was aromatising bridges that neither
RDKit nor the reference implementation aromatises. 4 526 divergences became 399.

Refusals are measured on the whole of PubChem rather than a sample: of
124 469 489 rows, 42 142 are refused (0.034%), all for a valence no element
allows, and RDKit refuses them too. Perchlorates written `[O-]Cl(=O)(=O)=O` and
hypervalent phosphorus written `PF6-` are now read rather than refused.

**Operating**

- **`serve` takes any number of source roots.** Two roots offering the same
  database name is an error, not a silent winner — the name is the collection's
  identity in the API.
- **A build is published by rename.** It writes into `<name>.building` and moves
  it into place when sealed, so a running server never maps a half-built shard.
  Replacing an existing shard requires a higher `--generation`, because row
  identifiers are stable only within one.
- **`POST /admin/reload`** rescans the roots and swaps the catalog while
  in-flight searches finish against their snapshot. Off unless `--enable-reload`.
- **`--build-dir`** makes a `chembl.smi` dropped in a root become the `chembl`
  database. Off unless asked for.
- **`--notify URL`** has a finished build tell a running server to reload.
- A build that dies no longer leaves its spill files behind.

**Fixed**

- `/config` returned an incomplete blob; it now returns every key the reference
  serves, including three that are deliberately empty because this build has no
  depiction service, no resolver and no default database.
- `memInfo` is present on the single-database endpoint and absent from the list,
  which is what the reference does. Asserting one fixture had made both wrong.

**Known limitations**

- `fmt=sdf` is refused with a `400`; it needs 2D coordinate generation.
- No canonicalisation, so no duplicate detection and no identity across
  databases.
- No sharding: one database is one index directory on one machine.
- 399 molecules in 2 897 819 still read a different aromatic system from RDKit,
  80 of them one homologous series. Catalogued, not unknown.


## 0.1.0 — 2026-08-09

First public build.

**Search**

- Substructure and SMARTS search, exhaustive rather than truncated: the reported
  count is the real one.
- `qopts` locks — `R` rings, `C` chains, `Q` charge, `I` isotope, in any
  combination.
- Similarity search with Tanimoto, Dice, Dissimilarity, Manhattan and
  parameterised Tversky, each with its own ranking direction, plus the 256-bucket
  score histogram.
- Fingerprint width is chosen when the index is built (`--bits`, default 512).

**Correctness**

Hit sets and molecule readings are diffed against RDKit over a fixed corpus of
2 897 819 ChEMBL molecules: **99.840% full agreement**, with the remaining
divergences catalogued rather than unknown. Of that corpus, 7 molecules are
refused where RDKit accepts; over 10 M PubChem molecules, 211 are refused and
RDKit refuses all 211 as well.

Benzene on a 3 004-molecule reference database returns 1973 hits, and 1453 with
ring systems locked — the same counts, and the same delta of 520, as the
reference implementation this API is compatible with.

**Index**

- Versioned on-disk format. A reader refuses an unknown version by name rather
  than guessing.
- Out-of-core build: memory stays bounded regardless of input size. 10 M
  molecules build at a peak of 375 MB resident.
- Every slice compressed; 91.6 bytes per molecule on a 10 M PubChem index.

**Serving**

- Arthor-compatible HTTP surface: `/config`, `/dt/data`, `/dt/{db}/data`,
  `GET` and `POST` search across multiple databases, `json`/`tsv`/`csv`.
- A missing, corrupt or out-of-date index takes only its own database out of
  service; the rest keep serving, and the reason is logged by name.

**Known limitations**

- `type=Formula` is accepted but returns no results — not implemented yet.
- `fmt=sdf` is refused with a `400`; it needs 2D coordinate generation.
- No canonicalisation, so no duplicate detection and no stable identity across
  databases.
- Indexes above roughly 12 M molecules do not yet carry a complete similarity
  index. Substructure and SMARTS are unaffected. Fix in progress.
