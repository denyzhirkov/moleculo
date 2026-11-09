# What 2.x promises

moleculo follows [semantic versioning](https://semver.org) from 1.0.0 onwards.
This file lists what that covers for 2.x. If something is not listed here, it is
not promised, and you should not rely on it staying the same.

⚠ **Two terms are used precisely below.** *Stable* means that a 2.x release
will not remove it or change what it means. A release may still add to it. A
*breaking change* is any change that could make a client, a script or an index
that worked on 2.a stop working, or start meaning something else, on 2.b. Only
3.0 may make one.

⚠ **2.0 made breaking changes against 1.x**, which is what a major version is
for. They are listed under [Upgrading from 1.x to 2.0](#upgrading-from-1x-to-20)
below. The first of them is that **2.0 does not serve a 1.x index until it is
converted** with `moleculo index migrate`.

---

## Stable across 2.x

### The HTTP interface

- **Every Arthor route this build answers**: `/config`,
  `/dt/data`, `GET|PUT /dt/{db}/data`, `GET|POST /dt/{db}/search`,
  `POST /dt/upload` and `POST /dt/setpriority`. Their parameters, response
  shapes, the error shape `{"reason","status","message"}`, the HTTP status
  codes, and the `hasMore` polling protocol are all stable. Two of the
  reasons are ours, because Arthor has no vocabulary for them: `Search
  Timeout` (503) and `Internal Error` (500, a request whose handler failed).
- **Our extensions**: `GET|POST /dt/{db}/identity`, `POST /admin/reload`,
  `/healthz`, `/readyz`, `/metrics`, `qopts=G`, `geometry=1`, and the
  `x-moleculo-version` response header, which is the same string
  `moleculo --version` prints. Since 2.0, an index entry in `/dt/data` may
  carry `perception`, the chemistry-rules version a shard was built with,
  present only when that is not the running build's; a current shard's entry
  is Arthor's shape exactly.
- **Additions are not breaking.** A new field in a JSON body, a new route, a new
  optional parameter and a new header can arrive in any minor release. Write
  clients that ignore fields they do not know.
- ⚠ **Where we answer differently from Arthor, the difference is stable too.**
  Each one is listed in `README.md` with the mechanism behind it, for example
  that the row column is a row id. A 2.x release will not quietly start
  behaving like the reference instead. If we ever do, the change is breaking and
  waits for 3.0.

### The command line

- Every `moleculo` subcommand and every flag documented in `README.md`, with its
  meaning and its environment variable.
- Exit status: `0` means success and non-zero means failure. The text of an
  error message is not stable. Script against the status, not the wording.
- ⚠ **A configuration that starts on 2.a starts on 2.b**, with one exception: a
  setting that was deprecated with a warning for at least one minor release.
  Unknown flags are refused rather than ignored, and that stays true, so removing
  a flag within 2.x would break a unit file that works today. That is why it
  does not happen.

### What is on disk

- **Index format version 4**, as [`INDEX-FORMAT.md`](INDEX-FORMAT.md) specifies
  it. It is the format of 2.0, frozen from 2.0.0-rc.1 on and for all of 2.x.
  Every index that 2.0.0 or a later 2.x release builds opens in every later
  2.x release, with no rebuild and no migration step. ⚠ **A release candidate
  promises less, and says what**: a shard rc.1 builds is format 4 and is
  expected to open in 2.0.0 unchanged, but should anything before 2.0.0 have to
  change the format, that is a new format version, 5, announced in the first
  line of its changelog entry and converted with `index migrate` — never a
  silent change to version 4. (Before rc.1, version 4 had never shipped, and
  its pre-release builds changed it without a new number; README § *Upgrading
  a 1.x index to 2.0* says what this build does with such a shard.)
- ⚠ **Version 3, the format of every 1.x release, is not read.** A 2.x release
  refuses a version-3 shard by name, and `moleculo index migrate` converts it.
  A version-2 index, built before 0.8.0, is refused as an unknown version and
  cannot be converted: rebuild it from its input.
- **`database.json`, the shard manifest and `parts.json`.**
- ⚠ **A release that needs a rebuild for correctness says so** in its changelog
  entry, first line, even when the old index still opens. Opening the old index
  is guaranteed. Getting the new answers from it is not. A 2.x shard records
  the chemistry rules it was built with, and one built under older rules is
  served as built and named in `/dt/data` (`perception`), so you can see which
  shards a rebuild would change. A coordinator's `/dt/data` does not carry the
  field: each part names its own such shards in its own `/dt/data`.

### Metrics

- The metric names and label names exported on `/metrics`. A metric may gain a
  label value, for example a new search type, in a minor release. Renaming or
  removing a metric is breaking.

---

## Not promised, deliberately

- **The oldest CPU a build runs on.** A minor release may raise it, and when it
  does, the changelog says so first and README § *Install* names the new
  floor. Since 1.1.0 the x86-64 build needs `x86-64-v2`.
- **Timings, throughput, memory use and index size.** They are measured and
  published per release, and they move.
- **Log text.** Events and their fields change as the code does. Alert on
  `/metrics`, not on log lines.
- **Row ids across a rebuild.** A row id is valid for one index generation, and
  that is all [`INDEX-FORMAT.md`](INDEX-FORMAT.md) promises about it. Keep the
  `Identifier` instead. (`index migrate` keeps every row id, because it does not
  change the generation.)
- **The order of hits with equal scores.**

### When a chemistry fix changes an answer

⚠ **This is the part of the promise that is weaker than the rest, and the
weakness is on purpose.** Correctness comes before stability. Hit sets are diffed
against RDKit, and wherever a hit set differs from what both RDKit and Arthor
answer, we treat it as a defect. Freezing defects for the life of a major
version would promise a wrong answer. So:

- A fix that changes hit sets, counts, similarity scores or identity keys **may
  ship in a minor release.**
- ⚠ **Every such release says so, under a heading of its own**. Its changelog
  entry has an **Answers moved** section. That section names each change, its
  mechanism, and how many molecules it moved on a named corpus. A release
  without that section changed no answer on any corpus we run. That claim is
  checked: 168 pinned hit sets are part of every gate.
- A patch release (2.x.**y**) never changes an answer.
- ⚠ **A fix that is in the stored chemistry reaches a shard only when the shard
  is rebuilt.** A 2.x shard stores the hydrogens, rings and aromaticity it
  perceived at build. Such a release says so first, as above, and `/dt/data`
  names every shard that still answers by the older rules (on a database of
  parts, each part's own `/dt/data` does).

---

## Deprecation

Anything stable is deprecated in a minor release before it is removed:

1. The changelog says so, names the replacement, and says which major version
   removes it.
2. Where the server or the CLI can tell that the deprecated thing was used, it
   logs a `warn` event **once per process**, not once per request.
3. The deprecated thing keeps working for the rest of 2.x.

`qopts=G` is not deprecated. It remains the spelling to write, for the reason
`README.md` gives. `geometry=1` exists for the day Arthor gives the letter `G`
a meaning of its own. If that day comes, `G` is deprecated through the steps
above and `geometry=1` carries the lock.

---

## Upgrading from 1.x to 2.0

Every item here is a change a 1.x client, script or index can notice. Read the
2.0 entry of the changelog for the measurements behind each.

- ⚠ **A 1.x index is not served until it is converted.** 1.x wrote index format
  3; 2.0 reads format 4. The server refuses each version-3 shard by name and
  says which command converts it. Run `moleculo index migrate <database>`: it
  converts in place, from the input lines the index already holds, and keeps
  every row id, record and deletion. README § *Upgrading a 1.x index to 2.0*
  has the details, including what it costs in disk and what happens to a line
  2.0 cannot store.
- **`index build --screen-codec` is gone**, with the screening index it
  compressed. Format 4 has none, and every substructure search scans. Unknown
  flags are refused, so a script that passes it stops with an error. Remove the
  flag.
- **`index build --codec` now compresses the record store only.** In 1.x it
  chose the codec of both the molecule store and the record store. In format 4
  the molecule store is always LZ4 HC, so `--codec none` leaves `smi.blk` raw
  and the molecule store compressed. A build that passed `--codec none` to get
  an uncompressed molecule store no longer gets one, and there is no flag that
  does.
- **`--max-seconds` bounds a search once, however many databases it names.**
  In 1.x a search over several databases (`/dt/db1,db2/search`) had the limit
  once per database, so three databases could run for three times it. A search
  rerun after it meets a damaged shard now spends the same deadline rather than
  a fresh one each time. A search over several databases can therefore time out
  where 1.x answered it.
- **A query past the new size and depth limits is a `400`.** A query of more
  than 4 096 atoms, a SMILES query of more than 512 aromatic atoms, a SMARTS
  expression nested more than 256 levels, more than two levels of `$()`, or
  more than 128 atoms inside `$()` is refused, with the limit named. 1.x
  answered such queries, after seconds to minutes of work, or overflowed a
  stack and stopped the whole server. README § *Searching* lists the limits.
- **`R<n>` and `r<n>` count relevant cycles**, not the rings of a minimal basis,
  so they no longer depend on how a molecule was written. In the 168 pinned hit
  sets, one answer moved: `[R2]` over the REAL Space sample lost three cubanes.
  The changelog's *Answers moved* says what else moves and on which molecules.
  Aromaticity, the identity key and the molfile's 2D layout still use the basis.
- **The first 1 000 rows of a similarity ranking are re-scored exactly**, from
  unfolded features, and re-ordered by that score. Their scores and order
  differ from 1.x wherever folding had decided them. The count, the histogram
  and everything below row 1 000 are the folded pass's, as before, and an
  answer that reaches past row 1 000 carries a `warning`.
- **`/dt/data` may carry a `perception` field** on the index entries of a shard
  built under older chemistry rules than the running build's. A shard built by
  the running build carries none, so its entry is unchanged, and a
  coordinator's entries for its parts never carry it.
- **A `database.json` that lists a shard id twice, or lists ids out of
  ascending order, is refused**, and that database is not served. 1.x opened a
  repeated id twice and served its rows twice. Every build, append and
  migration writes the list ascending, so only a hand-edited file is affected:
  fix the list.

What did not change: the routes, their parameters and response shapes, and the
metric names. Every flag and environment variable of 1.1.0 other than the three
above keeps its meaning. One describes different work without changing its
result: `--threads` still sets the workers of a build's parallel pass, and
format 4 has one such pass where format 3 had two; a shard still does not
depend on it.

### What 1.x promised, and where 2.0 breaks it

1.x promised the wire, the command line, the configuration, index format 3 and
the metric names for all of 1.x. It said that the next format version, and
`index migrate`, would come with 2.0. They did.

It also froze one thing even against a fix: **what counts as a ring.** Ring
perception used a minimal ring basis, and on molecules where two cycles of the
same size tie, such as fullerenes, bridged bicyclics and adamantanes, which ring
the basis holds depends on how the molecule was written. Adopting a definition
that does not depend on that would have moved every `R` and `r` query, which is
too large a shift for a minor release. 2.0 moves `R<n>` and `r<n>` off the
basis, as listed above. `README.md` § *Known limitations* measures what the
basis still costs aromaticity and the identity key.

---

## Upgrading from 0.x

- **Indexes**: anything built by 0.8.0 or later is format 3, and 2.0 converts
  it with `index migrate`. An index from 0.7.x or earlier is format version 2.
  2.0 refuses it as an unknown version and cannot convert it. Rebuild it from
  its input.
- **The wire**: 0.24.0 changed the row column on databases with more than one
  shard. If you are coming from an earlier release, read that entry.
- **Parts**: upgrade the parts before the coordinator. A coordinator warns when
  a part runs a different build, and a part older than 0.24.0 sends no version
  at all.
- Then read *Upgrading from 1.x to 2.0* above.
