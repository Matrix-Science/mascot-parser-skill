# msparser Perl API Reference

Exact Perl method names and calling conventions for msparser. **Use this reference
when writing Perl code.** The Python API (in the main skill file) has the same
method names but different calling syntax for constructors, constants, and objects.

## Critical Perl vs Python Differences

| Concept | Python | Perl |
|---|---|---|
| Constructor | `msparser.ms_inputquery(resfile, q)` | `msparser::ms_inputquery->new($resfile, $q)` |
| Static method | `msparser.ms_mascotresfilebase.createResfile(p)` | `msparser::ms_mascotresfilebase::createResfile($p)` |
| Constants | `msparser.ms_protein.DUPE_DuplicateSameQuery` | `$msparser::ms_protein::DUPE_DuplicateSameQuery` |
| Flags | `msparser.ms_mascotresults.MSRES_GROUP_PROTEINS` | `$msparser::ms_mascotresults::MSRES_GROUP_PROTEINS` |
| None check | `if prot:` | `if (defined $prot)` or `last unless $prot` |
| Boolean True | `inp.getStringTitle(True)` | `$inp->getStringTitle(1)` |

**Key rule:** Perl constants use `$msparser::ClassName::CONSTANT` (dollar prefix,
double-colon path, NO parentheses). Never use `ClassName::CONSTANT()` with parens.

## Opening Result Files

```perl
my $resfile = msparser::ms_mascotresfilebase::createResfile($path);
# NOTE: lowercase 'f' in createResfile
die $resfile->getLastErrorString() unless $resfile->isValid();
```

Both `.dat` (text) and `.msr` (binary compressed) files are opened the same way.

## Search Parameters

```perl
my $params = $resfile->params();   # returns ms_searchparams object

$params->getDB()           # database name (pass $i for multi-DB, 1-based)
$params->getDB(1)          # first database
$params->getNumberOfDatabases()  # how many databases
$params->getCOM()          # search title
$params->getTOL()          # peptide mass tolerance value
$params->getTOLU()         # tolerance units ("Da" or "ppm")
$params->getITOL()         # fragment ion tolerance
$params->getITOLU()        # fragment tolerance units
$params->getCLE()          # enzyme name ("Trypsin", etc.)
$params->getPFA()          # max missed cleavages
$params->getMODS()         # fixed mods string: "Carbamidomethyl (C)"
$params->getIT_MODS()      # variable mods string
$params->getINSTRUMENT()   # instrument name
$params->getSEARCH()       # search type: "MIS", "PMF", "SQ"
$params->getMASS()         # "Monoisotopic" or "Average"
$params->getCHARGE()       # charge state string
$params->getFILENAME()     # peak list filename
$params->getFORMAT()       # peak list format
$params->getQUANTITATION() # quantitation method name
$params->getDECOY()        # >0 if decoy search enabled
```

### Variable Modification Details (on ms_searchparams)

```perl
for my $i (1 .. 20) {
    my $name = $params->getVarModsName($i);
    last unless $name;
    my $delta = $params->getVarModsDelta($i);
}
```

### Fixed Modification Details (on ms_searchparams)

```perl
for my $i (1 .. 20) {
    my $name = $params->getFixedModsName($i);
    last unless $name;
    my $delta    = $params->getFixedModsDelta($i);
    my $residues = $params->getFixedModsResidues($i);
}
```

## Creating Summaries: Two-Argument Constructor (Recommended)

### Step 1: Get default parameters via ms_mascotresults_params

```perl
my $datfile = msparser::ms_datfile->new("<MASCOT_HOME>/config/mascot.dat");
my $mascotOptions = $datfile->isValid()
    ? $datfile->getMascotOptions()
    : msparser::ms_mascotoptions->new();

my $resParams = msparser::ms_mascotresults_params->new();
$resfile->get_ms_mascotresults_params($mascotOptions, $resParams);
```

### Step 2: Override as needed

```perl
# Set target FDR (decimal, not percentage)
$resParams->setTargetFDR(0.01);  # 1% FDR
$resParams->setTargetFDRType($msparser::ms_mascotresults::DS_COUNT_PSM);

# Require multiple significant unique peptides per protein
$resParams->setMinNumSigUniqueSequences(2);

# Enable Percolator (if available)
if ($resfile->params->getDECOY > 0 && $resfile->anyMSMS && !$resfile->isErrorTolerant) {
    my $flags2 = $resParams->getFlags2();
    $flags2 |= $msparser::ms_peptidesummary::MSPEPSUM_PERCOLATOR;
    $resParams->setFlags2($flags2);
}
```

### Step 3: Create the summary

```perl
my $summary;
if ($resParams->isUsePeptideSummary()) {
    $summary = msparser::ms_peptidesummary->new($resfile, $resParams);
} else {
    $summary = msparser::ms_proteinsummary->new($resfile, $resParams);
}
```

### Obsolete pattern (do not use in new code)

```perl
# OBSOLETE — shown for reference when reading old code
my ($scriptName, $flags, $minProbability, $maxHitsToReport,
    $ignoreIonsScoreBelow, $minPepLenInPepSummary,
    $usePeptideSummary, $flags2) = $resfile->get_ms_mascotresults_params($mascotOptions);

my $summary = msparser::ms_peptidesummary->new(
    $resfile, $flags, $minProbability, $maxHitsToReport,
    "", $ignoreIonsScoreBelow, $minPepLenInPepSummary, "", $flags2
);
```

## Scores and Significance

### Testing if a peptide match is significant

```perl
# These two tests are equivalent:
my $is_significant = $pep->getIonsScore() >= $summary->getPeptideThreshold($q, 20, $rank);
# same as:
my $is_significant = $pep->getExpectationValue() <= 0.05;
```

- `getIonsScore()` — Mascot ions score
- `getPeptideThreshold($query, 20, $rank)` — score threshold at p=0.05
- `getExpectationValue()` — E-value (significant if <= 0.05)
- The `20` argument is `1/p`, so 20 = p=0.05, 1000 = p=0.001

### Threshold methods

```perl
$summary->getPeptideIdentityThreshold($q, 20);      # Identity threshold at p=0.05
$summary->getHomologyThreshold($q, 20);              # Homology threshold at p=0.05
$summary->getPeptideThreshold($q, 20, $rank);        # Effective threshold for this PSM
$summary->getProbOfPepBeingRandomMatch($score, $q);  # Probability of random match
```

## Iterating Protein Hits

The `getHit()` method is on the summary object. Returns `undef` when exhausted.

```perl
my $hit = 1;
while (my $prot = $summary->getHit($hit)) {
    my $acc      = $prot->getAccession();
    my $score    = $prot->getScore();
    my $coverage = $prot->getCoverage();
    my $frame    = $prot->getFrame();
    my $db_idx   = $prot->getDB();

    # Description, mass are on the SUMMARY, not the protein:
    my $desc   = $summary->getProteinDescription($acc);
    my $mass   = $summary->getProteinMass($acc);

    $hit++;
}
```

**WRONG patterns (do NOT use):**
```perl
$prot->getDescription()              # WRONG - no such method
$prot->getMass()                     # WRONG - use $summary->getProteinMass($acc)
$prot->getLength()                   # WRONG - does not exist
$prot->getProteinDescription()       # WRONG - not on ms_protein
$summary->getProteinSequenceString() # WRONG - does not exist
$summary->getProteinLength()         # WRONG - does not exist
```

## Iterating Peptides for a Protein

```perl
for my $i (1 .. $prot->getNumPeptides()) {
    my $query = $prot->getPeptideQuery($i);
    my $rank  = $prot->getPeptideP($i);
    next if $query == -1 || $rank == -1;

    next if $prot->getPeptideDuplicate($i)
            == $msparser::ms_protein::DUPE_DuplicateSameQuery;

    my $pep = $summary->getPeptide($query, $rank);
    next unless defined $pep;

    # Peptide data
    my $seq       = $pep->getPeptideStr();
    my $score     = $pep->getIonsScore();
    my $expect    = $pep->getExpectationValue();
    my $mz_obs    = $pep->getObserved();
    my $mr_exp    = $pep->getMrExperimental();
    my $mr_calc   = $pep->getMrCalc();
    my $charge    = $pep->getCharge();
    my $delta     = $pep->getDelta();
    my $missed    = $pep->getMissedCleavages();
    my $var_mods  = $pep->getVarModsStr();

    # Significance test
    my $threshold   = $summary->getPeptideThreshold($query, 20, $rank);
    my $significant = $score >= $threshold;

    # Additional protein context
    my $is_bold = $prot->getPeptideIsBold($i);
    my $start   = $prot->getPeptideStart($i);
    my $end     = $prot->getPeptideEnd($i);
    my $prev_aa = $prot->getPeptideResidueBefore($i);
    my $next_aa = $prot->getPeptideResidueAfter($i);

    # Readable modifications
    my $readable = $summary->getReadableVarMods($query, $rank);
}
```

**WRONG patterns (do NOT use):**
```perl
$prot->getPeptideRank($i)   # WRONG - use getPeptideP($i)
$prot->getPeptide($i)       # WRONG - no such method on ms_protein
$pep->getIsDecoy()          # WRONG - no such method on ms_peptide
$pep->getIsBold()           # WRONG - use $prot->getPeptideIsBold($i)
```

## Input Query / Spectrum Data

```perl
my $inp = msparser::ms_inputquery->new($resfile, $query);
my $title     = $inp->getStringTitle(1);    # decode URL encoding
my $num_peaks = $inp->getNumberOfPeaks(1);  # peak list 1

for my $p (1 .. $num_peaks) {
    my $mz        = $inp->getPeakMass(1, $p);
    my $intensity = $inp->getPeakIntensity(1, $p);
}

my $rt         = $inp->getRetentionTimes();
my $scans      = $inp->getScanNumbers();
my $rawfile    = $inp->getRawfile();
my $ion_mob    = $inp->getIonMobility();
```

**WRONG patterns (do NOT use):**
```perl
$resfile->getQueryNum($query)     # WRONG - no such method
$resfile->getInputQuery($query)   # This exists but returns different object
# Always use: msparser::ms_inputquery->new($resfile, $query)
```

## Similar/Subset Proteins

```perl
my $i = 1;
while (my $sim_prot = $summary->getNextSimilarProteinOf($acc, $db_idx, $i)) {
    my $sim_acc = $sim_prot->getAccession();
    my $sim_score = $sim_prot->getScore();
    $i++;
}

my $j = 1;
while (my $sub_prot = $summary->getNextSubsetProteinOf($acc, $db_idx, $j)) {
    my $sub_acc = $sub_prot->getAccession();
    $j++;
}
```

**WRONG patterns (do NOT use):**
```perl
$prot->getNumberOfSimilarProteins()  # WRONG - use getSimilarProteins()
$prot->getSimilarProtein($i)         # WRONG - use getSimilarProteinName($i)
$prot->getProteinMember($i)          # WRONG - does not exist
$prot->getNumProteins()              # WRONG - does not exist
```

## Constants and Flags

All accessed via `$msparser::ClassName::CONSTANT` — dollar prefix, NO parens.

```perl
# Duplicate flags (on ms_protein)
$msparser::ms_protein::DUPE_DuplicateSameQuery
$msparser::ms_protein::DUPE_Duplicate
$msparser::ms_protein::DUPE_NotDuplicate

# Result flags (on ms_mascotresults)
$msparser::ms_mascotresults::MSRES_GROUP_PROTEINS
$msparser::ms_mascotresults::MSRES_SHOW_SUBSETS
$msparser::ms_mascotresults::MSRES_CLUSTER_PROTEINS
$msparser::ms_mascotresults::MSRES_MUDPIT_PROTEIN_SCORE
$msparser::ms_mascotresults::MSRES_REQUIRE_BOLD_RED
$msparser::ms_mascotresults::SCORE

# FDR count types (on ms_mascotresults)
$msparser::ms_mascotresults::DS_COUNT_PSM
$msparser::ms_mascotresults::DS_COUNT_SEQUENCE

# Peptide summary flags
$msparser::ms_peptidesummary::MSPEPSUM_USE_HOMOLOGY_THRESH
$msparser::ms_peptidesummary::MSPEPSUM_PERCOLATOR
```

## Quantitation (TMT/iTRAQ)

```perl
my $method = msparser::ms_quant_method->new();
$resfile->getQuantitationMethod($method);
my $protocol = uc($method->getProtocol()->getType());

my $quant = msparser::ms_ms2quantitation->new($summary, $method);

my $num_ratios = $quant->getNumberOfRatioNames();
for my $r (0 .. $num_ratios - 1) {
    my $ratio_name = $quant->getRatioName($r);
}

my $key = msparser::ms_peptide_quant_key->new($query, $rank);
my $ratio = $quant->getPeptideRatio($key, $ratio_name);
if ($ratio && $ratio->isValid()) {
    my $value = $ratio->getValue();
}

my $prot_ratio = $quant->getProteinRatio($acc, $db_idx, $ratio_name);
```

## Configuration (mascot.dat)

```perl
my $datfile = msparser::ms_datfile->new("<MASCOT_HOME>/config/mascot.dat");
my $options = $datfile->getMascotOptions();
my $dbs     = $datfile->getDatabases();

for my $i (0 .. $dbs->getNumberOfDatabases() - 1) {
    my $db = $dbs->getDatabase($i);
    my $name   = $db->getName();
    my $active = $db->isActive();
}
```

## Complete Working Pattern: Export All Significant PSMs

This is the canonical pattern for iterating all significant PSMs from a result
file using the modern two-argument constructor. Copy this when writing export modules.

```perl
use msparser;

my $resfile = msparser::ms_mascotresfilebase::createResfile($dat_path);
die $resfile->getLastErrorString() unless $resfile->isValid();

my $datfile = msparser::ms_datfile->new("<MASCOT_HOME>/config/mascot.dat");
my $options = $datfile->isValid() ? $datfile->getMascotOptions()
                                  : msparser::ms_mascotoptions->new();

my $resParams = msparser::ms_mascotresults_params->new();
$resfile->get_ms_mascotresults_params($options, $resParams);

# Optional: set target FDR for decoy searches
if ($resfile->params->getDECOY > 0) {
    $resParams->setTargetFDR(0.01);
    $resParams->setTargetFDRType($msparser::ms_mascotresults::DS_COUNT_PSM);
}

my $summary = msparser::ms_peptidesummary->new($resfile, $resParams);

my $hit = 1;
while (my $prot = $summary->getHit($hit)) {
    my $acc  = $prot->getAccession();
    my $desc = $summary->getProteinDescription($acc);
    my $mass = $summary->getProteinMass($acc);

    for my $i (1 .. $prot->getNumPeptides()) {
        my $query = $prot->getPeptideQuery($i);
        my $rank  = $prot->getPeptideP($i);
        next if $query == -1 || $rank == -1;
        next if $prot->getPeptideDuplicate($i)
                == $msparser::ms_protein::DUPE_DuplicateSameQuery;

        my $pep = $summary->getPeptide($query, $rank);
        next unless defined $pep;

        my $score     = $pep->getIonsScore();
        my $threshold = $summary->getPeptideThreshold($query, 20, $rank);
        next unless $score >= $threshold;  # Only significant PSMs

        my $seq    = $pep->getPeptideStr();
        my $expect = $pep->getExpectationValue();
        my $mz     = $pep->getObserved();
        my $charge = $pep->getCharge();
        my $delta  = $pep->getDelta();
        my $missed = $pep->getMissedCleavages();

        my $inp   = msparser::ms_inputquery->new($resfile, $query);
        my $title = $inp->getStringTitle(1);

        my $bold  = $prot->getPeptideIsBold($i);
        my $start = $prot->getPeptideStart($i);
        my $end   = $prot->getPeptideEnd($i);

        # ... output or collect results
    }
    $hit++;
}
```
