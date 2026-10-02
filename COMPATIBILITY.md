# What 1.x promises

moleculo follows [semantic versioning](https://semver.org) from 1.0.0 onwards.
This file lists what that covers. If something is not listed here, it is not
promised, and you should not rely on it staying the same.

⚠ **Two terms are used precisely below.** *Stable* means that a 1.x release
will not remove it or change what it means. A release may still add to it. A
*breaking change* is any change that could make a client, a script or an index
that worked on 1.a stop working, or start meaning something else, on 1.b. Only
2.0 may make one.

---

## Stable across 1.x

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
  `moleculo --version` prints.
- **Additions are not breaking.** A new field in a JSON body, a new route, a new
  optional parameter and a new header can arrive in any minor release. Write
  clients that ignore fields they do not know.
- ⚠ **Where we answer differently from Arthor, the difference is stable too.**
  Each one is listed in `README.md` with the mechanism behind it, for example
  that the row column is a row id. A 1.x release will not quietly start
  behaving like the reference instead. If we ever do, the change is breaking and
  waits for 2.0.

### The command line

- Every `moleculo` subcommand and every flag documented in `README.md`, with its
  meaning and its environment variable.
- Exit status: `0` means success and non-zero means failure. The text of an
  error message is not stable. Script against the status, not the wording.
- ⚠ **A configuration that starts on 1.a starts on 1.b**, with one exception: a
  setting that was deprecated with a warning for at least one minor release.
  Unknown flags are refused rather than ignored, and that stays true, so removing
  a flag within 1.x would break a unit file that works today. That is why it
  does not happen.

### What is on disk

- **Index format version 3**, as [`INDEX-FORMAT.md`](INDEX-FORMAT.md) specifies
  it. It has been the format since 0.8.0 and is frozen for all of 1.x. Every
  index that 0.8.0 or later built opens in every 1.x release, with no rebuild
  and no migration step. The next format version comes with 2.0, and so does
  `index migrate`.
- **`database.json`, the shard manifest and `parts.json`.**
- ⚠ **A release that needs a rebuild for correctness says so** in its changelog
  entry, first line, even when the old index still opens. Opening the old index
  is guaranteed. Getting the new answers from it is not.

### Metrics

- The metric names and label names exported on `/metrics`. A metric may gain a
  label value, for example a new search type, in a minor release. Renaming or
  removing a metric is breaking.

---

## Not promised, deliberately

- **Timings, throughput, memory use and index size.** They are measured and
  published per release, and they move.
- **Log text.** Events and their fields change as the code does. Alert on
  `/metrics`, not on log lines.
- **Row ids across a rebuild.** A row id is valid for one index generation, and
  that is all [`INDEX-FORMAT.md`](INDEX-FORMAT.md) promises about it. Keep the
  `Identifier` instead.
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
- A patch release (1.x.**y**) never changes an answer.

### One thing that will not change in 1.x, even as a fix

**What counts as a ring.** Ring perception uses a minimal ring basis. On
molecules where two cycles of the same size tie, such as fullerenes, bridged
bicyclics and adamantanes, which ring the basis holds depends on how the
molecule was written. `README.md` § *Known limitations* measures what that costs.
A basis that does not depend on how the molecule was written exists, and it is
RDKit's symmetrised SSSR. Adopting it would change the ring count on 1.39% of
PubChem, and with it every `R` and `r` query. That is too large a shift for a
minor release, so the current definition is **frozen for all of 1.x**. The
stable definition is a candidate for 2.0.

---

## Deprecation

Anything stable is deprecated in a minor release before it is removed:

1. The changelog says so, names the replacement, and says which major version
   removes it.
2. Where the server or the CLI can tell that the deprecated thing was used, it
   logs a `warn` event **once per process**, not once per request.
3. The deprecated thing keeps working for the rest of 1.x.

`qopts=G` is not deprecated. It remains the spelling to write, for the reason
`README.md` gives. `geometry=1` exists for the day Arthor gives the letter `G`
a meaning of its own. If that day comes, `G` is deprecated through the steps
above and `geometry=1` carries the lock.

---

## Upgrading from 0.x to 1.0

- **Indexes**: anything built by 0.8.0 or later opens as it is. An index from
  0.7.x or earlier is format version 2 and is refused with a message that says
  so. Rebuild it.
- **Configuration**: nothing was removed. 1.0 adds the switches that the
  changelog lists.
- **The wire**: 0.24.0 changed the row column on databases with more than one
  shard. If you are coming from an earlier release, read that entry.
- **Parts**: upgrade the parts before the coordinator. A coordinator warns when
  a part runs a different build, and a part older than 0.24.0 sends no version
  at all.
