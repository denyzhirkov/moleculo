# Changelog

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
