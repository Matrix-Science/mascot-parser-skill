# msparser Java API Reference

Exact Java method names and calling conventions for msparser. **Use this reference
when writing Java code.** The API method names are identical across all bindings;
only constructors, library loading, constant access, and out-parameter handling differ.

## Setup

### Library Loading

Every Java program using msparser must load the native library in a static initializer:

```java
import matrix_science.msparser.*;

public class MyScript {
    static {
        try {
            System.loadLibrary("msparserj");
        } catch (UnsatisfiedLinkError e) {
            System.err.println("Native code library failed to load. "
                + "Is msparserj.dll on the path?\n" + e);
            System.exit(1);
        }
    }

    public static void main(String[] args) {
        // ...
    }
}
```

The `msparserj.dll` (Windows) or `libmsparserj.so` (Linux) must be on the Java library path. Set via `-Djava.library.path=<MSPARSER_SDK>/java/` or system PATH.

The Java wrapper JAR is at `<MSPARSER_SDK>/java/msparser.jar`. Add to classpath:
```
java -classpath ".;<MSPARSER_SDK>/java/msparser.jar" MyScript
```

## Critical Java vs Python Differences

| Concept | Python | Java |
|---|---|---|
| Import | `import msparser` | `import matrix_science.msparser.*;` |
| Constructor | `msparser.ms_inputquery(resfile, q)` | `new ms_inputquery(resfile, q)` |
| Static factory | `msparser.ms_mascotresfilebase.createResfile(p)` | `ms_mascotresfilebase.createResfile(p)` |
| Constants | `msparser.ms_protein.DUPE_DuplicateSameQuery` | `ms_protein.DUPE_DuplicateSameQuery` |
| Flags | `msparser.ms_mascotresults.MSRES_GROUP_PROTEINS` | `ms_mascotresults.MSRES_GROUP_PROTEINS` |
| None check | `if prot:` | `if (prot != null)` |
| Boolean True | `True` | `true` |
| String empty | `if not seq:` | `if (seq.equals(""))` |

## Opening Result Files

```java
ms_mascotresfilebase file = ms_mascotresfilebase.createResfile(path);
// NOTE: lowercase 'f' in createResfile
if (!checkErrors(file)) {
    return;
}
```

### Error Checking Pattern

```java
private static boolean checkErrors(ms_mascotresfilebase file) {
    if (file.getLastError() > 0) {
        for (int i = 1; i <= file.getNumberOfErrors(); i++) {
            System.out.println("Error number: " + file.getErrorNumber(i)
                + " : " + file.getErrorString(i));
        }
    }
    // Call isValid before clearAllErrors, otherwise always returns true
    boolean bIsValid = file.isValid();
    file.clearAllErrors();
    return bIsValid;
}
```

## Search Parameters

```java
ms_searchparams p = file.params();

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

```java
ms_datfile datfile = new ms_datfile("<MASCOT_HOME>/config/mascot.dat");
ms_mascotoptions mascotOptions = new ms_mascotoptions();
if (datfile.isValid()) {
    mascotOptions = datfile.getMascotOptions();
}

ms_mascotresults_params resParams = new ms_mascotresults_params();
String scriptName = file.get_ms_mascotresults_params(mascotOptions, resParams);
```

### Step 2: Override as needed

```java
// Set target FDR (decimal, not percentage)
resParams.setTargetFDR(0.01);  // 1% FDR
resParams.setTargetFDRType(ms_mascotresults.DS_COUNT_PSM);

// Require multiple significant unique peptides per protein
resParams.setMinNumSigUniqueSequences(2);

// Enable Percolator (if available)
if (file.params().getDECOY() > 0 && file.anyMSMS() && !file.isErrorTolerant()) {
    long flags2 = resParams.getFlags2();
    flags2 |= ms_peptidesummary.MSPEPSUM_PERCOLATOR;
    resParams.setFlags2(flags2);
}
```

### Step 3: Create the summary

```java
ms_peptidesummary results;
if (resParams.isUsePeptideSummary()) {
    results = new ms_peptidesummary(file, resParams);
} else {
    ms_proteinsummary protResults = new ms_proteinsummary(file, resParams);
}
```

### Obsolete pattern (do not use in new code)

The old constructor uses Java's single-element array trick for out-parameters:

```java
// OBSOLETE — shown for reference when reading old code
boolean[] usePeptideSummary = {false};
long[]    flags = {0};
double[]  minProbability = {0};
int[]     maxHitsToReport = {0};
double[]  ignoreIonsScoreBelow = {0};
long[]    minPepLenInPepSummary = {0};
long[]    flags2 = {0};

String scriptName = file.get_ms_mascotresults_params(
    mascotOptions, flags, minProbability, maxHitsToReport,
    ignoreIonsScoreBelow, minPepLenInPepSummary, usePeptideSummary, flags2
);

// Access values from arrays:
if (usePeptideSummary[0]) {
    results = new ms_peptidesummary(file, (int)flags[0], minProbability[0],
        maxHitsToReport[0], "", ignoreIonsScoreBelow[0],
        (int)minPepLenInPepSummary[0], null, (int)flags2[0]);
}
```

## Scores and Significance

```java
// These two tests are equivalent:
boolean significant = pep.getIonsScore() >= results.getPeptideThreshold(q, 20, rank);
// same as:
boolean significant = pep.getExpectationValue() <= 0.05;
```

### Threshold methods

```java
results.getPeptideIdentityThreshold(q, 20)       // Identity threshold at p=0.05
results.getHomologyThreshold(q, 20)               // Homology threshold at p=0.05
results.getPeptideThreshold(q, 20, rank)          // Effective threshold for this PSM
results.getProbOfPepBeingRandomMatch(score, q)    // Probability of random match
```

## Iterating Protein Hits

```java
int hit = 1;
ms_protein prot = results.getHit(hit);

while (hit <= results.getNumberOfHits()) {
    String accession = prot.getAccession();
    String description = results.getProteinDescription(accession);
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

```java
int numPeps = prot.getNumPeptides();
for (int i = 1; i <= numPeps; i++) {
    int query = prot.getPeptideQuery(i);
    int p = prot.getPeptideP(i);

    if (p == -1 || query == -1) continue;
    if (prot.getPeptideDuplicate(i) == ms_protein.DUPE_DuplicateSameQuery) continue;

    ms_peptide pep = results.getPeptide(query, p);
    if (pep == null) continue;

    // Peptide data
    String seq = pep.getPeptideStr();
    double score = pep.getIonsScore();
    double expect = pep.getExpectationValue();
    double observed = pep.getObserved();
    int charge = pep.getCharge();
    double mrCalc = pep.getMrCalc();
    double delta = pep.getDelta();
    int missed = pep.getMissedCleavages();
    String varMods = pep.getVarModsStr();
    String readableMods = results.getReadableVarMods(query, p, 2);

    // Significance test
    double threshold = results.getPeptideThreshold(query, 20, p);
    boolean significant = score >= threshold;

    // Context from protein
    boolean isBold = prot.getPeptideIsBold(i);
    boolean isDuplicate = prot.getPeptideDuplicate(i) == ms_protein.DUPE_Duplicate;
}
```

## Input Query / Spectrum Data

```java
ms_inputquery inp = new ms_inputquery(file, query);  // 1-based
String title = inp.getStringTitle(true);
int numPeaks = inp.getNumberOfPeaks(1);

for (int i = 1; i <= numPeaks; i++) {
    double mz = inp.getPeakMass(1, i);
    double intensity = inp.getPeakIntensity(1, i);
}
```

## Configuration (mascot.dat)

```java
ms_datfile datfile = new ms_datfile("<MASCOT_HOME>/config/mascot.dat");
// Or with connection settings for URL access:
ms_connection_settings cs = new ms_connection_settings();
cs.setSessionID(sessionId);
ms_datfile datfile = new ms_datfile(url, 0, cs);

if (datfile.isValid()) {
    ms_databases dbs = datfile.getDatabases();
    if (dbs.isSectionAvailable()) {
        for (int i = 0; i < dbs.getNumberOfDatabases(); i++) {
            System.out.println(dbs.getDatabase(i).getName()
                + " : " + (dbs.getDatabase(i).isActive() ? "active" : "inactive"));
        }
    }
}
```

## HTTP Client & Authentication

```java
ms_connection_settings settings = new ms_connection_settings();
settings.setProxyServerType(ms_connection_settings.PROXY_TYPE_NO_PROXY);
settings.setUserAgent("JavaScript/1.0 " + settings.getUserAgent());

ms_http_client client = new ms_http_client(mascotUrl, settings);
if (!client.isValid()) {
    System.err.println("Connection failed: " + client.getLastErrorString());
    return;
}

ms_http_client_session session = new ms_http_client_session();
int rc = client.userLogin(username, password, session);

if (rc == ms_http_client.L_SUCCESS) {
    System.out.println("Logged in. Session: " + session.sessionId());
} else if (rc == ms_http_client.L_SECURITYDISABLED) {
    System.out.println("Security disabled");
} else {
    System.err.println("Login failed: " + rc);
}
```

## Constants and Flags

Constants are accessed directly on the class:

```java
// Duplicate flags
ms_protein.DUPE_DuplicateSameQuery
ms_protein.DUPE_Duplicate
ms_protein.DUPE_NotDuplicate

// Result flags
ms_mascotresults.MSRES_GROUP_PROTEINS
ms_mascotresults.MSRES_SHOW_SUBSETS
ms_mascotresults.MSRES_CLUSTER_PROTEINS
ms_mascotresults.MSRES_MUDPIT_PROTEIN_SCORE
ms_mascotresults.SCORE

// FDR count types
ms_mascotresults.DS_COUNT_PSM
ms_mascotresults.DS_COUNT_SEQUENCE

// Peptide summary flags
ms_peptidesummary.MSPEPSUM_USE_HOMOLOGY_THRESH
ms_peptidesummary.MSPEPSUM_PERCOLATOR

// Bitwise flag operations
int flags = (int) resParams.getFlags();
flags |= ms_mascotresults.MSRES_GROUP_PROTEINS;
flags &= ~ms_mascotresults.MSRES_SHOW_SUBSETS;
resParams.setFlags(flags);
```

## Unassigned Peptides

```java
results.createUnassignedList(ms_mascotresults.SCORE);
for (int u = 1; u <= results.getNumberOfUnassigned(); u++) {
    ms_peptide pep = results.getUnassigned(u);
    if (pep != null && pep.getPeptideStr().length() > 0) {
        System.out.println(pep.getPeptideStr() + " " + pep.getIonsScore());
    }
}
```

## Complete Working Pattern: Export All Significant PSMs

```java
import matrix_science.msparser.*;

public class ExportPSMs {
    static {
        try {
            System.loadLibrary("msparserj");
        } catch (UnsatisfiedLinkError e) {
            System.err.println("Native library failed to load: " + e);
            System.exit(1);
        }
    }

    public static void main(String[] args) {
        if (args.length < 1) {
            System.out.println("Usage: ExportPSMs <result_file>");
            return;
        }

        ms_mascotresfilebase resfile = ms_mascotresfilebase.createResfile(args[0]);
        if (!resfile.isValid()) {
            System.err.println(resfile.getLastErrorString());
            return;
        }

        ms_datfile datfile = new ms_datfile("<MASCOT_HOME>/config/mascot.dat");
        ms_mascotoptions mascotOptions = datfile.isValid()
            ? datfile.getMascotOptions() : new ms_mascotoptions();

        ms_mascotresults_params resParams = new ms_mascotresults_params();
        resfile.get_ms_mascotresults_params(mascotOptions, resParams);

        // Optional: set target FDR
        if (resfile.params().getDECOY() > 0) {
            resParams.setTargetFDR(0.01);
            resParams.setTargetFDRType(ms_mascotresults.DS_COUNT_PSM);
        }

        ms_peptidesummary results = new ms_peptidesummary(resfile, resParams);
        if (!resfile.isValid()) {
            System.err.println(resfile.getLastErrorString());
            return;
        }

        int hit = 1;
        ms_protein prot = results.getHit(hit);
        while (prot != null) {
            String acc = prot.getAccession();
            String desc = results.getProteinDescription(acc);

            for (int i = 1; i <= prot.getNumPeptides(); i++) {
                int query = prot.getPeptideQuery(i);
                int rank = prot.getPeptideP(i);
                if (query == -1 || rank == -1) continue;
                if (prot.getPeptideDuplicate(i) == ms_protein.DUPE_DuplicateSameQuery) continue;

                ms_peptide pep = results.getPeptide(query, rank);
                if (pep == null) continue;

                double score = pep.getIonsScore();
                double threshold = results.getPeptideThreshold(query, 20, rank);
                if (score < threshold) continue;  // Only significant PSMs

                System.out.println(String.format("%s\t%s\t%d\t%d\t%.2f\t%.4g\t%s",
                    acc, pep.getPeptideStr(), query, rank,
                    score, pep.getExpectationValue(),
                    results.getReadableVarMods(query, rank, 2)));
            }

            hit++;
            prot = results.getHit(hit);
        }
    }
}
```
