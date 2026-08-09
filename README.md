# moleculo

Chemical structure search over large molecule collections — substructure, SMARTS,
similarity and formula search, served over an HTTP API that is wire-compatible
with Arthor.

Distributed as a single static binary. No runtime, no dependencies, no
installer. **Bring your own molecules**: nothing is bundled, and the software
sends nothing anywhere.

## Install

Download the archive for your platform from
[Releases](https://github.com/denyzhirkov/moleculo/releases), verify it, and put
the binary on your `PATH`.

```sh
tar xzf moleculo-<version>-<platform>.tar.gz
cd moleculo-<version>-<platform>
shasum -a 256 -c ../SHA256SUMS      # optional, from the same release
./moleculo
```

| platform | archive |
|---|---|
| macOS, Apple Silicon | `aarch64-apple-darwin` |
| Linux x86-64 | `x86_64-unknown-linux-musl` |
| Linux arm64 | `aarch64-unknown-linux-musl` |

The Linux builds are statically linked against musl, so they have no glibc
version requirement and run on any distribution with a matching CPU.

## Use

Two steps: build an index from your molecules, then serve it. Indexes are
derived artifacts — reproducible from the input and the builder version, and
safe to delete and rebuild.

**1. Prepare input.** A tab-separated file, one molecule per line, SMILES first
and an identifier second:

```
CC(=O)Oc1ccccc1C(=O)O	aspirin
CN1CCC[C@H]1c1cccnc1	nicotine
```

**2. Build an index.** One directory per database; the directory name becomes
the database name in the API.

```sh
moleculo index build molecules.smi ./databases/mydb
```

Building is out-of-core: memory stays bounded regardless of input size, so a
machine with less RAM than the dataset can still build it. Useful options:

| option | meaning |
|---|---|
| `--bits N` | similarity fingerprint width (256/512/1024/2048, default 512). Narrower costs recall, wider costs disk. Fixed at build time. |
| `--codec none\|zstd` | compress the molecule and record stores. Default `zstd`. |
| `--fp-codec`, `--screen-codec` | as above, for the fingerprint column and screening index. |

Inspect what you built:

```sh
moleculo index inspect ./databases/mydb
```

**3. Serve.**

```sh
moleculo serve ./databases --port 8080
```

Every subdirectory that holds a valid index is served as a database. An index
that is missing, corrupt or of an unknown format version makes only *that*
database unavailable — the rest keep serving.

## Query

```sh
curl 'http://localhost:8080/dt/mydb/search?type=Substructure&query=c1ccccc1&length=10'
```

`type` is `Substructure`, `SMARTS` or `Similarity`. `fmt` is `json` (default),
`tsv` or `csv`.

**A search may need more than one request.** Large scans return partial results
with `"hasMore": true`; repeat the identical request until it comes back false,
and the counts have converged. Reading `recordsFiltered` from the first response
gives you a partial count.

`qopts` narrows a substructure query: `R` locks ring systems, `C` locks chains,
`Q` locks charge, `I` locks isotope, in any combination. For `Similarity` it
names the measure instead — `Tanimoto`, `Dice`, `Dissimilarity`, `Manhattan` or
`Tversky(α,β)`.

Other endpoints: `/config`, `/dt/data` for the database list, `/dt/{db}/data`
for one database.

## Known limitations

- **`type=Formula` is accepted but returns no results.** The parameter exists
  for API compatibility; the search behind it is not implemented in this
  version.
- **`fmt=sdf` is refused with a `400`** naming what is missing: it needs 2D
  coordinate generation.
- Molecules are not canonicalised, so there is no cross-database identity and no
  duplicate detection.
- **Above roughly 12 M molecules an index does not yet get a complete similarity
  index.** The build still succeeds and substructure and SMARTS work over every
  row; `type=Similarity` on such a database is refused with a `400`. Being
  fixed.

## Notes

- Query structures are treated as confidential: they are never logged in full
  and never leave the process.
- Every search carries a wall-clock and memory bound. An unbounded query is
  refused rather than queued forever.
- A malformed SMILES or SMARTS is a `400`, never a crash.

## Licence

Proprietary. See `LICENSE`. Source code is not distributed.

Third-party open source components and their licences are listed in
`THIRD-PARTY-LICENSES`.
