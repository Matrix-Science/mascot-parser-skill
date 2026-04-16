# Obsolete API Usage in SDK Example Scripts

The msparser SDK ships with example scripts that demonstrate common operations. Several of these use the **obsolete multi-argument `ms_peptidesummary` constructor** instead of the recommended two-argument constructor with `ms_mascotresults_params`.

## Affected Example Scripts

All four language versions of `resfile_summary` use the obsolete pattern:

| Script | Language | Issue |
|--------|----------|-------|
| `example_python/resfile_summary.py` | Python | Uses 9-argument `ms_peptidesummary()` constructor |
| `example_perl/resfile_summary.pl` | Perl | Uses 9-argument `msparser::ms_peptidesummary->new()` constructor |
| `example_java/resfile_summary.java` | Java | Uses 9-argument `new ms_peptidesummary()` constructor |
| `example_csharp/resfile_summary.cs` | C# | Uses 8-argument `new ms_peptidesummary()` constructor |

### Obsolete pattern (used in examples)

```python
# Python — DO NOT COPY
(scriptName, flags, minProbability, maxHitsToReport,
 ignoreIonsScoreBelow, minPepLenInPepSummary,
 usePeptideSummary, flags2) = resfile.get_ms_mascotresults_params(mascotOptions)

results = msparser.ms_peptidesummary(
    resfile, flags, minProbability, maxHitsToReport,
    "", ignoreIonsScoreBelow, minPepLenInPepSummary, "", flags2
)
```

### Recommended pattern (use this instead)

```python
# Python — USE THIS
resParams = msparser.ms_mascotresults_params()
resfile.get_ms_mascotresults_params(mascotOptions, resParams)

# Optional: override settings
resParams.setTargetFDR(0.01)

results = msparser.ms_peptidesummary(resfile, resParams)
```

## What else is fine in the examples

The remaining logic in these example scripts is correct and can be used as reference:
- Opening result files with `createResfile()`
- Iterating protein hits with `getHit()`
- Iterating peptides with `getPeptideQuery()` / `getPeptideP()`
- Duplicate checking with `getPeptideDuplicate()`
- Similar/subset protein enumeration
- Unassigned peptide listing
- Error checking pattern

## Production reference

The Mascot Server CGI scripts at `<MASCOT_HOME>/cgi/` (e.g. `common_subs.pl`, `export_dat_v3.pl`) and the library modules at `<MASCOT_HOME>/perl64/site/lib/PeptideSummary/Util.pm` use the modern two-argument constructor pattern and serve as the authoritative reference for correct API usage.
