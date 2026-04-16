# msparser API Class Reference

Detailed method signatures for core msparser classes. SDK version: MSPARSER_REL_3_1_0-2025-07-27.

## ms_mascotresfilebase

Factory/base class for opening Mascot result files.

```python
# Factory method - opens .dat or .msr files
resfile = msparser.ms_mascotresfilebase.createResfile(filename, keepAliveInterval=0)
```

**Validation & Errors:**
- `isValid()` -> bool
- `getLastError()` -> int (error code)
- `getLastErrorString()` -> str
- `getNumberOfErrors()` -> int
- `getErrorNumber(i)` -> int
- `getErrorString(i)` -> str
- `clearAllErrors()`

**File Information:**
- `getNumQueries()` -> int
- `getNumSeqs()` -> int (total sequences in database)
- `getNumSeqsAfterTax()` -> int (sequences after taxonomy filter)
- `getNumResidues()` -> float
- `getExecTime()` -> int (seconds)
- `getDate()` -> int (Unix timestamp)
- `getMascotVer()` -> str
- `getFastaVer()` -> str

**Search Type Detection:**
- `isPMF()` -> bool (peptide mass fingerprint)
- `isMSMS()` -> bool (MS/MS ion search)
- `isSQ()` -> bool (sequence query)
- `isErrorTolerant()` -> bool
- `anyPMF()` -> bool (prefer these over isPMF etc.)
- `anyMSMS()` -> bool
- `anySQ()` -> bool
- `anyFastaMatches()` -> bool

**Parameters & Results:**
- `params()` -> ms_searchparams
- `get_ms_mascotresults_params(mascotOptions, resParams)` -> str (scriptName). Populates the `resParams` object (ms_mascotresults_params) with defaults from the result file and mascot.dat options. **This is the recommended way to get default parameters.**
- `get_ms_mascotresults_params(mascotOptions)` -> tuple of (scriptName, flags, minProbability, maxHitsToReport, ignoreIonsScoreBelow, minPepLenInPepSummary, usePeptideSummary, flags2). **Obsolete form — use the two-argument version above.**

**Observed Data:**
- `getObservedMass(query)` -> float (1-based query number)
- `getObservedCharge(query)` -> int

---

## ms_mascotresults_params

Parameters object for creating `ms_peptidesummary` or `ms_proteinsummary`. This is the recommended way to pass parameters using the two-argument constructor.

### Constructors

```python
# Default constructor
resParams = msparser.ms_mascotresults_params()

# Constructor with explicit values
resParams = msparser.ms_mascotresults_params(
    flags,                    # uint (MSRES_* flags, default MSRES_GROUP_PROTEINS)
    minProbability,           # float (default 0.05)
    maxHitsToReport,          # int (default 0)
    unigeneIndexFile,         # str (default "")
    ignoreIonsScoreBelow,     # float (default 0.0)
    minPepLenInPepSummary,    # int (default 0)
    singleHit,                # str (default "")
    flags2                    # uint (MSPEPSUM_* flags, default MSPEPSUM_NONE)
)

# Copy constructor
resParams = msparser.ms_mascotresults_params(otherResParams)
```

### Populating from mascot.dat defaults

```python
datfile = msparser.ms_datfile("<MASCOT_HOME>/config/mascot.dat")
mascotOptions = datfile.getMascotOptions() if datfile.isValid() else msparser.ms_mascotoptions()
scriptName = resfile.get_ms_mascotresults_params(mascotOptions, resParams)
# resParams is now populated with appropriate defaults
```

### Getter Methods

- `getFlags()` -> uint — OR'd `ms_mascotresults.FLAGS` values
- `getFlags2()` -> uint — OR'd `ms_peptidesummary.MSPEPSUM` values
- `getMinProbability()` -> float — significance threshold (default 0.05)
- `getMaxHitsToReport()` -> int
- `getIgnoreIonsScoreBelow()` -> float — score cutoff (-1 means use minProbability)
- `getMinPepLenInPeptideSummary()` -> int
- `getMinNumSigUniqueSequences()` -> int — required significant unique sequences per protein (1-5)
- `getUnigeneIndexFile()` -> str
- `getSingleHit()` -> str
- `getTargetFDR()` -> float — target FDR (0-1, where 0 means not set)
- `getTargetFDRType()` -> int — `DS_COUNT_PSM` or `DS_COUNT_SEQUENCE`
- `isUsePeptideSummary()` -> bool — hint whether peptide or protein summary should be created

### Setter Methods

- `setFlags(flags)` — set all flags at once
- `setFlags2(flags2)` — set all pepsum flags at once
- `setFlag(flag, enabled)` — toggle individual `ms_mascotresults.FLAGS` flag
- `setFlag2(flag, enabled)` — toggle individual `ms_peptidesummary.MSPEPSUM` flag
- `setMinProbability(minProbability)` — 0-1 is probability; >1 treated as absolute score
- `setMaxHitsToReport(maxHitsToReport)`
- `setIgnoreIonsScoreBelow(ignoreIonsScoreBelow)` — -1 means use minProbability; 0-1 is probability
- `setMinPepLenInPeptideSummary(minPepLenInPeptideSummary)`
- `setMinNumSigUniqueSequences(n)` — range 1-5, default 1
- `setUnigeneIndexFile(unigeneIndexFile)`
- `setSingleHit(singleHit)` — format: `"3:CH60_HUMAN"` (db index prefix)
- `setTargetFDR(fdr)` — target FDR in range 0-1; **overrides minProbability** for target-decoy searches
- `setTargetFDRType(fdrType)` — `DS_COUNT_PSM` (default) or `DS_COUNT_SEQUENCE`
- `setUsePeptideSummary(usePeptideSummary)`

### Utility Methods

- `copyFrom(otherResParams)` — copy all settings from another params object

---

## ms_searchparams

Search parameters from the result file. Obtained via `resfile.params()`.

**Core Parameters:**
- `getLICENSE()` -> str
- `getCOM()` -> str (search title)
- `getDB()` -> str (database name)
- `getCLE()` -> str (enzyme name)
- `getSEARCH()` -> str (search type: "MIS", "PMF", "SQ")
- `getMASS()` -> str ("Monoisotopic" or "Average")
- `getFORMAT()` -> str (data format)
- `getFORMVER()` -> str
- `getINSTRUMENT()` -> str
- `getDECOY()` -> int (>0 if decoy search enabled)

**Tolerances:**
- `getTOL()` -> float (peptide mass tolerance)
- `getTOLU()` -> str (units: "Da", "ppm", etc.)
- `getITOL()` -> float (fragment ion tolerance)
- `getITOLU()` -> str (units)
- `getSEG()` -> float

**Modifications:**
- `getMODS()` -> str (fixed modifications)
- `getIT_MODS()` -> str (variable modifications)
- `getPFA()` -> int (missed cleavages)
- `getVarModsName(i)` -> str (1-based index; empty string when no more)
- `getVarModsDelta(i)` -> float
- `getVarModsNeutralLoss(i)` -> float

**User Info:**
- `getUSERNAME()` -> str
- `getUSEREMAIL()` -> str
- `getUSERField(i)` -> str (USER00-USER11, 0-based)

**Other:**
- `getFILENAME()` -> str (raw data filename)
- `getQUE()` -> str (input data)
- `getCHARGE()` -> str
- `getREPORT()` -> int (number of hits to display)
- `getOVERVIEW()` -> bool
- `getINTERMEDIATE()` -> str (repeat search file)
- `getPRECURSOR()` -> float
- `getTAXONOMY()` -> str
- `getREPTYPE()` -> str
- `getACCESSION()` -> str
- `getSUBCLUSTER()` -> int
- `getICAT()` -> bool
- `getERRORTOLERANT()` -> bool
- `getRULES()` -> str (ion series rules)

**Mass Values:**
- `getResidueMass(residue_char)` -> float (e.g. `getResidueMass('A')`)
- `getCTermMass()` -> float
- `getNTermMass()` -> float
- `getHydrogenMass()` -> float
- `getOxygenMass()` -> float
- `getCarbonMass()` -> float
- `getNitrogenMass()` -> float
- `getElectronMass()` -> float

---

## ms_peptidesummary

Peptide-level result summary for MS/MS searches. Inherits from ms_mascotresults.

### Recommended constructor (two-argument)

```python
results = msparser.ms_peptidesummary(resfile, resParams)
```

Where `resParams` is an `ms_mascotresults_params` object (see above).

### Obsolete constructor (do not use in new code)

```python
# OBSOLETE — shown only for reference when reading old code
results = msparser.ms_peptidesummary(
    resfile, flags, minProbability, maxHitsToReport,
    unigeneIndexFile, ignoreIonsScoreBelow,
    minPepLenInPepSummary, unigeneIndexFile2, flags2
)
```

**Protein Access (inherited from ms_mascotresults):**
- `getHit(hitNumber)` -> ms_protein or None (1-based)
- `getNumberOfHits()` -> int
- `getProteinDescription(accession)` -> str
- `getProteinMass(accession)` -> float
- `getNextSimilarProteinOf(accession, dbIdx, i)` -> ms_protein or None (1-based i)
- `getNextSubsetProteinOf(accession, dbIdx, i)` -> ms_protein or None (1-based i)
- `getNextFamilyProtein(hit, family)` -> ms_protein or None (for cluster mode)

**Peptide Access:**
- `getPeptide(query, rank)` -> ms_peptide or None (both 1-based)
- `getReadableVarMods(query, rank)` -> str (human-readable modification string)
- `getProteinsWithThisPepMatch(query, rank)` -> str

**Significance & Thresholds:**
- `getPeptideThreshold(query, sigLevel, rank)` -> float — effective significance threshold for this PSM
- `getPeptideIdentityThreshold(query, sigLevel)` -> float — identity threshold
- `getHomologyThreshold(query, sigLevel)` -> float — homology threshold
- `getProbOfPepBeingRandomMatch(score, query)` -> float — probability of random match

**Unassigned Peptides:**
- `createUnassignedList(sortOrder)` - call first with `msparser.ms_mascotresults.SCORE`
- `getNumberOfUnassigned()` -> int
- `getUnassigned(index)` -> ms_peptide (1-based)

---

## ms_proteinsummary

Protein-level result summary for PMF searches. Inherits from ms_mascotresults.

### Recommended constructor (two-argument)

```python
results = msparser.ms_proteinsummary(resfile, resParams)
```

### Obsolete constructor

```python
# OBSOLETE
results = msparser.ms_proteinsummary(resfile, flags, minProbability, maxHitsToReport)
```

Same protein/peptide access methods as ms_peptidesummary (inherited from ms_mascotresults).

---

## ms_protein

Individual protein hit. Obtained from `results.getHit(n)`.

- `getAccession()` -> str
- `getScore()` -> float
- `getFrame()` -> int (ORF frame, 0 for protein databases)
- `getCoverage()` -> int (number of residues covered)
- `getRMSDeltas(results)` -> float (RMS mass error)
- `getDB()` -> int (database index, 1-based for multi-DB)
- `getNumPeptides()` -> int (total matched peptides)
- `getNumDisplayPeptides()` -> int (display count)

**Per-Peptide Access (1-based index i):**
- `getPeptideQuery(i)` -> int (query number, -1 if none)
- `getPeptideP(i)` -> int (rank, -1 if none)
- `getPeptideDuplicate(i)` -> int (check against `ms_protein.DUPE_DuplicateSameQuery`, `ms_protein.DUPE_Duplicate`)
- `getPeptideIsBold(i)` -> bool
- `getPeptideShowCheckbox(i)` -> bool
- `getPeptideStart(i)` -> int (start position in protein)
- `getPeptideEnd(i)` -> int (end position in protein)
- `getPeptideResidueBefore(i)` -> str
- `getPeptideResidueAfter(i)` -> str

---

## ms_peptide

Individual peptide match. Obtained from `results.getPeptide(query, rank)`.

- `getAnyMatch()` -> bool (True if matched)
- `getQuery()` -> int
- `getRank()` -> int
- `getPrettyRank()` -> int
- `getPeptideStr()` -> str (amino acid sequence, e.g. "ALMLQGVDLLADAVAVTMGPK")
- `getIonsScore()` -> float (Mascot ions score)
- `getScore()` -> float (same as getIonsScore)
- `getExpectationValue()` -> float (E-value; significant if <= 0.05)
- `getObserved()` -> float (observed m/z)
- `getCharge()` -> int
- `getMrCalc()` -> float (calculated Mr)
- `getMrExperimental()` -> float (experimental Mr)
- `getDelta()` -> float (mass error)
- `getMissedCleavages()` -> int
- `getNumIonsMatched()` -> int
- `getVarModsStr()` -> str (modification position string)
- `getSeriesUsedStr()` -> str
- `getPeaksUsedFromIons1()` -> int
- `getPeaksUsedFromIons2()` -> int
- `getPeaksUsedFromIons3()` -> int
- `getPeaksUsedFromIons()` -> int

---

## ms_inputquery

Raw input spectrum/query data from the result file.

```python
inp = msparser.ms_inputquery(resfile, queryNumber)  # 1-based
```

- `getStringTitle(urlDecode)` -> str (pass True for decoded title)
- `getMassMin()` -> float
- `getMassMax()` -> float
- `getIntMin()` -> float
- `getIntMax()` -> float
- `getNumVals()` -> int
- `getNumUsed()` -> int
- `getStringIons1()` -> str (peak list as string)
- `getStringIons2()` -> str
- `getStringIons3()` -> str
- `getNumberOfPeaks(series)` -> int (series=1 for main peak list)
- `getPeakMass(series, index)` -> float (1-based index)
- `getPeakIntensity(series, index)` -> float (1-based index)
- `getPepTol()` -> float
- `getPepTolUnits()` -> str
- `getPepTolString()` -> str
- `getRetentionTimes()` -> str
- `getScanNumbers()` -> str
- `getRawfile()` -> str
- `getIonMobility()` -> float

---

## ms_http_client

HTTP client for connecting to Mascot Server.

```python
settings = msparser.ms_connection_settings()
settings.setProxyServerType(msparser.ms_connection_settings.PROXY_TYPE_NO_PROXY)
settings.setUserAgent("MyScript/1.0 " + settings.getUserAgent())

client = msparser.ms_http_client(baseUrl, settings)
```

- `isValid()` -> bool
- `baseUrl()` -> str
- `userLogin(username, password, session)` -> int
  - Returns: `ms_http_client.L_SUCCESS`, `ms_http_client.L_SECURITYDISABLED`, or error code
- `getLastErrorString()` -> str
- `getErrorHandler()` -> ms_errs

---

## ms_http_client_session

Session object for authenticated Mascot operations.

```python
session = msparser.ms_http_client_session()
rc = client.userLogin(username, password, session)
```

- `sessionId()` -> str
- `submitSearch(search, httpHeader, prologue, mgfFile, epilogue, progress)` -> bool
- `getSequenceFile(database, accessions, frames, localFileName)` -> bool
- `logout()`
- `isValid()` -> bool
- `getLastErrorString()` -> str

---

## ms_http_client_search

Search operations via HTTP.

```python
search = msparser.ms_http_client_search(session, taskID)
```

- `isValid()` -> bool
- `searchTaskId()` -> str
- `getStatus()` -> tuple(success: bool, returnCode: int, returnValue: int)
  - Return codes: `SS_UNKNOWN`, `SS_ASSIGNED`, `SS_QUEUED`, `SS_RUNNING`, `SS_COMPLETE`, `SS_ERROR`, `SS_SEARCH_CONTROL_ERROR`
- `getResultsFileName()` -> tuple(ok: bool, remoteFileName: str)
- `downloadResultsFile(localFileName, progress)` -> bool

---

## ms_datfile

Configuration file access (mascot.dat).

```python
datfile = msparser.ms_datfile(filepath)
# Or with connection settings for URL access:
datfile = msparser.ms_datfile(url, 0, connectionSettings)
```

- `isValid()` -> bool
- `getLastErrorString()` -> str
- `getDatabases()` -> ms_databases
- `getMascotOptions()` -> ms_mascotoptions
- `getParseOptions()` -> ms_parseoptions
- `getWWWOptions()` -> ms_wwwoptions
- `getProcessors()` -> ms_processors
- `getClusterParams()` -> ms_clusterparams
- `getCronOptions()` -> ms_cronoptions
- `getMaxTaxonomyRules()` -> int
- `getTaxonomyRules(index)` -> ms_taxonomyrules or None

---

## ms_databases

Database configuration from mascot.dat.

- `isSectionAvailable()` -> bool
- `getNumberOfDatabases()` -> int
- `getDatabase(index)` -> ms_databaseoptions (0-based)

### ms_databaseoptions
- `getName()` -> str
- `isActive()` -> bool

---

## ms_connection_settings

Connection configuration for HTTP operations.

- `setProxyServerType(type)` - `PROXY_TYPE_NO_PROXY`, `PROXY_TYPE_AUTO`
- `setUserAgent(agent)` / `getUserAgent()` -> str
- `setSessionID(sessionId)` / `getSessionID()` -> str

---

## ms_http_helper_progress

Progress tracking for HTTP operations.

```python
progress = msparser.ms_http_helper_progress()
```

Used as parameter to `submitSearch()` and `downloadResultsFile()`.

---

## ms_aahelper

Amino acid calculation utilities.

```python
aahelper = msparser.ms_aahelper()
```

Provides mass calculation utilities for amino acid sequences.

---

## ms_mascotoptions

Mascot server options from mascot.dat.

- `isSectionAvailable()` -> bool
- `getMascotCmdLine()` -> str
- `getPercolatorMinQueries()` -> int
- `getPercolatorMinSequences()` -> int

---

## Utility Classes

### VectorString / vectori / vectord
SWIG vector wrappers for passing lists to msparser functions.

```python
accessions = msparser.VectorString()
accessions.append("P02768")
frames = msparser.vectori()
```

### ms_errs (Error Handler)
- `setLoggingFile(filename, severity)` - e.g. `ms_errs.sev_debug3`
- `getNumberOfErrors()` -> int
- `getErrorNumber(i)` -> int
- `getErrorRepeats(i)` -> int

---

## Constants & Flags

### Result Display Flags (ms_mascotresults)
- `MSRES_NOFLAG` = 0
- `MSRES_GROUP_PROTEINS` - simple parsimony grouping
- `MSRES_SHOW_SUBSETS` - show subset proteins
- `MSRES_CLUSTER_PROTEINS` - hierarchical clustering
- `MSRES_MUDPIT_PROTEIN_SCORE` - MudPIT scoring

### Peptide Summary Flags (ms_peptidesummary)
- `MSPEPSUM_NONE` - no flags
- `MSPEPSUM_USE_HOMOLOGY_THRESH` - use homology threshold
- `MSPEPSUM_PERCOLATOR` - enable Percolator rescoring
- `MSPEPSUM_SINGLE_HIT_DBIDX` - singleHit uses db index prefix format

### Sort Orders (ms_mascotresults)
- `SCORE` - sort by score

### FDR Count Types (ms_mascotresults)
- `DS_COUNT_PSM` - FDR at PSM level
- `DS_COUNT_SEQUENCE` - FDR at unique sequence level

### Protein Duplicate Types (ms_protein)
- `DUPE_DuplicateSameQuery` - same peptide from same query
- `DUPE_Duplicate` - duplicate from different query
- `DUPE_NotDuplicate` - not a duplicate

### Login Return Codes (ms_http_client)
- `L_SUCCESS` - login successful
- `L_SECURITYDISABLED` - security not enabled on server

### Search Status Codes (ms_http_client_search)
- `SS_UNKNOWN`, `SS_ASSIGNED`, `SS_QUEUED`, `SS_RUNNING`, `SS_COMPLETE`, `SS_ERROR`, `SS_SEARCH_CONTROL_ERROR`
