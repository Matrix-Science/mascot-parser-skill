---
name: mascot-parser
description: Mascot Parser (msparser) Skill
---

# Mascot Parser (msparser) Skill

Always-on reference for writing scripts that interact with Mascot Server using the msparser SDK. Covers Python, Perl, Java, and C#.

**Language-specific references:**
- **Python**: This file (below) — constructor syntax: `msparser.ClassName(args)`
- **Perl**: [references/perl-api.md](references/perl-api.md) — constructor syntax: `msparser::ClassName->new($args)`, constants: `$msparser::ClassName::CONSTANT`
- **Java**: [references/java-api.md](references/java-api.md) — `import matrix_science.msparser.*;`, constructor syntax: `new ClassName(args)`
- **C#**: [references/csharp-api.md](references/csharp-api.md) — `using matrix_science.msparser;`, constructor syntax: `new ClassName(args)`, **note: `_params()` not `params()`**
- The API method names are identical across all bindings. Only constructors, constant access, and object syntax differ.

## IMPORTANT: Safety Rules

- **NEVER submit searches, modify data, or perform any write operations against `https://www.matrixscience.com/`.**
  That is Matrix Science's public server. You may only fetch help/documentation pages from it (read-only).
- All search submissions, result downloads, and server operations must target the user's local or designated Mascot Server only.
- Never hardcode credentials in scripts. Load from environment variables or `.env` files.

## Obtaining msparser

msparser is licensed software from Matrix Science. Download it from:
**https://www.matrixscience.com/msparser_download.html**

After downloading, extract the SDK. The bindings are organized by language:
- Python: `<MSPARSER_SDK>/python36_and_later/`
- Perl: `<MSPARSER_SDK>/perl5XX/` (version-matched)
- Java: `<MSPARSER_SDK>/java/msparser.jar`
- C#: `<MSPARSER_SDK>/csharp_vs2019/`

## Platform Paths

Mascot Server runs on Windows or Linux. Paths differ by platform:

| Resource | Windows (typical) | Linux (typical) |
|----------|-------------------|-----------------|
| Mascot root | `C:\inetpub\mascot\` | `/usr/local/mascot/` |
| Config | `<MASCOT_HOME>\config\mascot.dat` | `<MASCOT_HOME>/config/mascot.dat` |
| Result files | `<MASCOT_HOME>\data\` | `<MASCOT_HOME>/data/` |
| Sequence DBs | `<MASCOT_HOME>\sequence\` | `<MASCOT_HOME>/sequence/` |
| CGI URL | `http://<HOST>/mascot/cgi/` | `http://<HOST>/mascot/cgi/` |

Throughout this skill, `<MASCOT_HOME>` means the Mascot Server installation root and `<MSPARSER_SDK>` means the extracted msparser SDK directory.

## Setup (Python)

```python
import sys
sys.path.insert(0, "<MSPARSER_SDK>/python36_and_later")
import msparser
```

**Credentials** should be loaded from environment or `.env`:

```python
from dotenv import load_dotenv
import os
load_dotenv()

MASCOT_URL  = os.getenv("MASCOT_URL",  "http://localhost/mascot/cgi/")
MASCOT_USER = os.getenv("MASCOT_USER", "")
MASCOT_PASS = os.getenv("MASCOT_PASS", "")
```

## Core API Quick Reference

### Opening Result Files

All languages use the same factory method:

| Language | Code |
|----------|------|
| Python | `resfile = msparser.ms_mascotresfilebase.createResfile(path)` |
| Perl | `my $resfile = msparser::ms_mascotresfilebase::createResfile($path)` |
| Java | `ms_mascotresfilebase resfile = ms_mascotresfilebase.createResfile(path);` |
| C# | `ms_mascotresfilebase resfile = ms_mascotresfilebase.createResfile(path, 0, "");` |

**NOTE:** lowercase 'f' in `createResfile` — not `createResFile`.

Always check validity:
```python
if not resfile.isValid():
    print(resfile.getLastErrorString())
```

### Search Parameters

```python
params = resfile.params()  # returns ms_searchparams
params.getDB()       # PRIMARY database name (see multi-DB note below)
params.getCOM()      # search title
params.getTOL()      # peptide tolerance
params.getTOLU()     # tolerance units
params.getITOL()     # fragment ion tolerance
params.getITOLU()    # fragment ion tolerance units
params.getPFA()      # missed cleavages
params.getMODS()     # fixed modifications
params.getIT_MODS()  # variable modifications
params.getCLE()      # enzyme
params.getSEARCH()   # search type (MIS, PMF, SQ)
params.getMASS()     # Monoisotopic or Average
params.getCHARGE()   # charge state
params.getINSTRUMENT()  # instrument type
```

#### Multi-database searches: `getDB()` is not enough

Mascot supports searching multiple databases in a single submission (`DB`, `DB2`, `DB3`, ... in the search form, common when Distiller users tick multiple databases in the search dialog). **`params.getDB()` returns only the primary (first) database — silently hiding the rest.** Reporting "the database is X" based on `getDB()` alone is a confident-sounding bug when the actual search used more than one.

Always enumerate all databases. The indexed `getDB(i)` is **1-based** (consistent with queries, peptide ranks, fixed/variable mod indices, MS1 match iteration, etc.; only component indexing is 0-based):

```python
n = params.getNumberOfDatabases()
dbs = [params.getDB(i) for i in range(1, n + 1)]   # 1-based
```

| Call | Behaviour |
|------|-----------|
| `params.getDB()` | Primary DB only — equivalent to `params.getDB(1)` |
| `params.getDB(0)` | Returns `''` (no DB at index 0) |
| `params.getDB(i)` for `1 <= i <= getNumberOfDatabases()` | Database name at slot `i` |
| `params.getNumberOfDatabases()` | Total count — always check this before reporting "the database" |

Any summary tool should print all databases when `getNumberOfDatabases() > 1`. Distiller-side note: the `<SearchOptions>` blocks inside a `.rov`'s `rover_data` stream carry empty `Database` attributes — those are GUI preference templates, not the executed search. The authoritative DB list is on the embedded `.dat`'s `params` object.

### Creating Results: The Two-Argument Constructor (Recommended)

**Use `ms_mascotresults_params` with the two-argument constructor.** The old multi-argument constructor is obsolete.

#### Step 1: Get default parameters

| Language | Code |
|----------|------|
| Python | `resParams = msparser.ms_mascotresults_params()` |
| Perl | `my $resParams = msparser::ms_mascotresults_params->new()` |
| Java | `ms_mascotresults_params resParams = new ms_mascotresults_params();` |
| C# | `ms_mascotresults_params resParams = new ms_mascotresults_params();` |

Then populate from mascot.dat defaults:

```python
# Python
datfile = msparser.ms_datfile("<MASCOT_HOME>/config/mascot.dat")
mascotOptions = datfile.getMascotOptions() if datfile.isValid() else msparser.ms_mascotoptions()
scriptName = resfile.get_ms_mascotresults_params(mascotOptions, resParams)
```

```perl
# Perl
my $datfile = msparser::ms_datfile->new("<MASCOT_HOME>/config/mascot.dat");
my $mascotOptions = $datfile->isValid()
    ? $datfile->getMascotOptions()
    : msparser::ms_mascotoptions->new();
$resfile->get_ms_mascotresults_params($mascotOptions, $resParams);
```

#### Step 2: Override settings as needed

```python
resParams.setTargetFDR(0.01)  # 1% FDR
resParams.setTargetFDRType(msparser.ms_mascotresults.DS_COUNT_PSM)
resParams.setMinNumSigUniqueSequences(2)
```

#### Step 3: Create the summary

| Language | Code |
|----------|------|
| Python | `results = msparser.ms_peptidesummary(resfile, resParams)` |
| Perl | `my $results = msparser::ms_peptidesummary->new($resfile, $resParams)` |
| Java | `ms_peptidesummary results = new ms_peptidesummary(resfile, resParams);` |
| C# | `ms_peptidesummary results = new ms_peptidesummary(resfile, resParams);` |

#### Complete Python example

```python
resfile = msparser.ms_mascotresfilebase.createResfile(path)
datfile = msparser.ms_datfile("<MASCOT_HOME>/config/mascot.dat")
mascotOptions = datfile.getMascotOptions() if datfile.isValid() else msparser.ms_mascotoptions()

resParams = msparser.ms_mascotresults_params()
resfile.get_ms_mascotresults_params(mascotOptions, resParams)

# Optional: set target FDR
resParams.setTargetFDR(0.01)

if resParams.isUsePeptideSummary():
    results = msparser.ms_peptidesummary(resfile, resParams)
else:
    results = msparser.ms_proteinsummary(resfile, resParams)
```

### Protein Summary (PMF searches)

For PMF searches, use `ms_proteinsummary` with the same two-argument pattern:

```python
results = msparser.ms_proteinsummary(resfile, resParams)
```

## Scores and Significance

### Determining if a peptide match is significant

The correct way to test significance is:

```python
pep.getIonsScore() >= results.getPeptideThreshold(query, 20, rank)
```

This is equivalent to:

```python
pep.getExpectationValue() <= 0.05
```

Both tests answer the same question: "Is this match statistically significant at p < 0.05?"

- `getIonsScore()` returns the Mascot ions score for the PSM
- `getPeptideThreshold(query, 20)` returns the score threshold at p=0.05 (20 = 1/0.05)
- `getExpectationValue()` returns the expectation value (E-value) directly

**Use `getPeptideThreshold()` for threshold-based filtering.** Use `getExpectationValue()` when you need the actual E-value for reporting or ranking.

### Threshold methods on ms_peptidesummary

```python
results.getPeptideIdentityThreshold(query, 20)      # Identity threshold at p=0.05
results.getHomologyThreshold(query, 20)              # Homology threshold at p=0.05
results.getPeptideThreshold(query, 20, rank)         # Effective threshold for this PSM
results.getProbOfPepBeingRandomMatch(score, query)   # Probability of random match
```

The `20` argument is the significance level expressed as `1/p`, so `20` = p=0.05, `1000` = p=0.001.

### Multi-language significance test

| Language | Significance test |
|----------|-------------------|
| Python | `pep.getIonsScore() >= results.getPeptideThreshold(q, 20, rank)` |
| Perl | `$pep->getIonsScore() >= $results->getPeptideThreshold($q, 20, $rank)` |
| Java | `pep.getIonsScore() >= results.getPeptideThreshold(q, 20, rank)` |
| C# | `pep.getIonsScore() >= results.getPeptideThreshold(q, 20, rank)` |

## FDR Control

### Setting a target FDR

When the search includes a decoy database (`params.getDECOY() > 0`), you can set a target FDR on `ms_mascotresults_params` before creating the summary. This **overrides** the `minProbability` setting.

```python
resParams.setTargetFDR(0.01)  # 1% FDR (range 0.0 - 1.0)
resParams.setTargetFDRType(msparser.ms_mascotresults.DS_COUNT_PSM)  # PSM-level FDR
```

**FDR types:**
- `DS_COUNT_PSM` — FDR calculated at PSM (peptide-spectrum match) level
- `DS_COUNT_SEQUENCE` — FDR calculated at unique peptide sequence level

```perl
# Perl
$resParams->setTargetFDR(0.01);
$resParams->setTargetFDRType($msparser::ms_mascotresults::DS_COUNT_PSM);
```

```java
// Java
resParams.setTargetFDR(0.01);
resParams.setTargetFDRType(ms_mascotresults.DS_COUNT_PSM);
```

**Note:** `setTargetFDR()` expects a decimal (0.01 for 1%), not a percentage. If accepting user input as a percentage, divide by 100.

### Requiring multiple significant unique sequences per protein

```python
resParams.setMinNumSigUniqueSequences(2)  # Range 1-5, default 1
```

## Percolator Integration

Percolator rescoring uses machine learning to improve peptide identification. When enabled, Parser uses Percolator PEP (posterior error probability) values instead of Mascot E-values.

### Check if Percolator can be used

Percolator requires:
- A decoy search (`params.getDECOY() > 0`)
- MS/MS data (`resfile.anyMSMS()`)
- Not an error tolerant search
- Sufficient queries and sequences (server-configurable minimums)

### Enable Percolator

Set the `MSPEPSUM_PERCOLATOR` flag in `flags2`:

```python
flags2 = resParams.getFlags2()
flags2 |= msparser.ms_peptidesummary.MSPEPSUM_PERCOLATOR
resParams.setFlags2(flags2)
```

```perl
# Perl
my $flags2 = $resParams->getFlags2();
$flags2 |= $msparser::ms_peptidesummary::MSPEPSUM_PERCOLATOR;
$resParams->setFlags2($flags2);
```

### Check if Percolator is enabled by default

```python
def percolator_enabled(resParams):
    return bool(resParams.getFlags2() & msparser.ms_peptidesummary.MSPEPSUM_PERCOLATOR)
```

## Iterating Protein Hits

```python
hit = 1
prot = results.getHit(hit)
while prot:
    acc = prot.getAccession()
    score = prot.getScore()
    desc = results.getProteinDescription(acc)
    mass = results.getProteinMass(acc)
    coverage = prot.getCoverage()
    hit += 1
    prot = results.getHit(hit)
```

**⚠️ This loop sees only family representatives — see the next section if you
need every identified protein.**

## Reading all proteins: family members, not just `getHit()`

`ms_peptidesummary.getHit(n)` (and `ms_proteinsummary.getHit(n)`) returns **only
the protein-family REPRESENTATIVE** of each family. When result clustering is on
(it is, by default, in the canonical reader — `setProteinFamilySwitch(1)` /
`MSRES_CLUSTER_PROTEINS`), homologous proteins that group into a family are
stored as family **MEMBERS** and are **invisible to a `getHit()`-only loop**. A
reps-only extractor silently drops them — they were identified, they pass FDR,
and they never appear in your output.

**Verified failure (Mascot Server 3.1.241.27):** in a published *Vibrio
cholerae* 2D-TPP study, the three validated chemoreceptor hits — UniProt
**Q9KSE4** (VC1313), **Q9KQ43** (VC2161), **Q9KS54** (VC1406) — cluster into a
single 38-member family whose representative is **Q9KVD0** (VC2161 is family 251,
member 7). A `getHit()`-only loop dropped ~189 member proteins across 5 samples,
including all three biological hits, even though every one passed Percolator at
q = 0. On a routine bacterial TMT search, walking members recovered 66 extra
proteins that `getHit` alone missed (2690 vs 2624) — exactly
`getNumberOfFamilyMembers()`.

### Walking family members

There are two APIs. **`getNextFamilyProtein(masterHit, id)` is the simpler
per-family iterator** and is what most code should use:

```python
def iter_all_proteins(results):
    """Yield (protein, representative_accession_or_None) for EVERY protein."""
    n = results.getNumberOfHits()
    for hit in range(1, n + 1):                 # 1-based
        rep = results.getHit(hit)
        if rep is None:
            break
        yield rep, None                         # the representative
        rep_acc = rep.getAccession()
        member_id = 1
        while True:
            member = results.getNextFamilyProtein(hit, member_id)
            if member is None:                  # no more members in this family
                break
            yield member, rep_acc               # a homologous family member
            member_id += 1
```

`getNextFamilyProtein` returns `None` at `member_id = 1` when a family has no
extra members, so this degrades cleanly to a plain `getHit` walk on
non-clustered results. Family members expose the same `ms_protein` interface as
representatives (`getAccession`, `getScore`, `getNumPeptides`,
`getPeptideQuery`/`getPeptideP`, quant lookups all work on them).

The lower-level alternative is **`getHitAndFamilyMember(prot, hitAndFamily,
rules)`**, where the 3rd argument is a `ms_mascotresults::hitAndFamily_t` cursor
(NOT a simple int) and `rules` combines `FC_PROTEIN_IGN_*` flags
(`FC_PROTEIN_IGN_SUBSETS`, `FC_PROTEIN_IGN_SAMESETS`, `FC_PROTEIN_IGN_FAMILY`,
`FC_PROTEIN_IGN_MASK`). Use it when you need fine control over which
subset/sameset/family relationships to traverse. Note
`getNumberOfFamilyMembers()` is a **GLOBAL** total across the whole result, not a
per-family count — don't use it to bound a per-family loop; iterate
`getNextFamilyProtein` until it returns `None` instead.

### Gotcha summary

| Call | What it returns |
|------|-----------------|
| `getHit(n)` | family REPRESENTATIVE only — members are hidden |
| `getNextFamilyProtein(hit, id)` | the `id`-th member of family `hit`; `None` past the last |
| `getNumberOfFamilyMembers()` | GLOBAL member total (all families) — not a per-family bound |
| `getNumberOfHits()` | number of representatives (= number of families) |

## Importing Percolator results (not in the `.msr`)

**Percolator / ML-rescored results are NOT stored in the SQLite `.msr`.** They
live in server cache files written when Percolator ran:

```
<MASCOT_HOME>/data/cache/<YYYY>/<MM>/<hash>/<res>.<hash>.target.pop   # FDR-passing target PSMs
                                            <res>.<hash>.decoy.pop    # decoy PSMs
                                            <res>.<hash>.pip          # Percolator input
```

To get Percolator scores through msparser you must **import** them — the
canonical reader (`master_results_2.pl` →
`<MASCOT_HOME>/perl64/site/lib/PeptideSummary/Util.pm::make_ms_mascotresults_params`)
builds the summary with clustering and Percolator on by default:
`setProteinFamilySwitch(1)`, flags `MSRES_CLUSTER_PROTEINS | MSRES_SHOW_SUBSETS`,
Percolator enabled, `MSPEPSUM_USE_CACHE`, `setIgnoreIonsScoreBelow(...)`,
`setUsePeptideSummary(1)`. msparser computes its own cache hash and often looks
under a *different* path than the server wrote, so on real installs you must
stage the server's actual `.pop`/`.pip` files where msparser expects them (glob
the server cache, prime `setPercolatorFeatures`, copy to
`getPercolatorFileNames()`, then set `MSPEPSUM_PERCOLATOR`) — see
[Common Recipes](references/common-recipes.md) recipe 20.

### The `.target.pop` is directly usable and authoritative

For FDR-passing PSMs you often don't need msparser's summary at all — the
`.target.pop` is a tab-separated table you can read directly. Columns:

| Column | Meaning |
|--------|---------|
| `PSMId` | `query:N;rank:M` — **N is the msparser query number**, so the spectrum/peaks are reachable via `ms_inputquery(resfile, N)` |
| `score` | Percolator score |
| `q-value` | Percolator q-value — filter `q <= target FDR` |
| `posterior_error_prob` | PEP |
| `peptide` | `X.SEQUENCE.X` (flanking residues either side of dots) |
| `proteinIds` | space-separated accessions; a peptide mapping to exactly one accession is unique to it |

**Recommended robust pattern — drive per-protein identification/quant off the
`.target.pop` PSMs:** read the rows, keep `q <= target FDR`, group by
`proteinIds`. This captures representatives **AND** family members automatically
and bypasses the `getHit()` blind spot entirely. For quant, parse `query:N` out
of `PSMId`, read that spectrum with `ms_inputquery(resfile, N)`, sum the reporter
ions, and add the PSM's reporter sums to every protein in `proteinIds`. A
complete working TMT extractor built this way (PSMId parsing, cache glob,
`ms_inputquery` peak access) is in the project at
`TPP2D/scripts/msr_to_quant_summary_pop.py`.

### Getting Peptides for a Protein

```python
for i in range(1, 1 + prot.getNumPeptides()):
    query = prot.getPeptideQuery(i)
    rank = prot.getPeptideP(i)
    if query == -1 or rank == -1:
        continue
    if prot.getPeptideDuplicate(i) == msparser.ms_protein.DUPE_DuplicateSameQuery:
        continue
    pep = results.getPeptide(query, rank)
    if pep:
        seq = pep.getPeptideStr()
        score = pep.getIonsScore()
        expect = pep.getExpectationValue()
        mz = pep.getObserved()
        charge = pep.getCharge()
        delta = pep.getDelta()
        mods = pep.getVarModsStr()
        readable_mods = results.getReadableVarMods(query, rank)
        significant = score >= results.getPeptideThreshold(query, 20, rank)
```

### HTTP Client & Authentication

```python
settings = msparser.ms_connection_settings()
settings.setProxyServerType(msparser.ms_connection_settings.PROXY_TYPE_NO_PROXY)  # for localhost

client = msparser.ms_http_client(MASCOT_URL, settings)
session = msparser.ms_http_client_session()
rc = client.userLogin(MASCOT_USER, MASCOT_PASS, session)
# rc == ms_http_client.L_SUCCESS or ms_http_client.L_SECURITYDISABLED
```

### Configuration Access

```python
datfile = msparser.ms_datfile("<MASCOT_HOME>/config/mascot.dat")
dbs = datfile.getDatabases()
for i in range(dbs.getNumberOfDatabases()):
    db = dbs.getDatabase(i)
    print(db.getName(), "active" if db.isActive() else "inactive")
```

### Input Query / Spectrum Data

```python
inp = msparser.ms_inputquery(resfile, query_number)  # 1-based
title = inp.getStringTitle(True)
num_peaks = inp.getNumberOfPeaks(1)
for i in range(1, 1 + num_peaks):
    mz = inp.getPeakMass(1, i)
    intensity = inp.getPeakIntensity(1, i)
```

### Search Type Detection

```python
resfile.anyMSMS()         # MS/MS ion search (prefer anyXxx over isXxx)
resfile.anyPMF()          # peptide mass fingerprint
resfile.anySQ()           # sequence query
resfile.isErrorTolerant() # error tolerant search
```

### Key Flags

```python
msparser.ms_mascotresults.MSRES_GROUP_PROTEINS       # parsimony grouping
msparser.ms_mascotresults.MSRES_SHOW_SUBSETS          # show subset proteins
msparser.ms_mascotresults.MSRES_CLUSTER_PROTEINS      # hierarchical clustering
msparser.ms_mascotresults.MSRES_MUDPIT_PROTEIN_SCORE  # MudPIT scoring
msparser.ms_peptidesummary.MSPEPSUM_USE_HOMOLOGY_THRESH  # homology threshold
msparser.ms_peptidesummary.MSPEPSUM_PERCOLATOR        # Percolator rescoring
msparser.ms_mascotresults.SCORE                       # sort by score
```

### Unassigned Peptides

```python
results.createUnassignedList(msparser.ms_mascotresults.SCORE)
for u in range(1, 1 + results.getNumberOfUnassigned()):
    pep = results.getUnassigned(u)
    print(pep.getPeptideStr(), pep.getIonsScore())
```

## Cross-Language Syntax Reference

| Concept | Python | Perl | Java | C# |
|---------|--------|------|------|----|
| Constructor | `msparser.ms_inputquery(resfile, q)` | `msparser::ms_inputquery->new($resfile, $q)` | `new ms_inputquery(resfile, q)` | `new ms_inputquery(resfile, q)` |
| Static method | `msparser.ms_mascotresfilebase.createResfile(p)` | `msparser::ms_mascotresfilebase::createResfile($p)` | `ms_mascotresfilebase.createResfile(p)` | `ms_mascotresfilebase.createResfile(p, 0, "")` |
| Constant | `msparser.ms_protein.DUPE_DuplicateSameQuery` | `$msparser::ms_protein::DUPE_DuplicateSameQuery` | `ms_protein.DUPE_DuplicateSameQuery` | `ms_protein.DUPLICATE.DUPE_DuplicateSameQuery` |
| Flags | `msparser.ms_mascotresults.MSRES_GROUP_PROTEINS` | `$msparser::ms_mascotresults::MSRES_GROUP_PROTEINS` | `ms_mascotresults.MSRES_GROUP_PROTEINS` | `(uint)ms_mascotresults.FLAGS.MSRES_GROUP_PROTEINS` |
| None check | `if prot:` | `if (defined $prot)` | `if (prot != null)` | `if (prot != null)` |
| Boolean True | `True` | `1` | `true` | `true` |

## Critical Gotchas

1. **`createResfile` not `createResFile`** - the 'f' is lowercase
2. **Always check `isValid()`** after creating objects, then `getLastErrorString()` for details
3. **1-based indexing** - queries, peptide ranks, peak indices, fixed/variable mod indices, MS1 match iteration, AND `params.getDB(i)` for multi-DB enumeration all start at 1. (Component indexing in quantitation is the exception — that's 0-based.)
3a. **Multi-DB searches**: `params.getDB()` returns only the primary database. Always enumerate via `params.getNumberOfDatabases()` + `params.getDB(i)` for `i in 1..N`. See [Search Parameters → Multi-database searches](#search-parameters) above. Reporting a single DB name from `getDB()` alone is a known footgun.
4. **Use `PROXY_TYPE_NO_PROXY`** for localhost connections
5. **Use the two-argument constructor** for `ms_peptidesummary` / `ms_proteinsummary` — the multi-argument constructor is obsolete
6. **Protein iteration** - `getHit(n)` returns `None` when no more hits; start at 1. **But `getHit()` returns family REPRESENTATIVES only** — homologous family members are hidden. To read every identified protein, also walk `getNextFamilyProtein(hit, id)`. See [Reading all proteins: family members, not just `getHit()`](#reading-all-proteins-family-members-not-just-gethit).
7. **Peptide duplicate check** - always check `getPeptideDuplicate(i) != DUPE_DuplicateSameQuery` before processing
8. **Error handling pattern**: check `isValid()` first, then `getLastErrorString()`, then `clearAllErrors()`
9. **ms_datfile can read from URL** - pass `ms_connection_settings` with session ID as third argument
10. **Result files** are `.dat` (text) or `.msr` (binary) format; both opened the same way — **EXCEPT** the new Mascot 3.x SQLite-backed `.msr`: on **large DIA** results `ms_peptidesummary`/`ms_proteinsummary` can return 0 hits or hang, in which case read the SQLite tables directly ([SQLite-backed `.msr`](references/sqlite-msr.md)). **Do not mistake the more common "missing proteins" causes for a SQLite-format problem:** (a) family members hidden behind `getHit()` (see [Reading all proteins](#reading-all-proteins-family-members-not-just-gethit)); (b) Percolator/FDR-passing hits that live in cache `.pop` files, not the `.msr` at all (see [Importing Percolator results](#importing-percolator-results-not-in-the-msr)). Neither is fixed by switching to raw SQL.
11. **FDR target is a decimal** - `setTargetFDR(0.01)` for 1%, not `setTargetFDR(1)`
12. **Significance test** - `getIonsScore() >= getPeptideThreshold(q, 20, rank)` is equivalent to `getExpectationValue() <= 0.05`

## Example Scripts

The msparser SDK ships with example scripts in `<MSPARSER_SDK>/example_python/`, `example_perl/`, `example_java/`, and `example_csharp/`. Key examples: `resfile_summary`, `http_client`, `config_mascotdat`, `resfile_info`, `resfile_params`.

**Note:** Some example scripts still use the obsolete multi-argument `ms_peptidesummary` constructor. See [references/obsolete-examples.md](references/obsolete-examples.md) for details. Always use the two-argument constructor with `ms_mascotresults_params` in new code.

## Detailed References

- [Perl API Reference](references/perl-api.md) - Perl-specific method signatures, constructors, constants, and complete working patterns
- [Java API Reference](references/java-api.md) - Java-specific setup (`System.loadLibrary`), array out-parameters, and complete working patterns
- [C# API Reference](references/csharp-api.md) - C#-specific gotchas (`_params()`, enum wrappers, `GC.KeepAlive`, `out` parameters)
- [API Class Reference](references/api-classes.md) - full method signatures for all key classes including `ms_mascotresults_params`
- [Common Recipes](references/common-recipes.md) - copy-paste code patterns for common tasks (Python)
- [Server Configuration](references/server-config.md) - directory layout, authentication flow
- [Obsolete Examples](references/obsolete-examples.md) - which SDK example scripts need modernizing
- [SQLite-backed `.msr`](references/sqlite-msr.md) - Mascot 3.x writes `.msr` as a SQLite DB; `ms_peptidesummary` fails on large DIA results; full table schema + direct-SQL recipes
