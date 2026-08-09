# Changelog

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
