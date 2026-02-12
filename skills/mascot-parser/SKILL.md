# Mascot Parser (msparser) Skill

Always-on reference for writing Python scripts that interact with Mascot Server using the msparser SDK.

## IMPORTANT: Safety Rules

- **NEVER submit searches, modify data, or perform any write operations against `https://www.matrixscience.com/`.**
  That is Matrix Science's public server. You may only fetch help/documentation pages from it (read-only).
- All search submissions, result downloads, and server operations must target **`http://localhost/mascot/cgi/`** (the local server) only.
- Credentials in `.env` are for the local server. Never send them to any remote host.

## Obtaining msparser

msparser is licensed software from Matrix Science. Download it from:
**https://www.matrixscience.com/msparser_download.html**

After downloading, extract the SDK so that the Python bindings are at:
`<project_root>/msparser/python36_and_later/`

## Setup

Every script must begin with this preamble. The `msparser/`, `Documentation/`, and `mascot server scripts/` directories are not included in this repository because they contain licensed material. Ensure these are present locally before running any scripts.

```python
import sys
sys.path.insert(0, r"C:\Users\richardj\Qsync\dev\Matrix Science\Mascot Parser Skill\msparser\python36_and_later")
import msparser
```

**Credentials** are in `.env` at the project root (`C:\Users\richardj\Qsync\dev\Matrix Science\Mascot Parser Skill\.env`). Load with:

```python
from dotenv import load_dotenv
import os
load_dotenv(r"C:\Users\richardj\Qsync\dev\Matrix Science\Mascot Parser Skill\.env")

MASCOT_URL  = os.getenv("MASCOT_URL",  "http://localhost/mascot/cgi/")
MASCOT_USER = os.getenv("MASCOT_USER", "")
MASCOT_PASS = os.getenv("MASCOT_PASS", "")
```

**Local Mascot Server:**
- URL: `http://localhost/mascot/cgi/`
- Data directory: `C:\inetpub\mascot\data`
- Config directory: `C:\inetpub\mascot\config`
- Sequence databases: `C:\inetpub\mascot\sequence`
- Result files: `.dat` and `.msr` files in `C:\inetpub\mascot\data\` (e.g. `F001316.dat`, `F981142.msr`)

## Core API Quick Reference

### Opening Result Files

```python
resfile = msparser.ms_mascotresfilebase.createResfile(path)  # NOTE: lowercase 'f' in createResfile
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

### Getting Default Summary Parameters

```python
datfile = msparser.ms_datfile(r"C:\inetpub\mascot\config\mascot.dat")
mascotOptions = datfile.getMascotOptions() if datfile.isValid() else msparser.ms_mascotoptions()

(scriptName, flags, minProbability, maxHitsToReport,
 ignoreIonsScoreBelow, minPepLenInPepSummary,
 usePeptideSummary, flags2) = resfile.get_ms_mascotresults_params(mascotOptions)
```

### Peptide Summary (MS/MS searches)

```python
results = msparser.ms_peptidesummary(
    resfile, flags, minProbability, maxHitsToReport,
    "",  # unigene file
    ignoreIonsScoreBelow, minPepLenInPepSummary,
    "",  # unigene file 2
    flags2
)
```

### Protein Summary (PMF searches)

```python
results = msparser.ms_proteinsummary(resfile, flags, minProbability, maxHitsToReport)
```

### Iterating Protein Hits

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
        mz = pep.getObserved()
        charge = pep.getCharge()
        delta = pep.getDelta()
        mods = pep.getVarModsStr()
        readable_mods = results.getReadableVarMods(query, rank)
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
datfile = msparser.ms_datfile(r"C:\inetpub\mascot\config\mascot.dat")
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
resfile.isMSMS()          # MS/MS ion search
resfile.isPMF()           # peptide mass fingerprint
resfile.isSQ()            # sequence query
resfile.isErrorTolerant() # error tolerant search
# Prefer anyMSMS(), anyPMF(), anySQ() for type checking
```

### Key Flags

```python
msparser.ms_mascotresults.MSRES_GROUP_PROTEINS       # parsimony grouping
msparser.ms_mascotresults.MSRES_SHOW_SUBSETS          # show subset proteins
msparser.ms_mascotresults.MSRES_CLUSTER_PROTEINS      # hierarchical clustering
msparser.ms_mascotresults.MSRES_MUDPIT_PROTEIN_SCORE  # MudPIT scoring
msparser.ms_peptidesummary.MSPEPSUM_USE_HOMOLOGY_THRESH  # homology threshold
msparser.ms_mascotresults.SCORE                       # sort by score
```

### Unassigned Peptides

```python
results.createUnassignedList(msparser.ms_mascotresults.SCORE)
for u in range(1, 1 + results.getNumberOfUnassigned()):
    pep = results.getUnassigned(u)
    print(pep.getPeptideStr(), pep.getIonsScore())
```

## Critical Gotchas

1. **`createResfile` not `createResFile`** - the 'f' is lowercase
2. **Always check `isValid()`** after creating objects, then `getLastErrorString()` for details
3. **1-based indexing** - queries, peptide ranks, peak indices all start at 1
4. **Use `PROXY_TYPE_NO_PROXY`** for localhost connections
5. **`get_ms_mascotresults_params()`** returns a tuple of default parameters based on search type and mascot.dat settings
6. **Protein iteration** - `getHit(n)` returns `None` when no more hits; start at 1
7. **Peptide duplicate check** - always check `getPeptideDuplicate(i) != DUPE_DuplicateSameQuery` before processing
8. **Error handling pattern**: check `isValid()` first, then `getLastErrorString()`, then `clearAllErrors()`
9. **ms_datfile can read from URL** - pass `ms_connection_settings` with session ID as third argument
10. **Result files** are `.dat` (text) or `.msr` (binary) format; both opened the same way

## Example Scripts

Full working examples are at:
`C:\Users\richardj\Qsync\dev\Matrix Science\Mascot Parser Skill\msparser\example_python\`

Key examples: `resfile_summary.py`, `http_client.py`, `config_mascotdat.py`, `resfile_info.py`, `resfile_params.py`, `create_mgf.py`

## Detailed References

- [API Class Reference](references/api-classes.md) - full method signatures for all key classes
- [Common Recipes](references/common-recipes.md) - copy-paste code patterns for common tasks
- [Server Configuration](references/server-config.md) - local paths, databases, authentication flow
