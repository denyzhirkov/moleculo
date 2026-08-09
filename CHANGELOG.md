# Changelog

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
