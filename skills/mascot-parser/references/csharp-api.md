# msparser C# API Reference

Exact C# method names and calling conventions for msparser. **Use this reference
when writing C# code.** The API method names are identical across all bindings;
only constructors, constant/enum access, and out-parameter handling differ.

## Setup

### Assembly Reference

Add a reference to the msparser .NET assembly from `<MSPARSER_SDK>/csharp_vs2019/`. The native DLL (`msparser.dll`) must be accessible at runtime (copy to output directory or add to PATH).

```csharp
using System;
using matrix_science.msparser;

namespace MyProject
{
    public class MyScript
    {
        public static int Main(string[] args)
        {
            // ...
            return 0;
        }
    }
}
```

No explicit `System.loadLibrary()` call is needed — the .NET runtime handles native library loading.

## Critical C# vs Python Differences

| Concept | Python | C# |
|---|---|---|
| Import | `import msparser` | `using matrix_science.msparser;` |
| Constructor | `msparser.ms_inputquery(resfile, q)` | `new ms_inputquery(resfile, q)` |
| Static factory | `msparser.ms_mascotresfilebase.createResfile(p)` | `ms_mascotresfilebase.createResfile(p, 0, "")` |
| **params()** | `resfile.params()` | `resfile._params()` **(underscore!)** |
| Constants | `msparser.ms_protein.DUPE_DuplicateSameQuery` | `ms_protein.DUPLICATE.DUPE_DuplicateSameQuery` |
| Flags | `msparser.ms_mascotresults.MSRES_GROUP_PROTEINS` | `(uint)ms_mascotresults.FLAGS.MSRES_GROUP_PROTEINS` |
| Sort order | `msparser.ms_mascotresults.SCORE` | `ms_mascotresults.sortBy.SCORE` |
| None check | `if prot:` | `if (prot != null)` |
| Boolean | `True` / `False` | `true` / `false` |
| String empty | `if not seq:` | `if (seq.Length == 0)` |

### C#-Specific Gotchas

1. **`_params()` not `params()`** — `params` is a C# keyword, so the method is renamed to `_params()`
2. **Enum wrapper types** — Constants are accessed through nested enum types (e.g. `ms_mascotresults.FLAGS.MSRES_GROUP_PROTEINS`)
3. **Cast to uint** — Flag values often need explicit `(uint)` cast for bitwise operations
4. **`GC.KeepAlive()`** — Call `GC.KeepAlive(resfile)` at the end of processing to prevent premature garbage collection of the native object while `ms_mascotresults` still references it
5. **`out` keyword** — The old `get_ms_mascotresults_params` uses C# `out` parameters (not arrays like Java)
6. **`createResfile` takes 3 args** — `createResfile(path, 0, "")` (mode and session string)

## Opening Result Files

```csharp
ms_mascotresfilebase file = ms_mascotresfilebase.createResfile(path, 0, "");
if (!checkErrors(file))
{
    return;
}

// ... use file ...

// IMPORTANT: prevent GC from collecting file while results reference it
GC.KeepAlive(file);
```

### Error Checking Pattern

```csharp
private static bool checkErrors(ms_mascotresfilebase file)
{
    for (int i = 1; i <= file.getNumberOfErrors(); i++)
    {
        Console.WriteLine("Error number: {0} : {1}",
            file.getErrorNumber(i), file.getErrorString(i));
    }
    // Call isValid before clearAllErrors, otherwise always returns true
    bool isValid = file.isValid();
    file.clearAllErrors();
    return isValid;
}
```

## Search Parameters

**Note the underscore: `_params()` not `params()`.**

```csharp
ms_searchparams p = file._params();

p.getCOM()          // search title
p.getDB()           // database name
p.getCLE()          // enzyme
p.getPFA()          // missed cleavages
p.getMODS()         // fixed modifications
p.getIT_MODS()      // variable modifications
p.getTOL()          // peptide tolerance
p.getTOLU()         // tolerance units
p.getITOL()         // fragment tolerance
p.getITOLU()        // fragment tolerance units
p.getSEARCH()       // "MIS", "PMF", "SQ"
p.getMASS()         // "Monoisotopic" or "Average"
p.getCHARGE()       // charge state
p.getINSTRUMENT()   // instrument type
p.getDECOY()        // >0 if decoy search
```

## Creating Summaries: Two-Argument Constructor (Recommended)

### Step 1: Get default parameters

```csharp
ms_datfile datfile = new ms_datfile("<MASCOT_HOME>/config/mascot.dat");
ms_mascotoptions mascotOptions = new ms_mascotoptions();
if (datfile.isValid())
{
    mascotOptions = datfile.getMascotOptions();
}

ms_mascotresults_params resParams = new ms_mascotresults_params();
string scriptName = file.get_ms_mascotresults_params(mascotOptions, resParams);
```

### Step 2: Override as needed

```csharp
// Set target FDR (decimal, not percentage)
resParams.setTargetFDR(0.01);  // 1% FDR
resParams.setTargetFDRType(ms_mascotresults.DS_COUNT_PSM);

// Require multiple significant unique peptides per protein
resParams.setMinNumSigUniqueSequences(2);

// Enable Percolator (if available)
if (file._params().getDECOY() > 0 && file.anyMSMS() && !file.isErrorTolerant())
{
    uint flags2 = resParams.getFlags2();
    flags2 |= (uint)ms_peptidesummary.MSPEPSUM_FLAGS.MSPEPSUM_PERCOLATOR;
    resParams.setFlags2(flags2);
}
```

### Step 3: Create the summary

```csharp
ms_mascotresults results;
if (resParams.isUsePeptideSummary())
{
    results = new ms_peptidesummary(file, resParams);
}
else
{
    results = new ms_proteinsummary(file, resParams);
}
```

### Obsolete pattern (do not use in new code)

The old constructor uses C#'s `out` keyword:

```csharp
// OBSOLETE — shown for reference when reading old code
uint flags, flags2, minPepLenInPepSummary;
int maxHitsToReport;
double minProbability, ignoreIonsScoreBelow;
bool usePeptideSummary;

string scriptName = file.get_ms_mascotresults_params(
    mascotOptions,
    out flags, out minProbability, out maxHitsToReport,
    out ignoreIonsScoreBelow, out minPepLenInPepSummary,
    out usePeptideSummary, out flags2
);

if (usePeptideSummary)
{
    results = new ms_peptidesummary(file, flags, minProbability,
        maxHitsToReport, "", ignoreIonsScoreBelow,
        (int)minPepLenInPepSummary, null, flags2);
}
```

## Scores and Significance

```csharp
// These two tests are equivalent:
bool significant = pep.getIonsScore() >= results.getPeptideThreshold(q, 20, rank);
// same as:
bool significant = pep.getExpectationValue() <= 0.05;
```

### Threshold methods

```csharp
results.getPeptideIdentityThreshold(q, 20)       // Identity threshold at p=0.05
results.getHomologyThreshold(q, 20)               // Homology threshold at p=0.05
results.getPeptideThreshold(q, 20, rank)          // Effective threshold for this PSM
results.getProbOfPepBeingRandomMatch(score, q)    // Probability of random match
```

## Iterating Protein Hits

```csharp
int hit = 1;
ms_protein prot = results.getHit(hit);

while (hit <= results.getNumberOfHits())
{
    string accession = prot.getAccession();
    string description = results.getProteinDescription(accession);
    double mass = results.getProteinMass(accession);
    double score = prot.getScore();
    int coverage = prot.getCoverage();
    int dbIdx = prot.getDB();

    // Process protein...

    hit++;
    prot = results.getHit(hit);
}
```

## Iterating Peptides for a Protein

```csharp
int numPeps = prot.getNumPeptides();
for (int i = 1; i <= numPeps; i++)
{
    int query = prot.getPeptideQuery(i);
    int p = prot.getPeptideP(i);

    if (p == -1 || query == -1) continue;
    if (prot.getPeptideDuplicate(i) == ms_protein.DUPLICATE.DUPE_DuplicateSameQuery) continue;

    ms_peptide pep = results.getPeptide(query, p);
    if (pep == null) continue;

    // Peptide data
    string seq = pep.getPeptideStr();
    double score = pep.getIonsScore();
    double expect = pep.getExpectationValue();
    double observed = pep.getObserved();
    int charge = pep.getCharge();
    double mrCalc = pep.getMrCalc();
    double delta = pep.getDelta();
    int missed = pep.getMissedCleavages();
    string varMods = pep.getVarModsStr();
    string readableMods = results.getReadableVarMods(query, p, 2);

    // Significance test
    double threshold = results.getPeptideThreshold(query, 20, p);
    bool significant = score >= threshold;

    // Context from protein
    bool isBold = prot.getPeptideIsBold(i);
    bool isDuplicate = prot.getPeptideDuplicate(i) == ms_protein.DUPLICATE.DUPE_Duplicate;
}
```

## Input Query / Spectrum Data

```csharp
ms_inputquery inp = new ms_inputquery(file, queryNumber);  // 1-based
string title = inp.getStringTitle(true);
int numPeaks = inp.getNumberOfPeaks(1);

for (int i = 1; i <= numPeaks; i++)
{
    double mz = inp.getPeakMass(1, i);
    double intensity = inp.getPeakIntensity(1, i);
}
```

## Configuration (mascot.dat)

```csharp
ms_datfile datfile = new ms_datfile("<MASCOT_HOME>/config/mascot.dat");
// Or with connection settings for URL access:
ms_connection_settings cs = new ms_connection_settings();
cs.setProxyServerType(ms_connection_settings.PROXY_TYPE.PROXY_TYPE_AUTO);
cs.setSessionID(sessionId);
ms_datfile datfile = new ms_datfile(url, 0, cs);

if (datfile.isValid())
{
    ms_databases dbs = datfile.getDatabases();
    if (dbs.isSectionAvailable())
    {
        for (int i = 0; i < dbs.getNumberOfDatabases(); i++)
        {
            Console.WriteLine("{0} : {1}",
                dbs.getDatabase(i).getName(),
                dbs.getDatabase(i).isActive() ? "active" : "inactive");
        }
    }
}
```

## HTTP Client & Authentication

```csharp
ms_connection_settings settings = new ms_connection_settings();
settings.setUserAgent("CSharpScript/1.0 " + settings.getUserAgent());
settings.setProxyServerType(ms_connection_settings.PROXY_TYPE.PROXY_TYPE_NO_PROXY);

ms_http_client client = new ms_http_client(mascotUrl, settings);
if (!client.isValid())
{
    Console.Error.WriteLine("Connection failed: " + client.getLastErrorString());
    return;
}

// Optional: enable debug logging
client.getErrorHandler().setLoggingFile("log.txt", ms_errs.msg_sev.sev_debug3);

ms_http_client_session session = new ms_http_client_session();
ms_http_client.LoginResultCode rc = client.userLogin(username, password, session);

if (rc == ms_http_client.LoginResultCode.L_SUCCESS)
{
    Console.WriteLine("Logged in. Session: " + session.sessionId());
}
else if (rc == ms_http_client.LoginResultCode.L_SECURITYDISABLED)
{
    Console.WriteLine("Security disabled");
}
else
{
    Console.Error.WriteLine("Login failed: " + rc);
}
```

## Constants and Flags

C# wraps constants in nested enum types. Cast to `uint` or `int` for bitwise operations.

```csharp
// Duplicate flags (nested under DUPLICATE enum)
ms_protein.DUPLICATE.DUPE_DuplicateSameQuery
ms_protein.DUPLICATE.DUPE_Duplicate
ms_protein.DUPLICATE.DUPE_NotDuplicate

// Result flags (nested under FLAGS enum, cast to uint)
(uint)ms_mascotresults.FLAGS.MSRES_GROUP_PROTEINS
(uint)ms_mascotresults.FLAGS.MSRES_SHOW_SUBSETS
(uint)ms_mascotresults.FLAGS.MSRES_CLUSTER_PROTEINS
(uint)ms_mascotresults.FLAGS.MSRES_MUDPIT_PROTEIN_SCORE

// Sort order (nested under sortBy enum)
ms_mascotresults.sortBy.SCORE

// FDR count types
ms_mascotresults.DS_COUNT_PSM
ms_mascotresults.DS_COUNT_SEQUENCE

// Peptide summary flags
(uint)ms_peptidesummary.MSPEPSUM_FLAGS.MSPEPSUM_USE_HOMOLOGY_THRESH
(uint)ms_peptidesummary.MSPEPSUM_FLAGS.MSPEPSUM_PERCOLATOR

// Proxy types (nested under PROXY_TYPE enum)
ms_connection_settings.PROXY_TYPE.PROXY_TYPE_NO_PROXY
ms_connection_settings.PROXY_TYPE.PROXY_TYPE_AUTO

// Login codes (nested under LoginResultCode enum)
ms_http_client.LoginResultCode.L_SUCCESS
ms_http_client.LoginResultCode.L_SECURITYDISABLED

// Search status (nested under SearchStatusCode enum)
(int)ms_http_client_search.SearchStatusCode.SS_COMPLETE
(int)ms_http_client_search.SearchStatusCode.SS_ERROR

// Bitwise flag operations — cast to uint
uint flags = resParams.getFlags();
flags |= (uint)ms_mascotresults.FLAGS.MSRES_GROUP_PROTEINS;
flags &= ~(uint)ms_mascotresults.FLAGS.MSRES_SHOW_SUBSETS;
resParams.setFlags(flags);
```

## Unassigned Peptides

```csharp
results.createUnassignedList(ms_mascotresults.sortBy.SCORE);
for (int u = 1; u <= results.getNumberOfUnassigned(); u++)
{
    ms_peptide pep = results.getUnassigned(u);
    if (pep != null && pep.getPeptideStr().Length > 0)
    {
        Console.WriteLine("{0} {1}", pep.getPeptideStr(), pep.getIonsScore());
    }
}
```

## Complete Working Pattern: Export All Significant PSMs

```csharp
using System;
using matrix_science.msparser;

namespace ExportPSMs
{
    public class Program
    {
        public static int Main(string[] args)
        {
            if (args.Length < 1)
            {
                Console.WriteLine("Usage: ExportPSMs <result_file>");
                return 1;
            }

            ms_mascotresfilebase resfile = ms_mascotresfilebase.createResfile(args[0], 0, "");
            if (!resfile.isValid())
            {
                Console.Error.WriteLine(resfile.getLastErrorString());
                return 1;
            }

            ms_datfile datfile = new ms_datfile("<MASCOT_HOME>/config/mascot.dat");
            ms_mascotoptions mascotOptions = datfile.isValid()
                ? datfile.getMascotOptions() : new ms_mascotoptions();

            ms_mascotresults_params resParams = new ms_mascotresults_params();
            resfile.get_ms_mascotresults_params(mascotOptions, resParams);

            // Optional: set target FDR
            if (resfile._params().getDECOY() > 0)
            {
                resParams.setTargetFDR(0.01);
                resParams.setTargetFDRType(ms_mascotresults.DS_COUNT_PSM);
            }

            ms_peptidesummary results = new ms_peptidesummary(resfile, resParams);
            if (!resfile.isValid())
            {
                Console.Error.WriteLine(resfile.getLastErrorString());
                return 1;
            }

            int hit = 1;
            ms_protein prot = results.getHit(hit);
            while (prot != null)
            {
                string acc = prot.getAccession();

                for (int i = 1; i <= prot.getNumPeptides(); i++)
                {
                    int query = prot.getPeptideQuery(i);
                    int rank = prot.getPeptideP(i);
                    if (query == -1 || rank == -1) continue;
                    if (prot.getPeptideDuplicate(i) ==
                        ms_protein.DUPLICATE.DUPE_DuplicateSameQuery) continue;

                    ms_peptide pep = results.getPeptide(query, rank);
                    if (pep == null) continue;

                    double score = pep.getIonsScore();
                    double threshold = results.getPeptideThreshold(query, 20, rank);
                    if (score < threshold) continue;  // Only significant PSMs

                    Console.WriteLine("{0}\t{1}\t{2}\t{3}\t{4:F2}\t{5:G4}\t{6}",
                        acc, pep.getPeptideStr(), query, rank,
                        score, pep.getExpectationValue(),
                        results.getReadableVarMods(query, rank, 2));
                }

                hit++;
                prot = results.getHit(hit);
            }

            // Prevent premature GC of native object
            GC.KeepAlive(resfile);
            return 0;
        }
    }
}
```
