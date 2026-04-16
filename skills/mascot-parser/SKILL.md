---
name: mascot-parser
description: Mascot Parser (msparser) Skill
---

# Mascot Parser (msparser) Skill

Always-on reference for writing scripts that interact with Mascot Server using the msparser SDK. Covers Python, Perl, Java, and C#.

**Language-specific references:**
- **Python**: This file (below) — constructor syntax: `msparser.ClassName(args)`
- **Perl**: [references/perl-api.md](references/perl-api.md) — constructor syntax: `msparser::ClassName->new($args)`, constants: `$msparser::ClassName::CONSTANT`
- **Java**: `import matrix_science.msparser.*;` — constructor syntax: `new ClassName(args)`
- **C#**: `using matrix_science.msparser;` — constructor syntax: `new ClassName(args)`
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
params.getDB()       # database name
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
3. **1-based indexing** - queries, peptide ranks, peak indices all start at 1
4. **Use `PROXY_TYPE_NO_PROXY`** for localhost connections
5. **Use the two-argument constructor** for `ms_peptidesummary` / `ms_proteinsummary` — the multi-argument constructor is obsolete
6. **Protein iteration** - `getHit(n)` returns `None` when no more hits; start at 1
7. **Peptide duplicate check** - always check `getPeptideDuplicate(i) != DUPE_DuplicateSameQuery` before processing
8. **Error handling pattern**: check `isValid()` first, then `getLastErrorString()`, then `clearAllErrors()`
9. **ms_datfile can read from URL** - pass `ms_connection_settings` with session ID as third argument
10. **Result files** are `.dat` (text) or `.msr` (binary) format; both opened the same way
11. **FDR target is a decimal** - `setTargetFDR(0.01)` for 1%, not `setTargetFDR(1)`
12. **Significance test** - `getIonsScore() >= getPeptideThreshold(q, 20, rank)` is equivalent to `getExpectationValue() <= 0.05`

## Example Scripts

The msparser SDK ships with example scripts in `<MSPARSER_SDK>/example_python/`, `example_perl/`, `example_java/`, and `example_csharp/`. Key examples: `resfile_summary`, `http_client`, `config_mascotdat`, `resfile_info`, `resfile_params`.

**Note:** Some example scripts still use the obsolete multi-argument `ms_peptidesummary` constructor. See [references/obsolete-examples.md](references/obsolete-examples.md) for details. Always use the two-argument constructor with `ms_mascotresults_params` in new code.

## Detailed References

- [Perl API Reference](references/perl-api.md) - Perl-specific method signatures, constructors, constants, and complete working patterns
- [API Class Reference](references/api-classes.md) - full method signatures for all key classes including `ms_mascotresults_params`
- [Common Recipes](references/common-recipes.md) - copy-paste code patterns for common tasks
- [Server Configuration](references/server-config.md) - directory layout, authentication flow
- [Obsolete Examples](references/obsolete-examples.md) - which SDK example scripts need modernizing
