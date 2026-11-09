# Changelog

## 2.0.0-rc.1 — 2026-11-09

⚠ **2.0 does not serve a 1.x index (format 3) until it is converted with `moleculo index migrate`.**

**The release candidate for 2.0. Two measurements are owed before 2.0.0.**

- **The speed claim on the population its bar names.** Against 1.1.0's scan,
  on one thread, over the first 200 000 rows of PubChem, the 56 corpus queries
  ran **2.12× faster** summed, and each of the three ring queries `[R2]`,
  `[r5]` and `[r6]` at least **10.83×** faster. That was measured on
  2026-10-31, on the development tree of that day (`4db6cff` with the ring
  lookups merged), and this release has changed the scan path since: the
  search keeps its state on the heap past 256 placed atoms (below), and a
  store's blocks are held to stricter checks as they are read. The bar names the first 1 000 000 rows. That run is
  owed, and it will be taken on the release binary; until it is done, these
  figures are an indication.
- **The comparison with Arthor**, on time to the first page and on exhaustive
  throughput. Every figure README sets beside Arthor's was measured on 1.x and
  index format 3, and the public instance it was measured against did not
  answer when last tried, on 2026-11-09.

Everything 2.0 breaks against 1.x, and what 2.x promises, is in
[`COMPATIBILITY.md`](COMPATIBILITY.md).

### Upgrading an index

**`moleculo index migrate <database>` converts a 1.x index to 2.0's format in
place**, from the input lines the index already holds — no input file needed.
Row identifiers, records and deletions are kept; the result is what a fresh 2.0
build of the same lines writes, so it gives 2.0's answers. It is resumable and
swaps one shard at a time. See *Upgrading a 1.x index to 2.0* in the README.

**A 2.0 shard records which chemistry rules built it, and one built under
older rules is served as built and named.** Its index entries in `/dt/data`
carry an extra `perception` field holding the version it was built with, and
the server logs one warning per such shard when it opens the database. A
current shard carries no such field, so `/dt/data` is unchanged for it.
Searches over such a shard are complete under the rules it was built with and
carry no warning. Rebuild the shard from its input to take the new rules.

### Index format 4

**Every store carries a checksum, and it is checked as the store is read.**
`mol.pack` has a CRC-32C per block; `fp.col` its zstd frame checksums, or
`fp.sum` under `--fp-codec none`; `popcount.col` its `popcount.sum`; `smi.blk`
its zstd frame checksums, or `smi.sum` under `--codec none`; the offset tables
and `deleted.roar` a CRC-32C each, checked when the shard opens. 1.1.0 found
damage only where it broke a record's length, so a flipped bit inside a
fingerprint read as a different fingerprint and moved the scores. That is
found now, and the shard is served as missing, as for any other damage.
[`INDEX-FORMAT.md`](INDEX-FORMAT.md) lists every checksum.

**There is no screening index, so every substructure search scans.** A 1.x
shard carried an index of the path features its molecules hold, and since
0.16.0 a substructure search could use it to skip the rows without a rare
feature the query needed. On an
8 476 919-row PubChem shard it planned one of the 56 corpus queries, and it
cost 12–17 bytes a molecule on the three shards measured. `index build
--screen-codec` goes with it: passing it is now an error. An empty answer
still names the fragment the database lacks, from the feature counts each
shard keeps in its manifest.

**The molecule store is LZ4, in blocks of 256 molecules.** The first 1 000 000
rows of PubChem build to 84.9 bytes a molecule, of which the molecule store is
40.1, in ten files.

**`index build --codec` now chooses the record store's codec only.** In 1.x it
covered the molecule store too; in format 4 the molecule store is always LZ4
HC, so `--codec none` no longer gives an uncompressed one.

**Format 4 is frozen from this release on.** A shard 2.0.0-rc.1 builds is
expected to open in 2.0.0 unchanged. Should anything before 2.0.0 have to change
the format, it becomes version 5, announced in the first line of that release's
entry, and a shard built by this release then needs `index migrate` — the
no-migration promise of `COMPATIBILITY.md` starts at 2.0.0.

### Answers moved

**A similarity ranking's first 1 000 rows are scored exactly.** The search
still scores every molecule by its 512-bit folded fingerprint, which is what
makes the count and the histogram cover the whole collection, but folding
lets unrelated features share a bit, and for a small query that can decide
the ranking: `COC` over a 12 000-molecule sample ranked first a bipyridine
that shares nothing with it. Each database's first 1 000 rows are now scored
again from their unfolded features and re-ordered by that score, so a row
whose score was only collisions scores 0 and drops. Below row 1 000 the
order and the scores are still the folded ones, and a page that reaches
there says so in `warning`. Manhattan is unchanged. The re-scoring costs
about 8 ms per search, whatever the database size; on 500 000 molecules that
more than doubled it, 54 → 120 ms for 8 queries.

**`R<n>` and `r<n>` no longer depend on how a molecule was written.** They
used to count the rings of a minimal ring basis, and which of two equal rings
the basis held depended on the spelling: on 21 of 22 fullerene-type cages and
on about 1 in 100 ordinary PubChem molecules (bridged bicyclics, adamantanes,
quinuclidines). They now count every relevant cycle, meaning every ring that
is not a sum of smaller ones, which depends only on the structure. A ring
system with exponentially many equal rings, such as a cyclodextrin, counts
each such family once. Against RDKit, the share of molecules
where any atom's `R` or `r` differs fell from 255 to 5 in 22 423 across the
three test corpora. The five that remain are named in the documentation: in
each, RDKit keeps one of two equal rings. In the pinned query set, one answer
moved: `[R2]` over the REAL Space sample lost three cubanes, whose atoms all lie
on three faces. Over the first 200 000 rows of PubChem (199 977 molecules),
1.1.0 and this release's development tree of 2026-10-31 returned identical
hits on 55 of the 56 corpus queries, and `[R2]` went from 69 421 hits to
69 332. On prismanes and fully fused belts (cyclacenes, carbon
nanobelts) we now count the end rings or rims, and RDKit does not: it leaves
part of the ring structure uncounted. Aromaticity, `identity` and
the molfile's 2D layout still use the ring basis.

### Search time limits

- **Several databases named in one search now share one `--max-seconds`
  bound.** The limit used to apply to each database separately, so a search
  over three databases could run for three times the limit. The README's "per
  search" now describes what happens.
- **A search that meets a damaged shard and is rerun without it keeps its
  deadline.** Each rerun used to start a fresh `--max-seconds`, and up to eight
  reruns are allowed. A rerun with no time left answers as a search that ran
  out of time: a similarity search returns `503 Search Timeout`, and a
  substructure, SMARTS or formula search returns the count so far with
  `hasMore`. The background count keeps its own `--max-seconds`, so polling
  still reaches the full answer.
- **Similarity searches that arrive together are scored in one pass, and each
  still answers by its own `--max-seconds`.** The pass runs under the deadline
  of the search in it with the most time, and a search whose time runs out
  while it waits for a pass gets `503 Search Timeout` then, not when the pass
  ends. It used to stop at the deadline of the request that started it, which
  could time out a search that still had time left — and that request was held
  until no more searches were arriving, however long that took.

### Query bounds

- **A query past 4 096 atoms, or past 512 aromatic atoms counted either as
  written or after aromaticity is perceived, is a `400 Invalid Query`** with the
  limit named in `detail`. This applies to SMILES and SMARTS on every route that
  reads a query: substructure, SMARTS, similarity and `identity`. Such queries
  used to hold a core for seconds to minutes before any time limit applied (a
  4 002-atom aromatic ring took 57 s), and answered in the end. The largest real
  molecules sit far below the limits (ChEMBL: 855 atoms, 400 aromatic). A
  collection is still read without them. The aromatic limit is read on a
  SMILES query; a SMARTS query is held to the atom limit and to the three
  below.
- **A SMARTS expression nested more than 256 levels, more than two levels of
  `$()` (`$($(C))` is read, `$($($(C)))` is not), or more than 128 atoms inside
  all of a query's `$()` together, is a `400 Invalid Query`** naming the limit. Every walk over a
  query recursed once per level, so a few hundred kilobytes of `[*;*;*;…]`
  overflowed a thread's stack, and a stack overflow does not fail one request:
  it stops the whole server. Each `$()` level multiplies the work below it,
  which is why that limit is two.
- **Fixed: a query thousands of atoms long could stop the server.** Matching
  took one stack frame per query atom placed, so such a query against a
  molecule as long overflowed a worker's stack. Past 256 placed atoms the
  search now keeps its state on the heap. Answers, mappings and step counts
  are the same as before, compared on every corpus query over world-drugs.
- **`identity` says when it could not compare.** A query whose canonical search
  reaches its work budget answers `key: null`, `complete: false` and a
  `warning`, where it used to answer an empty `matches` beside
  `complete: true`. A candidate row that cannot be keyed makes the answer
  `complete: false`, with the count in `warning`. The budget is counted, not
  timed, so a `key: null` answer does not change when retried. No ChEMBL or
  PubChem-sample molecule lost its key. A 161-atom alkane dendrimer and a
  1 000-atom ring did.
- **The tautomer note on an empty search answer may be a floor.** It runs under
  a counted work budget, so it answers the same way on every machine. When the
  budget stops it, `tautomerFormsComplete` is `false` and the summary says "at
  least". On a 1-in-100 ChEMBL sample, 0.19% of reports that used to be
  complete are now floors.

### Also fixed

- **A hit whose record cannot be read retires its shard.** A result page
  rendered such a hit as a row with an empty `SMILES` column, with the count
  whole and no warning, and the shard kept serving. Now the shard is served as
  missing and the request is answered from the rest, with the warning naming
  it, exactly as for damage a scan meets.
- **A `database.json` that lists a shard twice is refused.** 1.x opened the
  shard twice and served its rows twice: a list of `[0, 1, 1]` served 438 rows
  of a 300-row collection. A list out of ascending order, or one naming a
  shard id of 2³¹ or more, is refused too, by the server and by `index
  append`, `migrate` and `inspect`. Only that database goes out of service.
  Every build, append and migration writes the list ascending, so only a file
  edited by hand is affected.
- **Two reloads at once no longer build the same source twice.** With
  `--build-dir`, two overlapping `POST /admin/reload` requests could both find
  the same new `.smi` and both start building it, and the second removed the
  staging directory the first was still writing. A reload that finds a build
  under way now waits for it, then answers as usual.
- **An `index append` killed in the middle of its swap is finished by the next
  one.** Appending to a single-shard database swaps the new layout in with two
  renames. Killed between them, the database was absent under its own name,
  and the next append stopped because it could not read it. The next append
  now finishes that state, after proving by file identity that the two
  directories are the halves of one append. Any other leftover state is
  refused, with what each directory holds named and nothing moved. README §
  *Adding to a database without rebuilding it* has the details.

## 1.1.0 — 2026-10-02

⚠ **The x86-64 build now needs an `x86-64-v2` CPU**: `POPCNT` and SSE4.2, which
means any Intel processor since 2008 and any AMD since 2011. RHEL 9 already
requires the same. An older processor stops with `Illegal instruction` at
startup; it does not give wrong answers. The server logs
`similarity kernel: popcnt` when it starts. No rebuild is needed, and no answer
moved: none of the pinned hit sets changed.

**Similarity is faster: −28.3% per search with `--fp-codec none` and −4.1% with
the default `zstd`** (`perf-compare.sh` on this machine, with every other
benchmark inside its own spread). The fingerprints are now read in place
instead of being copied, and the 512-bit comparison is laid out at its fixed
width so the compiler can vectorise it. Scores are unchanged bit for bit: every
row was checked against the reference at three widths and with both codecs.
⚠ With the default codec, about 85% of a similarity scan is decompression,
which is why the default moves less. A faster fingerprint encoding is planned
for 2.0.

**Fixed: a damaged index answered as though it were whole.** An index opens
without reading its stores end to end. When a search walked into a damaged
block, it stopped there and returned what it had: one overwritten byte made a
similarity search score 2 003 of 3 004 rows, with no warning. A selective
search could return no hits at all. Now the first search that finds damage
logs the file, the block and what was wrong, once, at error level. The shard
is then **served as missing**, exactly as a shard whose directory has gone:
answers cover the remaining shards, the warning names the damaged one, and a
database with no shards left is refused. A rebuilt shard serves again at the
next reload or restart. `index delete` refuses to run over a damaged store
rather than reporting identifiers as not found. ⚠ **Not yet detected**: damage
that leaves a record's length intact, such as a flipped bit inside a
fingerprint. Block checksums for that are in the 2.0 index format.

**Also fixed.**
- If one of a parallel scan's worker threads failed, the blocks it was
  scanning were silently dropped, and the answer counted fewer rows than it
  claimed. Such a search now returns `500 Internal Error` instead of a short
  answer.
- Logs written to a file, journald or `docker logs` contained terminal colour
  codes. Colour is now used only when the output is a terminal.

## 1.0.0 — 2026-10-02

**The first major release. The code is identical to 1.0.0-rc.1; only the
version number changed.** No rebuild is needed, and no answer moved. Every
index built since 0.8.0 opens.

**What the number promises is [`COMPATIBILITY.md`](COMPATIBILITY.md)**, and
here is the short form. For all of 1.x, the wire, the command line, the
configuration, index format v3 and the metric names stay as they are. A
chemistry fix that changes an answer can only ship in a minor release, and
only under its own *Answers moved* heading; a patch release never changes an
answer. What counts as a ring will not change before 2.0.

**Upgrading from 0.x.** Nothing to rebuild and nothing to reconfigure. If you
skip 0.24.0, read its entry: the row column changed on databases with more
than one shard. If you run parts, upgrade them before the coordinator. Then
read the 1.0.0-rc.1 entry below, which lists everything new in 1.0.

**What 1.0 does not claim.** The largest collection served from one machine
is 124 427 347 molecules. A billion-molecule run was in the original
definition of done; it is deferred to 1.x by decision, and has not been done.
Throughput across two machines has not been measured. And as before, **there
is no authentication**: the port is the security boundary (README §
*Security*).

## 1.0.0-rc.1 — 2026-09-25

**The release candidate for 1.0. No rebuild, no answer moved, and from here the
version number is a promise: [`COMPATIBILITY.md`](COMPATIBILITY.md) says what
it covers.** In short: the wire, the command line, the configuration, index
format v3 and the metric names are stable for all of 1.x. A chemistry fix may
change an answer in a minor release, and only under an *Answers moved* heading.
The ring basis will not change before 2.0.

⚠ **Fixed: under full load, the server could not answer anything else.**
Searches did their blocking work on the threads that answer every request.
That work is the wait for a search slot, the similarity scan, the wait for
first results and the rendering. A reload did the same while building. The
search pool defaults to one search per core, and so does the thread count, so
when every slot was busy even a request that needs no slot had to wait. That
included a health probe, and an orchestrator kills a server whose probe fails.
Blocking work now runs on threads of its own. A server whose every slot is busy
answers its probes in milliseconds.

**A stop drains instead of killing.** `SIGTERM` or `SIGINT` first makes
`/readyz` answer `503` for `--shutdown-delay-seconds` (default 0) while the
server still accepts. Then it accepts nothing new, and lets what is in flight
finish for up to `--shutdown-grace-seconds` (default 30). The log says whether
every connection closed or the grace ran out. A second signal exits at once
with `130`. ⚠ **Give your supervisor longer than delay plus grace.** Docker's
default is 10 seconds: use `docker stop -t 40`.

**Probes and metrics.** `/healthz` is liveness and `/readyz` is readiness. A
degraded database does not fail readiness, so one missing shard does not take
the others out of rotation. `/metrics` is Prometheus text. It counts requests
by route template, method and status, with a duration histogram, plus searches
by type and status, pool occupancy, and live rows and degraded state per
database. ⚠ No label is taken from what a client sends: routes are templates,
methods are one of seven or `other`, and databases are the ones you serve.
**No metric carries a query.** README § *Operating it* has Kubernetes and
systemd examples.

**`--access-log`** writes one line per request: method, route template,
database, status and time. ⚠ **It never logs the query.** A test drives real
searches and finds the query in the log zero times. Off by default.

**Fixed: an interrupted upload could be built as a truncated collection.**
`POST /dt/upload` wrote straight to `<name>.smi`, which is the name
`--build-dir` builds. An upload that was cut off by a kill or a full disk left
a truncated file, and the next reload served it as a database. Now an upload
is written under a hidden name and moved into place only when it is complete.
A hidden file that nobody has written to for ten minutes is treated as
abandoned and removed. On a filesystem without hard links, such as exFAT, SMB
or most FUSE mounts, it is renamed into place instead.

**Smaller things.** `moleculo --version`. A request whose handler fails
answers `500 Internal Error` in the usual error shape; before, the connection
was dropped. The container image has a `HEALTHCHECK` on `/readyz` and an
explicit `STOPSIGNAL`. The README's settings table was missing four settings:
it is now complete and checked against the parser by a test.

**What is not known, and not promised.** The largest collection served from one
machine is still 124 427 347 molecules. The billion-molecule run is moved to
1.x by decision, not done. Nothing has been measured on two machines. A drain's
count of requests in flight is a floor: a client that has sent half a request
holds a connection with nothing being answered.

## 0.24.0 — 2026-09-25

**No rebuild. The index format is written down, a coordinator can tell when its
parts run different builds, and the row column in a result is now unique across
the whole database.**

**The index format is specified: [`INDEX-FORMAT.md`](INDEX-FORMAT.md).** It
covers the header, the file kinds, `database.json`, the manifest, the block
store, the row id and the deletion bitmap. That is enough to write a program
that lists what a database holds without this software. To test it, a reader
was written from the document alone, with no access to the source. It agrees
with `index inspect` on every shard, and with a served search on the hash of
the live identifiers, including on a database that had been appended to and
deleted from. **Version 3 is frozen.** It is the format 0.8.0 introduced, so
nothing you have built needs rebuilding.

**`geometry=1` is the `qopts=G` lock under a parameter name of our own.** Both
spellings resolve to the same lock. ⚠ **Keep writing `qopts=G`.** Arthor ignores
parameters it does not know, so a client that sends `geometry=1` to Arthor gets
an unlocked answer and no error. `qopts=G` gets a `400` there, which is the
answer you want. The parameter exists for the day Arthor gives `G` a meaning of
its own. `qopts=G&geometry=0`, a value other than `1` or `0`, and `geometry` on
a similarity search are each a `400`.

**Every response carries `x-moleculo-version`, and a coordinator warns when a
part runs a different build.** A merged answer names the part and both versions
in `warning`. It warns rather than refuses, so you can upgrade machines one at a
time and the database keeps serving. ⚠ Read the warning anyway. A 0.23.0 part
ignores `geometry=1` completely and returns the unlocked answer: 207 rows where
this build returns 101 on the same index. Parts older than this release send no
version at all, and the coordinator names them for that too. **Upgrade the parts
before the coordinator**, or accept that the warning fires until you have.

⚠ **The second column of a result is now the row id,
`(shard << 32) | local`.** On a single-shard database, which is the default,
that is the same number as before, and every recorded fixture still passes. On a
database with several shards, rows after the first shard now carry values from
4 294 967 296 up. Before, the column restarted at 0 in each shard, and the page
keys a marked molecule by that column. So marking a row in one shard could
silently unmark a different molecule in another. Two clicks on a two-shard
database used to leave **0** marks and now leave **2**. Arthor restarts its
numbering per index part, so this is a deliberate divergence from it. If you
kept those numbers anywhere, switch to the identifier: a row id is only valid
within one index generation. A mark saved in the page by an earlier release on
a multi-shard database still exports the right molecule, but it can highlight
the wrong row. Clear marks once after upgrading.

**What is not known.** Nothing about two parts on two *machines* has been
measured. The coordinator's version check compares version strings, so two
builds with the same number and different code look the same to it. The largest
collection served on one machine is still 124 427 347 molecules.

## 0.23.0 — 2026-09-11

**No rebuild. A database can now grow and shrink without being rebuilt, and one
field on the wire changes what it counts.**

**Adding to a database without rebuilding it.** `moleculo index append <db>
<input.smi>` turns the new molecules into one more shard and touches nothing
already there. The result is the index a full rebuild of both inputs would
have written — **byte for byte, all ten files of every shard**, measured on the
first 200 000 lines of ChEMBL cut in two — and existing row identifiers do not
move. A database built as a single shard, which is the default, is converted by
**hard-linking** its files into the several-shard layout rather than copying
them, and swapped in with one rename.

**Deleting rows.** `moleculo index delete <db> --id ID` or `--ids-file FILE`
names molecules by the identifier in your input, never by row id — a row id
means something only until the next rebuild. A deleted row is gone from every
answer: over the first 200 000 lines of ChEMBL — 199 997 molecules — with 10 001
deleted, the database answers
**56 of 56 corpus queries identically to the same corpus rebuilt without them,
in every field**. An identifier that matches nothing is named in the output. ⚠
**Deletion frees no disk space until a rebuild**, and `index inspect` now shows
the built count beside the live one so you can see what a rebuild would reclaim.

⚠ **`recordsTotal` now counts the rows a client can still retrieve**, not the
rows the index was built from. On a database with nothing deleted — every
database before this release — the two are the same number and nothing you see
changes. It has to count live rows: a similarity search promises that
`recordsFiltered` equals `recordsTotal` and that the histogram sums to it, and
counting built rows would break both at once on the first deletion.

**Fixed, and it was in every earlier release: a repeated search after a reload
answered from the collection that was there before.** A database that gained a
shard and was reloaded kept answering an *identical* request with the old count
— 83 710 hits over 100 486 rows, beside a `recordsTotal` of 199 997 in the same
response — while any other query saw the whole database. The result cache was
keyed by the database's name, and the name is what did not change; it is keyed
by what the database holds now. Nobody had seen it because nothing had ever
changed a database under a running server before `--build-dir` and reload made
it routine.

⚠ **Neither command has an HTTP route, deliberately.** The server has no
authentication, and a destructive route in front of it would be a delete button
for anyone who can reach the port. Both run on the machine that holds the index,
and a running server sees the change at its next reload — pass `--notify` to
ask for one.

**What is not known.** The deletion check runs on every row of every scan; with
nothing deleted it costs a bounds check, and its cost with a full bitmap on a
billion-row shard has not been measured. On a 7 420-molecule benchmark the
difference was below what this laptop can resolve, which is no measurement, not
no cost.

## 0.22.1 — 2026-09-09

**No rebuild; only `fmt=sdf` output changes.**

**`fmt=sdf` states the stereochemistry it used to lose on sulfoxides,
bridgehead nitrogens and fused cyclopropanes.** Over the five populations the
README names, the records that stated *less* stereochemistry than the source
go from **134 to 8** of 76 094, with 0 stating anything false before and after.
Two mechanisms, both the writer's: a three-coordinate centre with a lone pair
— `[S@](=O)`, `[S@+]`, a nitrogen at a bridgehead — was never marked, because
the wedge chooser wanted four drawn directions and a lone pair is not drawn
(78 descriptors); and a three-membered ring fused on another was built with
its apex on the ring atom it was fused to whenever the shared bond wrapped the
ring's order (46). The second also changes drawings that carried no
stereochemistry: about 6.5 % of ChEMBL records draw differently, and records
with two atoms at one point fall from 21 % to 16 % of ChEMBL — ⚠ which says
the layout is legible, not pretty, and that figure is new and not a promise.
The 8 that remain are bridged cages and one crowded peptide.

## 0.22.0 — 2026-09-07

**No rebuild. A database can now be served from more than one machine, and the
warning about who may reach a server grows a second paragraph.**

**A database on more than one machine.** A collection that does not fit one
box is served as *parts*: every part is an ordinary `moleculo serve` holding
some of the shards, and a coordinator — the same binary, holding nothing of
its own — names them and answers as if it held everything. Name them on the
command line, `--part big=one@http://one:8080 --part big=two@http://two:8080`,
in shard order, or in `<root>/big/parts.json`; a flag wins over the file, and
a database is local *or* parts — both at once is refused at startup with the
name. The coordinator forwards every request to every part over the same
protocol you use, and merges: hits in the order one server would return them,
counts summed, a similarity re-ranked by score, `partStats` keyed by part
instead of by shard. A page across the seam between two parts costs each part
one request. Warming, evicting, uploading and reloading are done on the part
that holds the disk.

⚠ **One thing a coordinator answers differently, and it is declared rather
than hidden:** a similarity's ties. The wire carries a score to three decimals
and one server ranks by the exact one, so among rows a coordinator cannot tell
apart it orders by row index — 22 rows in seven ties of a `length=100` ranking
on world-drugs, swapped inside their tie and nowhere else, and a page that
ends inside a tie can show different members of it. The set inside a tie, the
scores and the histogram are identical.

**When a part does not answer** the database is *incomplete* in the same shape
a missing shard makes it: the warning names the part and the reason, the count
is a floor, `verified` is how many rows the answering parts looked at. ⚠ That
last number read **0** in the first build of this release — the test had asked
only whether the field existed, and the release check read it — so the figure
is now asserted, not its presence. The same check found an identity reply
carrying `"warning": null` where one server omits the field; it is omitted now. A similarity over a partial
collection is refused with `503` and a negated search with `400`, because a
ranking over some of the rows is not a ranking and the complement of a floor is
wrong permanently. `/dt/data` lists the parts without asking them — the first
version asked, and a coordinator hung on its own parts for it.

⚠ **THE PARTS MUST BE REACHABLE ONLY FROM THE COORDINATOR.** A part is a full
server with the same unauthenticated surface; query strings travel to it over
plain HTTP. The README says this in capitals and the coordinator prints it at
startup. Two lines of this README that said the software sends nothing anywhere
now say *except to the parts it was told to serve*.

⚠ **Demonstrated, not measured.** The test holds four servers in one process —
two parts, one server holding both their shards, one coordinator — to identical
answers on every corpus query, a page across the seam, a similarity, an
identity and all three text formats, then stops a part and checks the degraded
answers. Nobody has run it across two machines and there is no throughput
figure for doing so. **Shard order across parts is your promise, not a check**:
the wire carries a row's index within its shard and never the shard, so a
coordinator given the parts out of order returns hits out of order and cannot
know.

New in the binary: `ureq` 3 as the HTTP client, without TLS on purpose —
a part is inside your perimeter by contract. Connect timeout 5 s, a whole
exchange bounded by the search's deadline plus 5 s, response bodies capped at
256 MB.

**What has not changed.** Every hit set, identity key, aromatic perception and
ring set is byte-identical to 0.21.0; the index format is unchanged at version
3 and nothing needs rebuilding.

## 0.21.0 — 2026-09-07

**No rebuild. The identity key stopped hearing a slash that cannot mean
anything, one limitation in this README was eleven releases stale, and two more
questions with no oracle read zero at scale.**

**A directional mark that cannot carry a configuration no longer moves the
identity key.** `/dt/{db}/identity` answers from a key computed over the
molecule, and until now a `/` or `\` on a double bond went into it whatever the
bond could actually mean. Two places a mark means nothing: a double bond inside
a ring of fewer than eight atoms, whose geometry the ring forces; and a double
bond one of whose ends carries two substituents that are the same to chemistry
— `C/C=C(/C)C`, or a nitronate `C=[N+]([O-])[O-]` — where *cis to which one*
has no answer. `RDKit` assigns no stereo to either and its writer still emits
marks on them in a random walk, which is how one molecule of 40 000 at 25
spellings came back with two keys: a cyclohexenyl nitronate, 23 spellings one
key and the marked spelling another. The key now ignores both, by `RDKit`'s own
two rules, and the symmetry test uses the equitable refinement rather than the
canonical rank — the rank's ties have been broken so that a string can be
written, and it tells two methyls apart. ⚠ **Keys are not stored in the
index**, so nothing needs rebuilding. Measured: **0 keys moved on all 2 897 819
ChEMBL rows** (ChEMBL writes no such marks) and **0 of 40 000** unstable at 25
spellings after. A mark that does mean something still counts: cyclooctene
keeps its trans, and methyl-against-ethyl keeps its side.

⚠ **Two limitations in this README were stale, one for eleven releases.** *A
hydrogen on a charged sulfur is dropped from an SD file* had been fixed by
0.9.0's valence field and stood as live since; it is struck through with the
reason. *One molecule in 78 935 leaves `fmt=sdf` as a different isomer* was
measured over two ChEMBL samples nobody recorded how to draw; over five
populations anyone can rebuild it is **0 of 76 094**, and the entry now says
that the one — a fused macrolactam — is a real class whose rate is not known.
Both found by re-deriving every figure whose population was never written
down, which is the pass that found the bare-metal defect in 0.20.1.

**Two more zeros, at the scale they are about**, with the instruments
committed: the parallel scan returns the serial scan's answer on **56 of 56**
corpus queries over an 8 476 919-row PubChem shard — hits, rows verified and
rows undecided — where the test had only ever said so on 3 004 rows; and
`/dt/{db}/identity` finds **1 001 of 1 001** sampled rows of a 2.9 M ChEMBL
shard through the served endpoint, every answer complete. ⚠ A zero is a
statement about the population it was read on, and both are named.

**Unchanged:** no format bump, no vocabulary change, no change to any hit set,
aromatic perception or SD output.

## 0.20.1 — 2026-09-06

**No rebuild. One fix to the SD file writer, and an instruction if you exported
bare metals.**

**A bare metal came back from an SD file as its hydride.** `[Na]`, `[Li]`,
`[Cs]`, `[Ra]` — any atom outside the organic subset with no bonds and no
hydrogens — has a total valence of zero, and the writer put `0` in the atom
line's valence field. In V2000 a written zero means *not stated*, so a reader
derives the valence itself; RDKit's default table says sodium takes one bond,
radium two, and a lithium citrate written as three lithium atoms came back as
three lithium hydrides. **Three records in a 100 000-line ChEMBL prefix, each a
different compound on the way back.** V2000's code for zero valence is **15**,
and it is written now for every such atom, charged or not. Measured after the
fix over six populations — world-drugs, real-space, mcule, two 100 000-line
ChEMBL prefixes and the PubChem 100 000 sample — **0 records return as a
different graph**, and nothing else in the SD output moved.

⚠ **If your collection holds bare metals and you exported SD files with 0.20.0
or earlier, export them again.** Search was never affected: the molecule stored
and matched is correct, and so is every other output format.

⚠ **How it was found, because the method is the point.** The README's
"structures that come back as a different molecule are now 0 in 100 000" was
measured on a PubChem sample that holds no bare metal, and had stood for three
releases. Re-deriving a *different* row of the quality ledger — one whose
population nobody had recorded how to draw — over named ChEMBL prefixes turned
the three up as a bucket the diff tool already had and nothing had ever filled.

**Unchanged:** no format bump, no vocabulary change, no rebuild, no change to
any search or to the SMILES, TSV or CSV outputs.

## 0.20.0 — 2026-09-04

**No rebuild. Nothing about your index or your answers changes — this release is
speed, and two things learned about the product that the README now says.**
Every hit set, identity key, aromatic perception and ring set is byte-identical
to 0.19.0, asserted by the goldens and by two population comparisons rather than
believed.

**A substructure scan is a fifth faster, a ring query a quarter, and a scan on
four threads nearly a third.** Three changes, each measured against the commit
before it in four alternating rounds on one machine — the only comparison this
project accepts — and compounded here, because an absolute millisecond on this
laptop is not a number that travels:

| | change 1 | change 2 | change 3 | together |
|---|---:|---:|---:|---:|
| benzene over 7 420 molecules | — | −6.1% | −13.6% | **≈ −19%** |
| an ester query | — | −5.6% | −15.5% | **≈ −20%** |
| `[R2]`, which forces full ring perception | −23.0% | — | −3.5% | **≈ −26%** |
| benzene on four threads | — | −18.1% | −15.1% | **≈ −30%** |
| decoding alone, no matching | — | — | — | unmoved, as the control |

A dash is a verdict withheld: the change was smaller than that benchmark's own
spread. Similarity and screening are unchanged; nothing on those paths was
touched.

**What was actually done**, in three steps, each predicted before it was run:

- **A benzene scan allocates nothing per row.** It allocated 5.8 times per row
  and every one was scratch the next row needed again at the same size: the
  adjacency index a reused molecule rebuilt from nothing, the valence array of
  the hydrogen pass, the matcher's two search buffers. All three are kept now.
  ⚠ The parallel scans gained two to three times what the serial ones did —
  eight workers freeing the same three buffers per row made the allocator a
  shared cost no single-threaded profile could show.
- **The hydrogen pass runs only when the query reads a count.** Filling in the
  hydrogens a record does not store is a sweep over every bond of every row,
  and `c1ccccc1` never reads one. The scan now asks the query first; `H`, `X`,
  `v`, an enforced stereo check, a recursive subpattern and a formula still
  fill. This replaced the alternative of storing the counts in the index, which
  would have cost every operator a rebuild for the same gain.
- **Ring perception walks ring bonds only, and keeps its candidates in two
  arenas.** The shortest-cycle search was exploring every substituent of a
  drug-like molecule before finding its way round the ring, and allocating a
  pair of vectors per ring bond to hold what it found.

**Two things the README now says that it did not, both measured this week:**

- ⚠ **The screening index plans one corpus query in fifty-six on a real
  PubChem shard**, against four in fifty-six on the 200 000-row prefix it was
  judged on. Every hit set is still identical to the scan's; the features the
  other three need are simply above the density ceiling on the real
  collection. How much your index screens is a property of your collection,
  and the README says so where the four-in-fifty-six figure used to stand
  alone.
- ⚠ **The minimal ring basis costs something on ordinary chemistry**, not only
  on fullerenes: about one PubChem molecule in a hundred has a ring set that
  depends on how it was written, and on those `[R3]` — an atom in three rings
  — answers by the spelling the collection stored, on about 0.12% of
  molecules. `[R2]`, `[r5]` and `[r6]` never do. The ruling to keep the minimal
  basis stands; the README's limitation now carries this price beside the
  fullerene one.

**Unchanged:** no format bump, no vocabulary change, no rebuild. Every
limitation in the README stands where 0.19.0 left it, plus the one above.

## 0.19.0 — 2026-09-03

⚠ **Rebuild your indexes when convenient; nothing forces it.** The format is
unchanged at version 3 and every existing index opens and answers. Three things
in this release change what is *stored* — which molecules the parser accepts,
which bonds are aromatic, and which rings a cage has — and an index built
earlier keeps the old answer on exactly those rows. Measured, the rows are few:
**1 in 2 897 819** of ChEMBL reads differently, **39 in 9 999 789** of PubChem,
plus **10 in 500 000** whose rings change without their counts changing, and a
handful of molecules per million that older readers refused outright and are
therefore absent from an older shard.

**The reader stopped consulting the perceiver.** A valid SMILES that RDKit's own
writer produces — `c1ccccc#1`, `c1cc[siH]cc1`, `c1cc[n+](-[b-]2ccc3ccccc3c2)cc1`
— came back as a `400`, because the table that turns a lower-case string into
alternating bonds had no entry for an atom this build would never itself call
aromatic. Reading a string and deciding what it means are different jobs. Three
entries and one element symbol, every cell run against RDKit: a triple bond
fixes an atom exactly as a double one does, silicon shares carbon's rule, `[b-]`
is isoelectronic with carbon. ⚠ **The chemistry did not change** — those rings
still read as *not* aromatic, which is what Arthor answers; RDKit alone calls
them aromatic. **6 of 499 971** PubChem molecules refused in one spelling and
accepted in another are now **0**; ChEMBL's four aromatic-boron refusals read;
and a triple bond inside a thiophene, which the README had defended refusing,
reads too, as it does in Arthor. **On ChEMBL, the molecules this build refuses
are now exactly the ones RDKit refuses.**

**Five ring-fusion bonds read single where both references read aromatic.** In
a fused system where two rings fail the aromaticity test alone and a third
passes, the combination that puts their shared bond on an aromatic circuit was
never examined, because every ring in it was already settled by another. A
settled ring is not a marked bond. ⚠ **The same skip had made those five
molecules' identity keys depend on how they were spelled** — two keys each,
now one. The four fusion bonds that remain all sit in one tetra-azo molecule
where Arthor reads every bond as this build does.

**The ring basis held things that were not rings.** The fallback that finds
rings in bridged cages dropped the first atom of its walk, and nothing checked
that the last atom was bonded to the first — so a five-ring missing one atom
was stored as a four-ring, and it displaced a real ring. **A `[r3]` query used to
match a pentaprismane, which has no three-ring.** Ten molecules in 500 000 held
such a ring; every one is a cage or a bridged polycyclic. One fullerene adduct
that read one aromatic atom fewer than both references reads correctly now;
three strapped porphyrins read as Arthor reads them where RDKit gives seven
different answers depending on spelling. ⚠ Ring *counts* moved nowhere — this
was always the right number of rings, one of them wrong.

**A ruling recorded, not a change.** The owner has decided to keep the minimal
ring basis rather than adopt every relevant cycle. What that keeps: every `R`
and `r` ring primitive answers as it does today on the 1.39% of PubChem where
the two definitions disagree. What it costs, now stated in the README as a
declared limitation rather than an open question: on fullerene cages the ring
choice follows the spelling on 21 of 22, `identity` is unreliable there as a
class, and one fullerene adduct reads 65 aromatic atoms where both references
read 66.

⚠ **Three numbers in the README were wrong and are corrected, each with the
reason.** *Refused where RDKit accepts: 5* was measured on a corpus that had
been recovered from a shard and so could not hold a refusal at all. *455 of
497 693 undecidable rows* was a sample nobody recorded, 150× out; the figure is
30 of 9 999 789. *84.3 bytes per molecule* for the 124 M index was a number no
other document carried; it is 92.5, built by 0.17.0.

**Unchanged:** no format bump, no vocabulary change (the instance was asked:
`CINQRS`, no gaps, and the collision on `G` that the 2026-08-25 sweep found is gone — the instance no longer accepts it). Speed is
untouched. Every limitation in the README stands where 0.18.0 left it except
the three closed above.

## 0.18.0 — 2026-09-01

**No rebuild. Nothing about your index or your answers changes — this release is
speed.** After two versions that asked for a rebuild, this one asks for nothing:
every hit set, identity key, aromatic perception and molfile signature is
byte-identical to 0.17.0, and that is asserted by tests rather than believed.

**A substructure search is 17–23% faster, and decoding is 26% faster.** Measured
against 0.17.0 on the same machine in one sitting, four alternating rounds:

| | 0.17.0 | 0.18.0 |
|---|---:|---:|
| benzene over 7 420 molecules | 16.4 ms | **12.7** |
| an ester query | 13.8 | **11.3** |
| `[R2]`, which forces full ring perception | 61.8 | **50.9** |
| the same on two threads | 10.3 | **8.2** |
| decoding alone, no matching | 7.2 | **5.3** |

Similarity and screening are unchanged, because nothing on those paths was
touched.

**What was actually wrong.** Three things, each an allocation nobody had looked
for, found by profiling rather than by reading:

- the matcher allocated a vector **on every node of every search of every row**,
  to hold the candidates it was about to loop over;
- decoding a molecule built its **entire adjacency index** — three allocations —
  to compute hydrogen counts, a question that is one pass over the bonds, and it
  did that before anything had asked whether the row was even a candidate;
- ring perception allocated three vectors **per ring bond**, where one set per
  molecule does.

⚠ **The honest shape of it**: a profiler said the allocator was 40.7% of a ring
query's time, and the first two readings of *which* allocation mattered were
wrong while being right that allocation did. What worked was reading the call
path rather than the largest-looking data structure.

**Unchanged:** no format bump, no vocabulary change, and every limitation in this
README stands where 0.17.0 left it — the fullerene identity limitation, the 202
documented aromatic divergences, the rows too branched to decide, and the six
molecules per 500 000 still refused in one spelling and accepted in another.

## 0.17.0 — 2026-08-31

⚠ **Rebuild your indexes when convenient.** The format is unchanged at version 3
and every existing index opens and answers. This release makes the parser accept
molecules it used to refuse, and a molecule refused at build time is not in the
shard at all — so if your collection writes those structures in the aromatic
form, an older index is **missing** those rows from every answer and no query can
reach them.

⚠ **On PubChem that is zero rows, and the number is here because it was measured
rather than predicted.** All 124 427 347 molecules were rebuilt at 0.17.0 and the
count came back **exactly the same**: PubChem writes these structures Kekulé
(`C1=CC=[C-]C=C1`), which every earlier version already read. The rows this
release adds are the ones a collection writes *aromatic*. What a rebuild buys
every collection is the corrected perception and 0.16.0's working screen — and
⚠ **it costs 7.1% more index**, 86.37 → 92.54 bytes per molecule on that build,
all of it in the screening slice, which is what 0.16.0's fix indexes twice.

**A molecule is no longer accepted or refused depending on how it was drawn.**
An aryl carbanion — a phenyl anion, a Grignard, an aryllithium — is written
`C1=CC=[C-]C=C1` by PubChem, which we read, and `[c-]1ccccc1` by RDKit, which we
answered with a `400`. One structure, two answers, decided by the notation. Over
the first 500 000 lines of PubChem at six spellings each, **45 molecules were
split that way; 6 remain**, and those six are three different things: an aromatic
ring holding a triple bond, aromatic silicon, and an aromatic borate.

⚠ **The rule, for anyone who needs to predict it.** A charged aromatic carbon
donates its lone pair to the ring — and so takes no double bond — only when it
carries a hydrogen or a third substituent, which is cyclopentadienyl and
tropylium. With two ring bonds and no hydrogen the charge sits in the orbital the
missing hydrogen left, the ring keeps its six π electrons, and the atom still
needs a double bond. Read the other way it left five carbons needing one, an odd
number, and the molecule could not be kekulized at all.

**Nothing else moved, and that was measured rather than assumed.** Identity keys
over the three reference corpora — 22 423 molecules — are unchanged, and so are
the keys of all 67 ChEMBL molecules that hold a charged aromatic carbon. The rule
is consulted only for aromatic atoms and differs only for a charged carbon, so a
molecule without one is bit-identical. The whole decision boundary was checked
against RDKit 2026.03.5: thirteen cases, thirteen agreements, including the five
it refuses where we refuse too.

⚠ **A claim in this README has been corrected rather than quietly dropped.** It
said "40 in 100 000 are refused in one spelling and accepted in another". A
measurement on a named population first read **zero** — and that zero was wrong,
because every spelling in it was generated by RDKit and therefore aromatic, so
the Kekulé form that disagrees was never in the sample. With the collection's own
string included it reads 10 in 100 000, and the original claim was right in
substance all along.

**Unchanged:** no format bump; hit sets, counts and every search type answer as
they did; the 202 documented aromatic divergences, the fullerene identity
limitation and the rows too branched to decide are all where 0.16.0 left them.

## 0.16.0 — 2026-08-30

⚠ **Rebuild your indexes when convenient — nothing breaks if you do not.** The
format is unchanged at version 3 and every existing index opens, searches and
answers correctly. What changed is that a substructure search can now **reject
rows before decoding them**, and the shard has to have been built by this
version for it to happen: what the stored feature strings mean changed, the
shard records which meaning it was built with, and a planner that does not
recognise it declines rather than guessing. An older index only scans.

**A substructure search skips rows it can prove cannot match.** Every shard
already carried an index of the features its molecules hold; until this release
the planner could not build a plan from an ordinary query, so nothing was ever
skipped and every row was decoded and matched. Measured over 199 977 rows and
the 56 corpus queries, with every hit set asserted identical either way: four
queries get a plan and all four win — `S(=O)(=O)N` reads **3 801 candidates for
3 712 real hits**, `FC(F)F` 4 673 for 3 904, `c1cn[nH]c1` 3 264 for 405, and
`c-c` is exact at 10 724.

⚠ **Two defects, one on top of the other, and the second was only reachable once
the first was fixed.** A requirement needed a *concrete bond*, and every bond an
ordinary SMILES query writes is implicit — so `S(=O)(=O)N` could name no feature
at all and the planner returned nothing on **56 of 56** corpus queries, for the
life of the project. The first plan it built then dropped **17 real hits of
3 712**, because formal charge was in a molecule's stored feature while no query
can ask about a charge: every sulfonamide spelled `[N-]` was invisible. Charge is
gone from the vocabulary — the knob that carried it was deleted rather than
defaulted off, because a setting with an unsound position should not exist.

⚠ **What still scans, and it is not a budget question.** The other 52 corpus
queries are refused a plan and that is correct: most are bracket SMARTS the
planner cannot read, and they name things too common to be worth skipping. **A
fused ring is unscreenable by construction** — naphthalene's short paths are the
same runs of aromatic carbon as benzene's — so `c1ccc2ccccc2c1` scans however
your index was built.

**An abandoned search is no longer reported as an absent match.** The matcher
abandons a molecule after a step budget, correctly, since an index build must not
hang on one row. Two paths still folded *we gave up* into *there is nothing
here*:

- **A highlight.** A hit with no highlight meant both *we looked and there is
  nothing to draw* and *we ran out of steps*. The rows were always hits and still
  are; the answer's `warning` now says, in its own sentence, how many hits on the
  page have no highlight because the search hit its budget. The `highlights`
  array keeps exactly the shape it had.
- ⚠ **The identity key, which is the one that could return a wrong answer.**
  `/dt/{db}/identity` standardises a molecule by searching for the patterns of
  the rules that might apply. A search that was abandoned came back
  indistinguishable from a rule that does not apply, so the rule was skipped and
  the molecule keyed as though it had been checked — and a quietly different key
  answers *"we do not hold this"* about a compound the collection holds.
  Standardisation now **refuses** that molecule instead, naming what it could not
  decide.

⚠ **This has never fired on any collection we can build, and that is said rather
than glossed.** Instrumented over `world-drugs`, `mcule`, `real-space` and
500 000 PubChem molecules: **42 million searches, zero abandoned**, the deepest
1 332 steps against a budget of 1 000 000. It is fixed because the class of
answer was wrong, not because anyone saw a wrong key. Control: 122 423 molecules
re-keyed either side of the change, **not one key moved**.

**The identity limitation on fullerenes is now stated with its price.** The
README said the key moves for "2 in 40 000" molecules, which reads as rare and
nearly fixed. That is how often a fullerene lands in a random forty thousand, not
how often the defect fires: on the population it is about — 22 bare carbon cages
at 24 spellings each — **21 of 22 give more than one key** and one gives
thirteen. If your collection is fullerene chemistry, treat `/dt/{db}/identity` as
unavailable rather than rare. The alternative is now measured too: the ring
*count* on a fused cage is stable, the *choice* of rings is not, and adopting the
stable definition would change the ring count on **1.39% of PubChem** and with it
every answer built on ring perception. That is a decision nobody has taken, so
the limitation stands.

**Unchanged and still true.** No index format bump. Hit sets, counts and every
search type answer exactly as they did — the only search-visible additions are
the new warning sentence and the standardisation refusal that has never
triggered. The 202 documented aromatic divergences, the 40 in 100 000 spellings
refused where another spelling is accepted, and the rows too branched to decide
are all where 0.15.0 left them.

## 0.15.0 — 2026-08-30

⚠ **Rebuild your indexes when convenient — nothing breaks if you do not.**
The format is unchanged at version 3 and every existing index opens and answers.
But this release changes which bonds are perceived aromatic, and an index stores
that perception rather than recomputing it, so a shard built before 0.15.0 keeps
the older answer on **about one row in five thousand five hundred**. Measured
over 500 000 PubChem molecules: 90 rows differ, on 144 bond flags, and **not one
atom flag**. Only a query that asks about bond aromaticity can tell.

**A fused ring system marks its aromatic bonds the way the reference does.**

Aromaticity in a system of several fused rings is decided over *combinations* of
those rings, and a bond that lies inside one aromatic combination can lie on the
edge of another. This build used to decide once, from the first pair it found,
and never revisit — so such a bond was recorded as not aromatic even when a
second combination put it on a genuine circuit.

Measured against RDKit over 398 040 ring-fusion bonds in 500 000 molecules,
disagreements in the affected class go from **145 to 9**. Over the 1 527
molecules that hold such a bond, full agreement goes from 1 459 to **1 521**.
Nothing else moved: the documented macrocyclic divergences stay at 185, three
committed corpora stay at 100.000%, the SMILES round trip over 500 000 molecules
is still exact, and 168 corpus-query hit sets are byte-identical.

⚠ **Aromatic *atoms* did not change**, here or anywhere in that measurement. If
you have been reading atom aromaticity, this release is invisible to you.

**A formula query can now find a molecule with a wildcard atom in it.**

A wildcard — `*` in SMILES, the atom that stands for anything — was dropped from
the formula this build writes and refused in the formula it reads. So
`*C(=O)Nc1ccccc1` was described as `C7H6NO`, which is a formula for a different
molecule, and no formula query could reach that row, not even one written from
the row itself. The formula now carries it, `C7H6*NO`, in the position RDKit
uses; `*3` asks for exactly three wildcard atoms, the same way `C3` asks for
exactly three carbons.

⚠ **If your collections are ordinary molecule files this changes nothing** —
there is not one wildcard atom in any corpus this project measures against. It
matters for Markush structures and R-group catalogues.

**What still does not work.**

- The two fullerene cages whose identity key depends on how they were written
  are still there, and on fullerene chemistry generally the endpoint remains
  unreliable — 21 of 22 bare carbon cages give more than one key.
- Nine ring-fusion bonds in 398 040 still disagree with RDKit, down from 145.
- Everything the 0.14.2 notes list as outstanding is still outstanding.

## 0.14.2 — 2026-08-29

✅ **No index needs rebuilding**, and nothing about how a molecule is stored or
searched moved. This release is about answers the previous ones gave that they
should not have given.

⚠ **One thing that used to answer now refuses.** If you negate a search — `qopts=N`
— over a collection holding a row the matcher could not decide, you get an error
instead of a result. That is deliberate, and the paragraph below says why.

**A negated search no longer hands back rows it cannot vouch for.**

The matcher abandons a molecule that is too branched to decide inside its work
budget, and reports those rows apart, as `undecided`. But a negated search asks
for the *complement* — the rows your query does **not** occur in — and an
undecided row was being counted as a non-match, so it was returned to you as a
row your query is absent from. It might well contain it; nobody looked.

That is a wrong row in your results, not a missing one, which is why the answer
is now a refusal naming how many rows it could not decide. ⚠ **Raising the work
budget is not offered as the fix** — it trades a silent wrong answer for a slow
one, and the bound is what stops an index build hanging on a single molecule. Run
the search without `qopts=N`: the positive answer reports the same rows as
`undecided` rather than guessing about them.

**`identity` no longer says its answer is complete when it is not.**

`/dt/{db}/identity` answers *do you already hold this compound*. If a candidate
row could not be decided, its key was never computed and it could never appear in
`matches` — and the response still said `"complete": true`. It now says `false`,
which is the difference between *we looked and it is not here* and *we could not
finish looking*.

**And the tautomer count in `explain` no longer claims to be complete when a rule
was abandoned.**

`explain.tautomerFormsComplete` was set `false` only when the form limit was
reached. A rule whose search ran out of budget, or hit the per-rule match cap,
left the field saying `true` over an enumeration that had been cut off. A field
named *complete* is the worst place for that.

**What did not change.**

- No format bump, no migration, no rebuild.
- Hit sets, counts and rendering are all unmoved for every search that did not
  involve an undecided row — which, measured, is essentially all of them: 30
  rows in ten million cannot decide *themselves*, the easiest query there is.

## 0.14.1 — 2026-08-28

✅ **No index needs rebuilding.** The format is unchanged at version 3. An index
built by any earlier version opens, and this release does not change a single
byte of one — that was measured on 21 923 molecules, file by file, not argued.

⚠ **If you hold SDF files from 0.14.0 or earlier, export them again.** This is
the third correction in two releases to what a reader takes out of an SDF record,
and the largest.

**An SDF record no longer claims unpaired electrons an atom does not have.**

This build expected a negatively charged phosphorus, sulfur or iodine to reach a
*higher* valence than it actually does, and wrote the shortfall down as a
radical. The consequence: **every iodide counter-ion this project has ever
exported carried a triplet radical — 1 189 records in 100 000.** A reader is told
those atoms have unpaired electrons. They do not.

Measured atom for atom against RDKit over 100 000 PubChem molecules: records
making a claim no atom supports go from **1 189 to 0**, agreement from 98 239 to
**99 386**, and nothing that was right became wrong.

⚠ **Nothing could see this until something looked for it.** The check this
project runs on SDF output compares the molecule a reader gets back, and RDKit
renders an iodide with unpaired electrons to exactly the same string as one
without. The defect was invisible to every measurement taken here until a
per-atom comparison was written.

**And the last three structures that came back as a different molecule are
fixed.** They were the phosphide anion `PH2-`, the same expectation seen from the
other side. **Structures that come back as a different molecule are now 0 in
100 000**, down from 15 before 0.14.0.

**What still does not work.**

- **`F[Kr]F` cannot be read back by RDKit.** We write it correctly; its valence
  model declines krypton difluoride.
- **Radical state on a metal is not modelled here at all.** 601 records in
  100 000 state fewer unpaired electrons than RDKit does, every one of them on an
  atom outside the organic subset. That is a gap, not a wrong answer.
- Everything the 0.14.0 notes list as outstanding is still outstanding.

## 0.14.0 — 2026-08-28

✅ **No index needs rebuilding.** The format is unchanged at version 3, and
nothing about how a molecule is *stored* or *searched* moved in this release.

⚠ **If you have SDF files exported by an earlier version, export them again.**
Two things an SDF record used to say were not in your input, and both are fixed.
The coordinates are the same; what changed is what a reader takes from them.

**An SDF record no longer claims a double-bond geometry you did not state.**

A molfile has no way to shrug in its coordinates: a reader works out cis or trans
from where the atoms sit, so the drawing answered a question nobody asked. Over
100 000 PubChem molecules, **9 751 records came back through RDKit carrying a
descriptor their input never wrote** — nearly one in ten. The format does have a
way to say "either" — the crossed double bond — and it is now written wherever
the structure leaves the geometry open, matching RDKit bond for bond. That count
is **0**, and agreement on molecules with stereochemistry goes from 93.3% to
**99.8%**.

⚠ This mattered more than its size because `fmt=sdf` is what this README
recommends for round-tripping results through another toolkit, *precisely
because* `fmt=smiles` has a portability problem. The escape hatch was the leakier
of the two.

**An SDF record now states the valence of an atom a reader cannot guess.**

Fifteen molecules in 100 000 came back as a *different molecule* because the
total valence was never written down, so the reader applied its own default —
and the default was wrong in both directions. `C[SnH3]` came back with one
hydrogen instead of three; `[Te][Te]` came back with one hydrogen each where it
had none. Tin, thallium, tellurium, silicon, manganese, arsenic and iridium.
**Twelve of the fifteen now round-trip.**

Carbon, nitrogen and oxygen were never affected — a reader reconstructs those
correctly, which is why ordinary organic chemistry has always been fine here and
why this took thirteen releases to surface.

**What still does not work, and one correction.**

⚠ **The README used to say `fmt=sdf` "does not have this problem".** It had two.
That line is corrected rather than quietly rewritten.

- **Three molecules in 100 000 still come back different**, all of them the
  phosphide anion `[PH2-]`. The cause is not the writer: this build expects four
  hydrogens on a negatively charged phosphorus where RDKit expects two, so the
  two it has are reported as unpaired electrons. Fixing that changes how every
  charged atom is *read*, not how it is written, so it is tracked separately.
- **`F[Kr]F` cannot be read back by RDKit.** We write it correctly; its valence
  model declines krypton difluoride.
- Everything the 0.13.0 notes list as outstanding is still outstanding, including
  the two fullerene cages whose identity key moves with the spelling.

## 0.13.0 — 2026-08-28

✅ **No index needs rebuilding.** The format is unchanged at version 3.

⚠⚠ **Every identity key changes value.** Nothing stores one — identity is
computed from the index you already have, which is why no rebuild is needed — but
**if you cached keys of your own, recompute them.** A cached key from 0.12.0 will
not match a fresh one, and nothing will tell you.

**A query drawn from a molecule finds it, whichever way either was written**

Three fixes to the same underlying thing: whether the answer depends on how the
SMILES was *typed* rather than on what the molecule *is*.

⚠ **The identity key was reading a coin-toss.** When a ring is aromatic the key
ignores which way the alternating single/double assignment landed, because that
is a choice with no chemistry in it. The test for "is this a free choice" was too
narrow — it asked whether the *bond* was flagged aromatic, when the question is
whether both its *atoms* are. One molecule in eighty-three spellings changed key
on nothing but where a double bond happened to fall.

⚠ **A stereocentre's parity was fitted to how a corpus writes molecules.** The
rule that decides when a three-coordinate sulfur, phosphorus or nitrogen reads as
its own mirror image was derived twice, both times on molecules as their corpus
wrote them — and re-spelling the same molecules broke thirteen sulfur centres
that had been correct. The rule now asks the *molecule* (does this centre carry a
double bond?) instead of the *string* (is there an `=` after the ring digit?),
which turns out to be the same question with a spelling-proof answer.

Together: **the same compound spelled two ways gave two identity keys 17 times in
100 000, and now does so 2 times in 40 000** — see the corrected limitation below.

**Structures we write are more portable**

A ring-fusion bond inside a fused system was marked aromatic when the circuit
that made the system aromatic does not run through it. Both RDKit and Arthor mark
that bond single, and the disagreement made our written SMILES read back as a
different molecule. Over a 497 877-molecule sample, structures another toolkit
misreads go from **22 to 10**.

### Two corrections to what 0.11.0 and 0.12.0 published

⚠ **"The identity key is not invariant … 17 times in 100 000."** Both halves were
wrong. The number came from a check that drew **six** spellings per molecule and
reported a *sampled lower bound* — a figure that cannot be compared between
releases at all, because which molecules surface depends on the draw. Measured
properly, at 24 fixed spellings over 40 000 molecules, it is **2**.

⚠ **"22 of 497 877 written structures are read as a different molecule."** Now
**10**. Twelve were the ring-fusion bond above.

### What is left, and why one half of it is a decision rather than a bug

The two molecules whose key still moves are fullerene cages. On a fused cage the
ring *count* is stable but the *choice* of rings is not — SSSR is famously
non-unique — and this build deliberately takes the cyclomatic number rather than
RDKit's symmetrised SSSR. ⚠ **RDKit symmetrises precisely to make that choice
canonical.** Changing it would change what a ring means everywhere, including
under the aromatic divergences the README lists, so it is a decision to take
rather than a defect to fix.

Unchanged and still declared: **40 in 100 000** molecules are refused in one
spelling and accepted in another; **455 of 497 693** rows cannot be decided
inside the matcher's budget; **one molecule in 78 935** leaves `fmt=sdf` as a
different isomer.

### Inside, where nothing should be visible

Two files were split along the lines of what changes them — rendering an answer
is a different job from deciding it, and counting π electrons is a different job
from perceiving a ring. ⚠ **Two further splits were considered and declined**,
because the sections of an MDL record are that format's own structure and
extracting them would scatter the reasoning that keeps it checkable. Behaviour is
unchanged, and this release's verification was run against that claim rather than
around it.

## 0.12.0 — 2026-08-27

✅ **No index needs rebuilding.** The format is unchanged at version 3.

Three endpoints the reference has and this did not. All three are **off by
default**, each behind its own switch, and everything this server did before is
unchanged.

**Warming an index before you need it** — `--enable-index-control`

The first search against a cold index pays for reading it off disk, which is an
hour at 124 M. `PUT /dt/{db}/data?idxtouch=SUB` (or `SIM`) asks for that in
advance and answers with the database's description, `memInfo` included, so you
can see how much arrived.

⚠ **Warming is advice, not a command.** The kernel may honour it now, later or
never. Measured: on a 2.9 M index the substructure slices went from 0% resident
to 100% in three seconds; on Linux a 64 MB mapping came back at 320 pages of
16 384 within a second, because the readahead is started and not waited for.

⚠ **And it is one of the few things here by which a careless client hurts its
neighbours rather than itself** — warming a 124 M index pulls 10.5 GB into the
page cache and can push everything else on the machine out of it. That is why it
is a switch.

**`idxevict` works on Linux and cannot work on macOS** — and says so

Dropping a file's pages needs `posix_fadvise`, which macOS does not have. ⚠ The
call that looks like it should work returns success and does nothing:

```text
                                   macOS                    Linux
madvise(MADV_DONTNEED)   0, 100% still resident   0, 16384 still resident
posix_fadvise(DONTNEED)  does not exist           0, 16384 -> 0 resident
```

So on macOS the endpoint refuses and explains, rather than answering 200 over a
page cache it did not touch.

**Sending a collection over HTTP** — `--enable-upload`, with `--build-dir`

```sh
curl -X POST --data-binary @catalogue.smi "http://host:8080/dt/upload?name=vendor"
```

⚠ **`"built": false` is in the response and is the part to read.** The file is
stored where a hand-copied one goes; it does not build here, because a 124 M
build is hours and a request that runs for hours looks like a hung server. It
becomes searchable at the next reload, exactly as a copied file does.

⚠ **A name already in use is refused, never replaced.** Uploads are bounded —
2 GB by default, `--max-upload-bytes` to change it, and an oversized body is
refused before it is read into memory. The name is checked against a strict
allow-list because it becomes a filename.

⚠ **There is still no authentication in front of any of this**, which is why
uploading is off unless you switch it on, and why the two settings are refused
as a pair: `--enable-upload` without `--build-dir` will not start.

**A default order for databases** — `--enable-index-control`

`POST /dt/setpriority` takes a JSON array of names and sets the order
`GET /dt/data` lists them in. Names it does not have are ignored rather than
refused, so one stale entry does not fail the request. ⚠ The order is not saved
across a restart — unlike a database's *name*, which stands in every URL, an
ordering that reverts costs a client nothing it cannot see.

### Refused with a reason rather than implemented

`?idxmigrate=`, `?name=` and `?resolver=` on `PUT /dt/{db}/data` answer `400`
and say why. What migrating an index means is undocumented, and a name or
resolver changed at runtime would not survive a restart — a rename that reverts
is worse than no rename.

### Unchanged from 0.11.0

The limitations that release declared all still stand: the identity key is not
yet invariant to how a molecule was written (17 in 100 000), 22 of 497 877
written structures are read by other tools as a different molecule and 12 are
not read at all, and 455 of 497 693 rows cannot be decided inside the matcher's
budget.

## 0.11.0 — 2026-08-27

✅ **No index needs rebuilding.** The format is unchanged at version 3.

⚠⚠ **Three of these change answers you have already been given.** If you have
searched for a deuterated compound, a metal complex, or a molecule with a
sulfoxide in a ring, the result you got was wrong and this release corrects it.

**A query naming an element inside brackets now means that element**

Inside a SMARTS bracket, element symbols and query primitives share one syntax,
and this build resolved every collision toward the primitive. `[2H]` compiled to
*"isotope 2, carrying one hydrogen"* — **with the element dropped entirely** — so
a deuterium, which carries no hydrogens, failed a query drawn from itself, while
the same pattern matched unrelated atoms such as `[NH+]`. Wrong in two
directions from one cause.

Seventeen two-letter symbols went the same way. `[Rh]` was read as *"in a ring,
with one implicit hydrogen"*; `[Rb]` as *"in a ring, and an aromatic boron"*, a
contradiction that matches nothing. ⚠ Eight were misread silently and nine were
refused outright as a `400` — and **the silent eight were the dangerous half**,
because a refusal is visible and a contradiction is not.

Measured over a 497 335-molecule PubChem sample: **3 144 molecules could not be
found by a query written from themselves, and now can.** A further 358 molecules
could not be turned into a query at all and now can.

⚠ `[CH3]` still means *a carbon with three hydrogens*, and `[H1]`, `[!H]`,
`[H;R]` and `[H,C]` still count hydrogens. The rule was read out of the
reference's grammar rather than invented: `H` is the element only when the whole
bracket is *isotope, symbol, charge, label* and nothing else, which is why
`[H+]` is a proton and `[H&+]` is "one hydrogen and a positive charge".

**A sulfoxide in a ring is no longer returned as its mirror image**

A stereocentre written with its ring-closure digit followed by a bond — the shape
a cyclic sulfoxide takes, `[S@]1=O` — had its configuration inverted. ⚠ In the
whole of PubChem the pattern occurs **3 671 times and every one is a sulfoxide**;
all 3 671 were mirrored and now agree with the reference oracle. Round-tripping a
molecule through this build no longer changes 14 stereocentres in 497 877.

**A search now tells you when it could not decide about a row**

Finding whether a query occurs in a molecule is NP-complete, so the matcher gives
up on a molecule after a fixed amount of work rather than hanging your build on
one row. ⚠ **Until now those rows were reported as "your query does not occur
here."** They are now counted apart, in a new `undecided` field, and the warning
says the count is a floor.

For calibration: matching every molecule against **itself** — the easiest query
there is — **455 of 497 693** PubChem molecules exceed the budget. All of them
are symmetric giants: dendrimers, polyfluorinated repeats, cage boranes. ⚠
Raising the budget is deliberately not offered as a fix, because it trades a
silent wrong answer for a slow one.

⚠ The field is omitted when it is zero, so a client written against the
reference API sees exactly the response shape it saw before.

**Concurrent similarity searches now share one pass over the data**

Similarity scores every molecule in the collection, and until now every
concurrent query did that separately. Queries arriving while a pass is running
join the next one, so the server's work per query stops growing with load:
**at ten concurrent queries it was 142 ms each and is now about 60**.

⚠ A single-user server pays nothing for this — one query is 21.5 ms against
21.1 before. ⚠ And a query never joins a pass **already in progress**: it would
receive a score distribution covering part of the collection, which is a wrong
answer rather than a partial one.

### Still true, and now measured more precisely

- ⚠ **The identity key is not yet invariant to how a molecule was written.** The
  same compound spelled two ways gives two keys **17 times in 100 000**, so
  `/dt/{db}/identity` can answer *"we do not hold this"* about a compound the
  collection holds; **40 more in 100 000** are refused in one spelling and
  accepted in another. The three reference corpora read 100.0000% over 179 304
  strings, so it takes awkward chemistry to show. Search is unaffected — no hit
  set depends on the key.
- ⚠ **22 of 497 877 structures we hand back as SMILES are read by other tools as
  a different molecule, and 12 are not read at all.** All are fused ring systems
  where this build marks a ring-fusion bond aromatic and both references mark it
  single. The molecule stored and searched here is correct; the written form is
  not portable. `fmt=sdf` does not have this problem.
- **One molecule in 78 935 still leaves `fmt=sdf` as a different isomer**, and a
  hydrogen on a charged sulfur is still dropped from an SD file. Unchanged.

### A correction to what 0.8.0 and 0.9.0 published

The README told you that if you had concluded a deuterated compound could not be
searched here, *"it could, and it always could."* ⚠ **That was too strong.** The
model always kept the deuterium — nothing was lost reading the SMILES, which is
what that correction was about and it was true — but the **query** side dropped
the element, so searching for one did not work. It works from this release.

## 0.10.0 — 2026-08-26

✅ **No index needs rebuilding.** The format is unchanged at version 3.

⚠⚠ **If you run 0.9.0 or earlier, this fixes something that can break a
database you already have.** Build a sharded database, then run the same command
again — which is what anyone does after a failure, or by habit — and every shard
is refused, correctly, **and the database's manifest is rewritten to say it has
no shards.** The shard directories are all still there; the database simply
stops opening, and the only sign is `every shard is missing or unreadable`.

Reproduced on the released 0.9.0 binary. The manifest is now read from the disk:
it describes the database, not the build that just failed. **If a database of
yours has stopped opening this way, the data is intact** — re-run the build with
`--resume` and it will repair the manifest without rebuilding anything.

**An interrupted build can be continued**

`--resume` skips a shard already sealed at the destination. A 124 M build is
hours; before this there was no way to continue one, and killing it meant
starting over. A resumed database is byte-for-byte the database a full build
would have made.

⚠ **Only correct if the input file has not changed.** A shard records how many
molecules it holds, not which bytes of the input it came from, so nothing can
tell a shard built from your current file from one built before you edited it.
Resuming over a different file would build a database mixing two collections and
looking entirely valid. That is why the flag exists rather than the behaviour.

⚠ And `--shard` together with `--shards` is now **refused** instead of quietly
building everything. The check tested the flag's value rather than whether you
gave it, so `--shards 16 --shard 0` built all sixteen.

**Substructure search stopped losing molecules**

Four separate defects, found by one question with no oracle in it: *does a
molecule match itself as a substructure query?* It always should. Across three
corpora, **3 221 molecules of 22 423 did not — now none do.**

- A bare two-letter element was read as a different element: `Sc1ccccc1` parsed
  as scandium plus five carbons, six atoms where seven were written. Any query
  holding `S` before an aromatic carbon matched the wrong thing or nothing.
  Phenothiazines are the family that found it.
- A chiral descriptor was stored in one convention by the SMILES reader and
  another by the SMARTS reader, so the two disagreed whenever anything preceded
  the centre — which is nearly always.
- Chirality was compared before the query's neighbours had been matched to the
  target's, which cannot mean anything: a descriptor is a parity over an order.
- A `/` or `\` was read as a demand for a single bond, so a mark next to an
  aromatic ring asked for connectivity the molecule never had.

⚠ **A query that draws a stereocentre now matches both enantiomers again**, and
that is deliberate. Both Arthor and `RDKit` do the same by default — asked on a
3 004-molecule collection, Arthor answers 212 for a chiral query and 212 for its
mirror, `RDKit` 207 and 207 — and this build had been the odd one out for its
whole life, first invertedly and then correctly. **Correct is not the same as
right**: a search that halves your hit set because you drew a wedge is answering
a question you did not ask. Use `qopts=G` to say you meant it.

**Three more `qopts` letters, and two of them were always yours**

- **`S`** and **`N`** are Arthor's, and this build was answering `400` to both.
  A client using either got an answer from the reference and a refusal here.
- **`S`** locks stereochemistry *out*: every atom the query did not already
  speak about must not be a stereocentre. The opposite of what the letter
  suggests.
- **`N`** negates: it returns the molecules that do **not** match. Verified an
  exact complement — on a 2.9 M collection, 689 064 matching and 2 208 735 not,
  summing to the whole.
- **`G`** is ours. It enforces the configurations your query draws.

⚠ Two things `N` refuses rather than fudges. Its count is a **ceiling** where an
ordinary count is a floor, so the rows are withheld until it settles — a
half-finished negation would name rows that are about to match. And a search a
time limit cut short **cannot be negated at all**, because its complement would
contain matching rows permanently.

**Still true, and still limitations**

- A hit whose structure will not fit V2000 is left out of the SD file rather
  than truncated; over 999 atoms or bonds needs V3000, which is not implemented.
- One molecule in 78 935 with stereochemistry comes back from `fmt=sdf` as a
  different isomer, and 164 state less than they could.
- A combination product files under whichever of its drugs is larger, so
  `identity` will not find it under the smaller one.
- Sharding is within one machine, not across machines.
- 211 molecules in 2 897 799 of ChEMBL read a different aromatic system from
  RDKit, **202 of them cases where RDKit's own answer is not a function of the
  molecule**.
- No authentication and no per-client rate limiting. Bind to loopback or put
  this behind something that has them.

## 0.9.0 — 2026-08-24

✅ **No index needs rebuilding.** The format is unchanged at version 3. After a
release that forced every operator to rebuild, that is the first thing worth
saying.

⚠ **A correction to what 0.8.0 told you, and it was about search rather than
export.** The last release said that an isotope written as an explicit hydrogen
and a radical electron are *"lost when the SMILES is read, so they affect search
as much as export"*. **That was wrong.** `[2H]C([2H])([2H])Oc1ccccc1` and
`[O]N1C(C)(C)CC(C)(C)N1[O]` both survive a SMILES round trip untouched; both
defects lived in the molfile writer, and the isotope half was in fact already
fixed by the `M  ISO` line that shipped *in that release*. If you read it and
concluded a deuterated compound could not be searched here — **it could, and it
always could.**

**`fmt=sdf` no longer returns a different molecule than it was given**

Three property lines and one atom-line field closed the last of it. `M  RAD`
carries radical electrons, so a nitroxide stops coming back reduced; V2000's
*valence* field carries a hydrogen the model cannot derive, so a sulfoximine
stops coming back one hydrogen short. Measured on a 100 000-molecule ChEMBL
sample, the class where the SD file is a *different graph* than the input went
**133 → 23 → 8 → 0**.

⚠ The control matters more than the fix: on that sample **exactly one record of
100 000 differs** between the build before the valence change and the build
after, byte for byte. A count of fixes cannot tell a surgical change from a
lucky one.

**Duplicates are answered, never removed**

`GET|POST /dt/{database}/identity?query=...` returns the rows holding the same
compound as your query — the free acid, its sodium salt, its lysine salt — each
saying **how** it merged and what it set aside.

⚠ **The one endpoint here that is ours rather than Arthor's.** Their API has no
notion of identity, so nothing existing changed and a client written against
them never calls it. It needs no stored key and no format change: a salt
contains its own charge parent, so a substructure search on the parent finds the
candidates and the key is computed on those alone.

⚠ **It answers and never acts**, and that is a decision rather than an
unfinished half. A key merges 4.12% of ChEMBL and **99.88% of those merges go
through the "largest fragment is the compound" rule**, which cannot tell a
counter-ion from a second active ingredient. Shown to a chemist that population
is informative; acted on, it silently deletes distinct substances.

**Two things to know before relying on it.** A combination product files under
whichever of its drugs is larger — ask about the larger one and the row appears
with the other named, ask about the smaller and **the row is not there at all**.
And its per-row work is a standardisation rather than a scan step, so ask it
about compounds: benzene against 2.9 M rows rejects 2 440 588 candidates on size
and still runs to its deadline, returning `"complete": false`.

**A long build stops looking like a hung one**

The pass that reads your file and writes the shard now reports like the other
three. It was silent, and it is the *first* thing a build does — on a large
rebuild three shards spent about six and a half hours inside it while the log
showed only the previous shards' tails, which reads exactly like a hang.

⚠ It reports in **bytes**, because it is the pass that discovers how many
molecules there are and so has no molecule count to report against. Every line
now names its `unit`.

**Under the hood, where you should see nothing change**

- Parsers are held to a bound they were not held to before: a molecule or a
  query pattern may never hold more atoms or bonds than its string holds bytes.
  That is the "or an OOM" half of "a bad query is a 400, not a crash or an OOM",
  and nothing had been testing it.
- **12 600 single-edit mutations of this project's real query corpus** run on
  every commit — 4 266 of them parse and 826 run all the way to a match. Random
  characters reach shallow: of 4 096 such strings only 198 parse at all.
- Three corpus golden files now pin what the molfile writer draws and what ring
  and aromaticity perception sees. The second guards the headline correctness
  figure, which nothing in CI had been guarding.

**A defect this release found and did not fix**

⚠ **A substructure query that specifies a stereocentre returns the other
enantiomer.** Ask for the R form of a chiral compound and you get the S row, and
the other way round. Chirality is enforced — exactly one of the two matches —
but inverted.

It was found while verifying this release, by driving the new `identity`
endpoint against a chiral molecule and getting no answer where there plainly
was one. **It is not new**: it reproduces on the 0.8.0 binary you may already be
running, and it is declared here rather than quietly carried for a second
release. Only queries that *write* a configuration are affected; an unspecified
query matches both forms, which is correct.

⚠ `identity` does not inherit it. Its candidate search carries no chirality on
purpose — that search is a prefilter and the key comparison is the predicate,
and **a prefilter stricter than its predicate drops answers**. Fixing that was
the difference between an endpoint that works for chiral drugs and one that
answers "not held" for nearly all of them.

**Still true, and still limitations**

- A hit whose structure will not fit V2000 is left out of the SD file rather
  than truncated; over 999 atoms or bonds needs V3000, which is not implemented.
- One molecule in 78 935 with stereochemistry comes back as a different isomer —
  a fused 29-membered macrolactam the layout refuses by construction — and 164
  state less than they could.
- Sharding is within one machine, not across machines.
- 211 molecules in 2 897 799 of ChEMBL read a different aromatic system from
  RDKit, **202 of them cases where RDKit's own answer is not a function of the
  molecule**.
- No authentication and no per-client rate limiting. Bind to loopback or put
  this behind something that has them.

## 0.8.0 — 2026-08-22

⚠ **Every index must be rebuilt.** The packed molecule changed shape and the
index format moved from version 2 to 3. A shard of the older version is refused
**by name**, loudly, rather than read with the wrong rules — which is what the
version field is for, and why an old shard is not quietly misread into molecules
that are garbage wearing a valid shape.

**Results leave the building as SD files**

`fmt=sdf` returns MDL V2000 records with generated 2D coordinates — the format
every drawing program and registration system opens. It returned a `400` for the
life of this project, because a molfile is a *drawing*: nothing in the file says
cis or trans, a reader works it out from where the atoms sit, and a wrong drawing
is a wrong molecule with nothing marking it as wrong.

It is served now because the drawing was measured to a conclusion rather than
argued about. Against `RDKit` over five corpora — 78 935 molecules carrying
stereochemistry — **78 770 agree, 164 state less than they could, and one states
something false.** That one is a 29-membered macrolactam whose ring shares 27
atoms with its neighbour; the layout refuses a fused macrocycle by construction.

⚠ Two things it does not carry: an isotope written as an explicit hydrogen
(`[2H]` becomes an ordinary H, so a deuterated compound is indistinguishable from
its parent) and radical electrons (a nitroxide comes back reduced). Roughly 20
and 1 records per 100 000 respectively.

⚠⚠ **Corrected after release.** This paragraph originally said both were *"lost
when the SMILES is read, so they affect search as much as export"*. **That was
wrong.** Both molecules survive a SMILES round trip untouched — the defects were
in the molfile writer, which had no line for a mass number and none for an
unpaired electron. The isotope half was in fact already fixed by the `M  ISO`
line **in this release**; the radical half is fixed in the next one. Search was
never affected by either. The sentence is left standing above with its
correction attached rather than quietly edited away.

**An empty answer says how many tautomeric forms your query has**

A substructure search is exact. Draw the enol, search a collection that stores
the keto form, and the answer is zero with no reason attached. The `explain`
object now carries `tautomerForms`.

⚠ **A fact about your query and never about the collection.** "This query has
four forms" is arithmetic on the string you sent; "another of its forms is in
here" would be a statement about the molecules, and this does not make it. The
count includes the form you drew, so `1` means there is no other and rules the
question out. `tautomerFormsComplete: false` means enumeration hit its bound and
the count is a floor.

It runs the reference's own 37 transforms and agrees with it on **94.5%** of a
`ChEMBL` sample, compared set against set. About 20 ms on an answer that already
found nothing, and `--no-tautomer-report` switches it off entirely.

**271 molecules were reported as their mirror image**

A stereocentre with three neighbours and exactly one ring-closure digit came back
as the other enantiomer — a wrong molecule in a highlight and a wrong canonical
key beside it. 0.0094% of `ChEMBL`, and not the kind of small that is also
harmless. **271 fixed, 0 broken.**

**Smaller indexes and faster similarity**

The packed molecule is **8.1% smaller** and a scan **5.2% faster**: an atom with
no unusual property costs one byte instead of two, and a bond that continues the
chain costs its flags byte and nothing else. Across a whole index that is about
2.6%.

Similarity is **about 20% faster** for a separate reason: the scan was walking
each fingerprint's words three times, computing the intersection and then both
popcounts, where the query's count is the same for the whole pass and the
target's was already stored in the index.

**What is still behind**

⚠ Similarity throughput remains the gap, and it now has a shape rather than a
number. On this build, running without the fingerprint codec is **5.4x** faster
than with it — that is a *setting*, and it costs a larger index. A further ~2x
sits in the fingerprint width, and that one stays where it is: 512 bits was
chosen on recall, and recall outranks speed here.

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
