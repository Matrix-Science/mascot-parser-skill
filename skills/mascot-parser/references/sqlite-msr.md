# The SQLite-backed `.msr` result file (Mascot Server 3.x)

Mascot Server **3.1.241.x** (and later) writes its `.msr` result files as
**SQLite 3 databases** ("Mascot SQLite Results"), not the legacy flat-text
`.dat` format. The legacy `.dat` is still embedded inside, gzip-chunked, in a
`legacy__dat28_file` table — but on large DIA results that embedded copy can be
incomplete or expensive to materialise.

> ## ⚠️ Before you reach for direct SQL: it is usually NOT the SQLite format
> ## that is hiding your proteins
>
> "Direct SQL on the `.msr`" is the right escape hatch only for the specific
> **large-DIA `ms_peptidesummary` hang / 0-hits** failure described below. It is
> **not** the fix for the two far more common "missing proteins" causes, and
> the SQLite `.msr` **cannot** supply what those need:
>
> 1. **Protein-family members are invisible to a `getHit()`-only loop.**
>    `ms_peptidesummary.getHit(n)` returns only family *representatives*;
>    homologous proteins clustered into a family are stored as *members* and
>    are silently dropped by a reps-only walk. This is a logic gap, not a file
>    format problem — switching to raw SQL does not fix it by itself, and you'd
>    be reimplementing Mascot's clustering. **Walk family members instead** (see
>    SKILL.md → *Reading all proteins: family members, not just `getHit()`*).
>
> 2. **Percolator / ML-rescored results are NOT in the `.msr` SQLite at all.**
>    Percolator output lives in the server cache files
>    `<res>.<hash>.target.pop` / `.decoy.pop` / `.pip`. No table in the schema
>    below contains q-values or PEPs. If a protein "passes at q=0" but you can't
>    find it, it is almost certainly a Percolator-only, FDR-passing hit you
>    haven't imported — the `.msr` will never show it via SQL **or** msparser
>    until you stage those `.pop` files (see SKILL.md → *Importing Percolator
>    results (not in the .msr)*). For FDR-passing PSMs + per-protein quant, the
>    **`.target.pop`-driven approach** in that section is the robust path and it
>    sidesteps the `getHit()` family blind spot for free.
>
> Reach for the direct-SQL recipes here only after you've ruled out (1) and (2).

## The gotcha — `ms_peptidesummary` on a big SQLite `.msr`

On a narrow-window DIA result (hundreds of thousands of pseudo-spectra, multi-DB
search), `msparser.ms_peptidesummary(resfile, params)` and `ms_proteinsummary`:

- **return `getNumberOfHits() == 0`** (no protein hits) — or
- **hang for many minutes** (it decompresses the ~hundreds-of-MB `legacy__dat28_file`
  and rebuilds a summary over all queries × all databases) — or both, depending
  on the build and the `params`.

Meanwhile `resfile.isValid()`, `anyMSMS()`, `anyFastaMatches()`,
`hasQuantitation()` all return **True** — so the empty result *looks like* a
silent failure rather than an unsupported format. Direct SQL on the same file
shows the data is fully there (e.g. 700+ PSMs for a single biomarker protein in
one raw file). On a DIA result with Percolator/MS2Rescore FDR this overlaps with
the older `MSPEPSUM_PERCOLATOR`-makes-`getHit()`-return-`None` bug (see the main
skill's Percolator section) — clearing that flag doesn't fix the hang.

**Practical rule:** for the new SQLite `.msr` on a large DIA result *where the
summary API genuinely hangs or returns 0 hits*, **read the SQLite tables
directly with the `sqlite3` module** instead of going through msparser's summary
API. msparser is still the right tool for `resfile.params()` (search
parameters, including multi-DB enumeration), `anyMSMS()` / `isErrorTolerant()` /
`hasQuantitation()` predicates, and `ms_inputquery` spectrum access. It's
specifically the *protein/peptide summary* on big DIA SQLite results that you
should bypass.

**But first confirm this is actually your failure mode.** If `getNumberOfHits()`
returns a sensible non-zero count and the call returns promptly, your "missing
proteins" are almost certainly family members or Percolator-only hits, not a
SQLite-format problem — see the caveat box at the top of this file. Direct SQL
here gives you raw `psm__peptides` / `protein__data` rows but **no q-values, no
PEPs, and no family clustering** (those tables simply don't exist), so it will
not, on its own, recover Percolator-passing or family-member proteins.

Detecting the format: open the file with `sqlite3` and read `schema_version`:

```python
import sqlite3
con = sqlite3.connect(f"file:{path}?mode=ro", uri=True)
maj, minr, prod, ver = con.execute(
    "SELECT major_version, minor_version, product_name, product_version "
    "FROM schema_version").fetchone()
# e.g. (1, 0, 'Mascot Server', '3.1.241.27')
```

(If `sqlite3.connect` raises `DatabaseError: file is not a database`, you have a
legacy flat `.dat`/`.msr` — use msparser normally.)

## Table layout (schema_version 1.0)

| Table | Rows (typical DIA file) | Purpose / key columns |
|---|---|---|
| `schema_version` | 1 | `major_version`, `minor_version`, `product_name`, `product_version` |
| `search__header` | ~17 | key/value: search title (`COM`), user, etc. |
| `search__parameters` | ~170 | key/value: every Mascot search-form field (`TOL`, `ITOL`, `CLE`, `PFA`, `MODS`, `IT_MODS`, `DECOY`, `INSTRUMENT`, …) |
| `search__databases` | one row per DB | `db_id`, `db_name`, `fasta_file`, `release`, `db_type`, `sequences`, `residues`, `decoy_type`, `et_sequences` — **this is the authoritative multi-DB list** |
| `search__fixed_mods` | one per fixed mod | `mod_num`, `mod_name`, `delta`, `neutral_loss`, `residues` |
| `search__variable_mods` | one per var mod | `mod_num`, `mod_name`, `delta` |
| `search__variable_mod_nl` | one per var-mod NL | `mod_num`, `idx`, `type`, `value` |
| `search__symbol_masses` | ~33 | `symbol` → `mass` (residue + special symbol masses) |
| `search__taxonomy` | 0+ | taxonomy filter rows |
| `search__config_files` | one per config | enzyme/mods/etc config snapshots |
| `query__data` | one per query | `query_id`, `source_index`, `title` (URL-encoded MGF title), `charge`, `rt_in_seconds`, `scans`, `raw_file`, `mass_min`/`mass_max`, `num_vals`, **`ions1` / `ions1_charge`** (the fragment peak list as `"mz:intensity,…"` / `"z,z,…"`; `ions2`/`ions3` for multi-spectrum queries), `ion_mobility`, `it_mods` |
| `query__summary` | ~2× query count | one row per (query, psm_type): `query_id`, `psm_type`, `observed_mr`, `observed_m_z`, `charge`, **`intensity`** (precursor MS1 intensity Distiller assigned during peak detection), `qmatch` (homology-threshold proxy) |
| `psm__peptides` | millions | one row per PSM: `query_id`, `rank_id`, `sequence_idx`, **`psm_type`** (0 = MS/MS-fragment-matched; 1 = secondary precursor-only, max score ~50), `missed_cleavages`, `peptide_mr`, `delta`, **`ions_score`**, **`sequence`**, **`varmods_string`** (per-position digit string, `'0'`=none), `summed_mods_string`/`local_mods_string` (for site-localisation), `ions_matched`, `ion_series_found`, `peaks_used_from_ions{1,2,3}`, `drange_start_pos`/`drange_end_pos`, **`quant_component_name`** (Distiller quant component this PSM was assigned to, when the search came from a Distiller project) |
| `psm__proteins` | ~2× PSM count | PSM → protein map: `query_id`, `rank_id`, `sequence_idx`, `psm_type`, **`protein_id`**, `frame_number`, **`start_idx`** / **`end_idx`** (1-based residue range in the protein), `multiplicity`, **`residue_before`** / **`residue_after`** (flanking residues), `sl_*` (spectral-library provenance) |
| `protein__data` | 10k–200k | **`protein_id`**, **`is_decoy`** (0/1), `protein_seq_id`, **`db_id`** (→ `search__databases`), **`accession_str`**, `mass`, `title` (description), `taxonomy_id`, `has_top_scoring_peptide` |
| `psm__substitutions` | 0+ | error-tolerant substitutions: `query_id`, `rank_id`, `psm_type`, `site`, `ambiguous_residue`, `residue` |
| `psm__et_mods`, `psm__seq_tags`, `psm__sl_mods`, `psm__linked_sites`, `psm__monolinks` | 0+ | ET mods / sequence tags / spectral-library mods / crosslink sites |
| `pmf_hit__*` | 0 in MS/MS searches | PMF (peptide-mass-fingerprint) hit tables |
| `legacy__dat28_file` | tens–hundreds of chunks | `idx`, `data` (BLOB) — the classic flat `.dat` file, gzip-compressed and split into ordered chunks. Reassemble by `ORDER BY idx`, concatenate, `gzip.decompress`. Can be hundreds of MB on a DIA result. This is what `ms_mascotresfilebase.createResfile()` actually parses — and why the summary API chokes. |

## Working query: PSMs for one protein, one raw file

```python
import sqlite3
con = sqlite3.connect(f"file:{msr_path}?mode=ro", uri=True)

SQL = """
SELECT pp.query_id, pp.rank_id, pp.psm_type,
       pp.sequence, pp.peptide_mr, pp.delta, pp.ions_score, pp.varmods_string,
       qs.observed_m_z, qs.charge, qs.intensity, qs.qmatch,
       qd.rt_in_seconds, qd.scans, qd.raw_file, qd.title,
       pd.accession_str, pd.db_id, pd.is_decoy,
       px.start_idx, px.end_idx, px.residue_before, px.residue_after
FROM psm__peptides pp
JOIN psm__proteins px
       ON px.query_id = pp.query_id AND px.rank_id = pp.rank_id
      AND px.psm_type = pp.psm_type AND px.sequence_idx = pp.sequence_idx
JOIN protein__data pd  ON pd.protein_id = px.protein_id
JOIN query__summary qs ON qs.query_id = pp.query_id AND qs.psm_type = pp.psm_type
JOIN query__data    qd ON qd.query_id = pp.query_id
WHERE pd.accession_str LIKE ?            -- e.g. 'P02750%'
  AND pd.is_decoy = 0
  AND pp.psm_type = 0                    -- MS/MS-fragment-matched only
  AND pp.ions_score >= ?                 -- e.g. 20.0
ORDER BY pp.ions_score DESC
"""
for row in con.execute(SQL, ("P02750%", 20.0)):
    ...
```

Notes:
- **Always filter `psm_type = 0`** unless you specifically want the secondary precursor-only matches.
- **`is_decoy = 0`** on `protein__data` excludes decoy hits.
- There is no `expectation_value` column — score-vs-`qmatch` is the available significance proxy. If you need true E-values, decode the embedded legacy `.dat` (`legacy__dat28_file`) and feed it to msparser — but on a big DIA file that's the slow path you're trying to avoid.
- `rt_in_seconds`, `scans`, `charge` in `query__data` are stored as **strings** (occasionally comma-lists for multi-scan queries) — cast/split as needed.
- `query__data.title` is **URL-encoded** — `urllib.parse.unquote` it to get the original MGF spectrum title (carries the raw-file path and scan number).
- `varmods_string` is one digit per peptide residue position (plus N-/C-term slots); a digit `n` means variable mod `mod_num = n` (from `search__variable_mods`) is on that residue. All-zeros = unmodified.
- The `quant_component_name` column links a PSM to a Mascot Distiller quant component, but the **integrated XIC areas themselves are not in the `.msr`** — they live in the Distiller `.rov` project's binary segment streams (see the `mascot-distiller` skill's `ROV_FILE_FORMAT.md`). For a quick label-free proxy you can trapezoidally integrate `query__summary.intensity` vs `query__data.rt_in_seconds` over the matched PSMs of a peptide form — symmetric across forms, so ratios are robust, but it underestimates the true Distiller XIC.

## When msparser *does* still work on a SQLite `.msr`

- `resfile = ms_mascotresfilebase.createResfile(path)` — opens fine (`isValid()` True).
- `resfile.params()` — full search parameters, including `getNumberOfDatabases()` + `getDB(i)` for the multi-DB list. Use this, not the `search__databases` table, if you want the msparser-typed view.
- `resfile.anyMSMS()` / `anyPMF()` / `isErrorTolerant()` / `hasQuantitation()` — all reliable.
- `ms_inputquery(resfile, q)` — spectrum/peak access works (it can be slow to first-touch because of the legacy `.dat` materialisation).
- `ms_peptidesummary` / `ms_proteinsummary` on a **small** search (a handful of files, modest query count) — fine. It's specifically the large-DIA case that breaks.
