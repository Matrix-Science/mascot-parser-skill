# msparser Common Recipes

Ready-to-use code patterns. All recipes assume msparser is imported and available.

```python
import sys
sys.path.insert(0, "<MSPARSER_SDK>/python36_and_later")
import msparser
```

---

## 1. Open a Result File and Get Basic Info

```python
def get_result_info(filepath):
    """Open a result file and print basic search information."""
    resfile = msparser.ms_mascotresfilebase.createResfile(filepath)
    if not resfile.isValid():
        print(f"Error: {resfile.getLastErrorString()}")
        return None

    import time
    info = {
        "queries": resfile.getNumQueries(),
        "sequences": resfile.getNumSeqs(),
        "sequences_after_tax": resfile.getNumSeqsAfterTax(),
        "residues": resfile.getNumResidues(),
        "exec_time": resfile.getExecTime(),
        "date": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(resfile.getDate())),
        "mascot_version": resfile.getMascotVer(),
        "fasta_version": resfile.getFastaVer(),
        "is_msms": resfile.anyMSMS(),
        "is_pmf": resfile.anyPMF(),
        "is_sq": resfile.anySQ(),
        "has_matches": resfile.anyFastaMatches(),
        "is_decoy": resfile.params().getDECOY() > 0,
    }
    return info
```

---

## 2. Get Search Parameters

```python
def get_search_params(resfile):
    """Extract search parameters as a dictionary."""
    p = resfile.params()
    params = {
        "title": p.getCOM(),
        "database": p.getDB(),
        "enzyme": p.getCLE(),
        "missed_cleavages": p.getPFA(),
        "fixed_mods": p.getMODS(),
        "variable_mods": p.getIT_MODS(),
        "peptide_tol": f"{p.getTOL()} {p.getTOLU()}",
        "fragment_tol": f"{p.getITOL()} {p.getITOLU()}",
        "mass_type": p.getMASS(),
        "charge": p.getCHARGE(),
        "instrument": p.getINSTRUMENT(),
        "search_type": p.getSEARCH(),
        "taxonomy": p.getTAXONOMY(),
        "username": p.getUSERNAME(),
        "filename": p.getFILENAME(),
        "decoy": p.getDECOY() > 0,
    }
    return params
```

---

## 3. Create a Summary (Standard Pattern)

Uses the recommended two-argument constructor with `ms_mascotresults_params`.

```python
def create_summary(resfile, mascot_dat_path=None, target_fdr=None):
    """Create a peptide or protein summary using the recommended two-argument constructor.

    Args:
        resfile: ms_mascotresfilebase object
        mascot_dat_path: path to mascot.dat (optional, uses defaults if unavailable)
        target_fdr: target FDR as a decimal (e.g. 0.01 for 1%), or None
    """
    mascotOptions = msparser.ms_mascotoptions()
    if mascot_dat_path:
        datfile = msparser.ms_datfile(mascot_dat_path)
        if datfile.isValid():
            mascotOptions = datfile.getMascotOptions()

    resParams = msparser.ms_mascotresults_params()
    resfile.get_ms_mascotresults_params(mascotOptions, resParams)

    if target_fdr is not None and resfile.params().getDECOY() > 0:
        resParams.setTargetFDR(target_fdr)
        resParams.setTargetFDRType(msparser.ms_mascotresults.DS_COUNT_PSM)

    if resParams.isUsePeptideSummary():
        results = msparser.ms_peptidesummary(resfile, resParams)
    else:
        results = msparser.ms_proteinsummary(resfile, resParams)

    return results
```

---

## 4. Get Protein Hits with Scores

```python
def get_protein_hits(resfile, mascot_dat_path=None, max_hits=50):
    """Get top protein hits with accession, description, score, mass, coverage."""
    results = create_summary(resfile, mascot_dat_path)

    proteins = []
    hit = 1
    prot = results.getHit(hit)
    while prot and (max_hits == 0 or hit <= max_hits):
        acc = prot.getAccession()
        proteins.append({
            "hit": hit,
            "accession": acc,
            "description": results.getProteinDescription(acc),
            "score": prot.getScore(),
            "mass": results.getProteinMass(acc),
            "coverage": prot.getCoverage(),
            "num_peptides": prot.getNumDisplayPeptides(),
            "rms_error": prot.getRMSDeltas(results),
        })
        hit += 1
        prot = results.getHit(hit)

    return proteins, results
```

---

## 5. Get Peptide Matches for a Protein (with Significance)

```python
def get_protein_peptides(prot, results):
    """Get all peptide matches for a given protein hit, with significance info."""
    peptides = []
    for i in range(1, 1 + prot.getNumPeptides()):
        query = prot.getPeptideQuery(i)
        rank = prot.getPeptideP(i)
        if query == -1 or rank == -1:
            continue
        if prot.getPeptideDuplicate(i) == msparser.ms_protein.DUPE_DuplicateSameQuery:
            continue

        pep = results.getPeptide(query, rank)
        if not pep or not pep.getAnyMatch():
            continue

        score = pep.getIonsScore()
        threshold = results.getPeptideThreshold(query, 20, rank)

        peptides.append({
            "query": query,
            "rank": rank,
            "sequence": pep.getPeptideStr(),
            "score": score,
            "expect": pep.getExpectationValue(),
            "threshold": threshold,
            "significant": score >= threshold,
            "observed_mz": pep.getObserved(),
            "charge": pep.getCharge(),
            "mr_calc": pep.getMrCalc(),
            "mr_exp": pep.getMrExperimental(),
            "delta": pep.getDelta(),
            "missed_cleavages": pep.getMissedCleavages(),
            "ions_matched": pep.getNumIonsMatched(),
            "var_mods": pep.getVarModsStr(),
            "readable_mods": results.getReadableVarMods(query, rank),
            "is_bold": prot.getPeptideIsBold(i),
            "is_duplicate": prot.getPeptideDuplicate(i) == msparser.ms_protein.DUPE_Duplicate,
        })
    return peptides
```

---

## 6. Export Results to CSV

```python
import csv

def export_proteins_csv(resfile, output_path, mascot_dat_path=None):
    """Export protein hits to CSV."""
    proteins, results = get_protein_hits(resfile, mascot_dat_path)

    with open(output_path, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=[
            "hit", "accession", "description", "score", "mass",
            "coverage", "num_peptides", "rms_error"
        ])
        writer.writeheader()
        writer.writerows(proteins)


def export_peptides_csv(resfile, output_path, mascot_dat_path=None):
    """Export all peptides from all protein hits to CSV."""
    results = create_summary(resfile, mascot_dat_path)

    with open(output_path, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=[
            "protein_accession", "query", "rank", "sequence", "score",
            "expect", "significant", "observed_mz", "charge", "mr_calc",
            "delta", "missed_cleavages", "readable_mods"
        ])
        writer.writeheader()

        hit = 1
        prot = results.getHit(hit)
        while prot:
            acc = prot.getAccession()
            peptides = get_protein_peptides(prot, results)
            for pep in peptides:
                row = {"protein_accession": acc}
                row.update({k: pep[k] for k in [
                    "query", "rank", "sequence", "score", "expect",
                    "significant", "observed_mz", "charge", "mr_calc",
                    "delta", "missed_cleavages", "readable_mods"
                ]})
                writer.writerow(row)
            hit += 1
            prot = results.getHit(hit)
```

---

## 7. List Available Databases from mascot.dat

```python
def list_databases(mascot_dat_path):
    """List all configured databases and their status."""
    datfile = msparser.ms_datfile(mascot_dat_path)
    if not datfile.isValid():
        print(f"Error: {datfile.getLastErrorString()}")
        return []

    dbs = datfile.getDatabases()
    if not dbs.isSectionAvailable():
        return []

    databases = []
    for i in range(dbs.getNumberOfDatabases()):
        db = dbs.getDatabase(i)
        databases.append({
            "name": db.getName(),
            "active": db.isActive(),
        })
    return databases
```

---

## 8. Connect to Server and Authenticate

Credentials should be loaded from `.env` or environment variables.

```python
from dotenv import load_dotenv
import os
load_dotenv()

def connect_to_mascot():
    """Connect to Mascot Server and authenticate."""
    url = os.getenv("MASCOT_URL", "http://localhost/mascot/cgi/")
    username = os.getenv("MASCOT_USER", "")
    password = os.getenv("MASCOT_PASS", "")

    settings = msparser.ms_connection_settings()
    settings.setProxyServerType(msparser.ms_connection_settings.PROXY_TYPE_NO_PROXY)
    settings.setUserAgent("PythonScript/1.0 " + settings.getUserAgent())

    client = msparser.ms_http_client(url, settings)
    if not client.isValid():
        print(f"Connection failed: {client.getLastErrorString()}")
        return None, None

    session = msparser.ms_http_client_session()
    rc = client.userLogin(username, password, session)

    if rc == msparser.ms_http_client.L_SUCCESS:
        print(f"Logged in. Session: {session.sessionId()}")
    elif rc == msparser.ms_http_client.L_SECURITYDISABLED:
        print("Security disabled on server, using default session")
    else:
        print(f"Login failed with code {rc}")
        return client, None

    return client, session
```

---

## 9. Submit a Search via HTTP

```python
import time

def submit_search(session, mgf_path, params=None):
    """Submit an MS/MS search to Mascot Server.

    params dict can override defaults: DB, CLE, MODS, IT_MODS, TOL, TOLU, ITOL, ITOLU, etc.
    """
    defaults = {
        "SEARCH": "MIS",
        "DB": "SwissProt",
        "CLE": "Trypsin/P",
        "PFA": "1",
        "QUANTITATION": "None",
        "TAXONOMY": "All entries",
        "MODS": "Carbamidomethyl (C)",
        "IT_MODS": "Oxidation (M)",
        "TOL": "10",
        "TOLU": "ppm",
        "PEP_ISOTOPE_ERROR": "1",
        "ITOL": "0.1",
        "ITOLU": "Da",
        "CHARGE": "2+",
        "MASS": "Monoisotopic",
        "FORMAT": "Mascot generic",
        "INSTRUMENT": "ESI-TRAP",
        "REPORT": "AUTO",
        "DECOY": "1",
        "USERNAME": "Mascot Parser Script",
        "USEREMAIL": "",
    }
    if params:
        defaults.update(params)

    boundary = "---------FormBoundary4C9ByKKVofH"
    httpHeader = f"Content-Type: multipart/mixed; boundary={boundary}"

    parts = [f"FORMVER\n\n1.01"]
    for key, val in defaults.items():
        parts.append(f"{key}\n\n{val}")

    prologue = ""
    for part in parts:
        prologue += f"-----------{boundary}\nContent-Disposition: form-data; name=\"{part.split(chr(10))[0]}\"\n{chr(10).join(part.split(chr(10))[1:])}\n"

    import os
    filename = os.path.basename(mgf_path)
    prologue += f'-----------{boundary}\nContent-Disposition: form-data; name="FILE"; filename="{filename}"\n'
    epilogue = f"-----------{boundary}--"

    search = msparser.ms_http_client_search(session, "")
    if not search.isValid():
        print(f"Search object error: {search.getLastErrorString()}")
        return None

    progress = msparser.ms_http_helper_progress()
    if not session.submitSearch(search, httpHeader, prologue, mgf_path, epilogue, progress):
        print(f"Submit failed: {session.getLastErrorString()}")
        return None

    print(f"Search submitted. Task ID: {search.searchTaskId()}")
    return search
```

---

## 10. Check Search Status and Download Results

```python
def wait_for_search(search, timeout=600):
    """Poll search status until complete. Returns local filename or None."""
    import time
    start = time.time()
    while time.time() - start < timeout:
        success, code, value = search.getStatus()
        if not success:
            print("Status check failed")
            return None

        if code == msparser.ms_http_client_search.SS_COMPLETE:
            print("Search complete!")
            break
        elif code == msparser.ms_http_client_search.SS_ERROR:
            print(f"Search error: {value}")
            return None
        elif code == msparser.ms_http_client_search.SS_RUNNING:
            print(f"Running... {value}% complete")
        elif code == msparser.ms_http_client_search.SS_QUEUED:
            print("Queued...")

        time.sleep(2)
    else:
        print("Timeout waiting for search")
        return None

    ok, remote_file = search.getResultsFileName()
    if not ok or not remote_file:
        print("Could not get results filename")
        return None

    import os
    local_file = os.path.basename(remote_file)
    progress = msparser.ms_http_helper_progress()
    if search.downloadResultsFile(local_file, progress):
        print(f"Downloaded to {local_file}")
        return local_file
    else:
        print("Download failed")
        return None
```

---

## 11. Find Recent Result Files

```python
import os
import glob

def find_result_files(data_dir, extensions=(".dat", ".msr")):
    """Find result files in the Mascot data directory, sorted by modification time."""
    files = []
    for ext in extensions:
        files.extend(glob.glob(os.path.join(data_dir, f"*{ext}")))
        files.extend(glob.glob(os.path.join(data_dir, f"*/*{ext}")))

    files.sort(key=os.path.getmtime, reverse=True)

    results = []
    for f in files:
        results.append({
            "path": f,
            "name": os.path.basename(f),
            "size_mb": os.path.getsize(f) / (1024 * 1024),
            "modified": os.path.getmtime(f),
        })
    return results
```

---

## 12. FDR-Controlled Results with Significance

```python
def get_significant_peptides(resfile, mascot_dat_path=None, target_fdr=0.01):
    """Get all significant peptide matches at a given FDR threshold."""
    results = create_summary(resfile, mascot_dat_path, target_fdr=target_fdr)

    significant = []
    hit = 1
    prot = results.getHit(hit)
    while prot:
        acc = prot.getAccession()
        for i in range(1, 1 + prot.getNumPeptides()):
            query = prot.getPeptideQuery(i)
            rank = prot.getPeptideP(i)
            if query == -1 or rank == -1:
                continue
            if prot.getPeptideDuplicate(i) == msparser.ms_protein.DUPE_DuplicateSameQuery:
                continue

            pep = results.getPeptide(query, rank)
            if not pep or not pep.getAnyMatch():
                continue

            score = pep.getIonsScore()
            threshold = results.getPeptideThreshold(query, 20, rank)

            if score >= threshold:
                significant.append({
                    "protein": acc,
                    "query": query,
                    "rank": rank,
                    "sequence": pep.getPeptideStr(),
                    "score": score,
                    "expect": pep.getExpectationValue(),
                    "threshold": threshold,
                })

        hit += 1
        prot = results.getHit(hit)

    return significant
```

---

## 13. Enable Percolator Rescoring

```python
def create_summary_with_percolator(resfile, mascot_dat_path=None, target_fdr=0.01):
    """Create a summary with Percolator rescoring enabled (if available)."""
    mascotOptions = msparser.ms_mascotoptions()
    if mascot_dat_path:
        datfile = msparser.ms_datfile(mascot_dat_path)
        if datfile.isValid():
            mascotOptions = datfile.getMascotOptions()

    resParams = msparser.ms_mascotresults_params()
    resfile.get_ms_mascotresults_params(mascotOptions, resParams)

    # Check Percolator prerequisites
    can_percolate = (
        resfile.params().getDECOY() > 0
        and resfile.anyMSMS()
        and not resfile.isErrorTolerant()
    )

    if can_percolate:
        flags2 = resParams.getFlags2()
        flags2 |= msparser.ms_peptidesummary.MSPEPSUM_PERCOLATOR
        resParams.setFlags2(flags2)

    if target_fdr is not None and resfile.params().getDECOY() > 0:
        resParams.setTargetFDR(target_fdr)
        resParams.setTargetFDRType(msparser.ms_mascotresults.DS_COUNT_PSM)

    results = msparser.ms_peptidesummary(resfile, resParams)
    return results
```

---

## 14. Get Unassigned Peptide List

```python
def get_unassigned_peptides(results):
    """Get peptides that didn't match any protein above threshold."""
    results.createUnassignedList(msparser.ms_mascotresults.SCORE)
    unassigned = []
    for u in range(1, 1 + results.getNumberOfUnassigned()):
        pep = results.getUnassigned(u)
        if pep.getPeptideStr():
            unassigned.append({
                "query": pep.getQuery(),
                "sequence": pep.getPeptideStr(),
                "score": pep.getIonsScore(),
                "expect": pep.getExpectationValue(),
                "observed_mz": pep.getObserved(),
                "charge": pep.getCharge(),
                "readable_mods": results.getReadableVarMods(pep.getQuery(), pep.getRank()),
            })
    return unassigned
```

---

## 15. Read Spectrum Data from Result File

```python
def get_spectrum_data(resfile, query_number):
    """Extract spectrum peak list for a given query."""
    inp = msparser.ms_inputquery(resfile, query_number)

    spectrum = {
        "query": query_number,
        "title": inp.getStringTitle(True),
        "pepmass": resfile.getObservedMass(query_number),
        "charge": resfile.getObservedCharge(query_number),
        "num_peaks": inp.getNumberOfPeaks(1),
        "peaks": [],
    }

    for i in range(1, 1 + inp.getNumberOfPeaks(1)):
        spectrum["peaks"].append({
            "mz": inp.getPeakMass(1, i),
            "intensity": inp.getPeakIntensity(1, i),
        })

    return spectrum


def export_mgf(resfile, output_path):
    """Export all queries from a result file to MGF format."""
    with open(output_path, "w") as f:
        for q in range(1, 1 + resfile.getNumQueries()):
            inp = msparser.ms_inputquery(resfile, q)

            if inp.getNumberOfPeaks(1) == 0:
                f.write(f"{resfile.getObservedMass(q)}\n")
                continue

            f.write("BEGIN IONS\n")
            f.write(f"PEPMASS={resfile.getObservedMass(q)}\n")

            charge = resfile.getObservedCharge(q)
            if charge > 0:
                f.write(f"CHARGE={charge}+\n")

            title = inp.getStringTitle(True)
            if title:
                f.write(f"TITLE={title}\n")

            for i in range(1, 1 + inp.getNumberOfPeaks(1)):
                f.write(f"{inp.getPeakMass(1, i)} {inp.getPeakIntensity(1, i)}\n")

            f.write("END IONS\n\n")
```

---

## 18. CRITICAL: chdir to Mascot CGI dir before opening result files

`msparser` resolves the unimod XML schema (`unimod_2.xsd`) via a *relative path*
`../html/xmlns/schema/unimod_2/unimod_2.xsd`. If your CWD is anywhere outside
`<MASCOT_HOME>/cgi/`, the schema fails to load — `getLastErrorString()` reports
"Failed to load unimod xml file" and `pep.getPeptideStr()` returns empty
strings on every peptide (silent failure, no exception). PSM counts via
`getNumberOfHits()` come back as 0.

**Always:**
```python
import os
os.chdir(r"C:\inetpub\mascot\cgi")  # Linux: /usr/local/mascot/cgi
resfile = msparser.ms_mascotresfilebase.createResfile(path)
```

This is not documented in the SDK guide but is required by every script that
opens local result files. The QC dashboard (`mascot_qc_report.py`) does this
explicitly; copy that pattern.

---

## 19. Get PSM/sequence counts at a target FDR (the right way)

The naive call `getNumHitsAboveIdentity(0.05, ...)` returns counts at the *raw*
identity threshold (p ≤ 0.05). To match the CSV-export-header convention
(counts at 1 % FDR-adjusted threshold) use `getThresholdForFDRAboveIdentity`,
which returns a **list of 5 values** `[ok, achieved_FDR, sigLevel, n_target, n_decoy]`:

```python
M = msparser.ms_mascotresults
target_fdr = 0.01

# 3-arg overload — db_match_type ONLY accepts DS_IDENTITY (DS_HOMOLOGY returns
# [False, -1, -1, -1, -1]).
def fdr_count(method, count_type):
    r = method(target_fdr, count_type, M.DS_IDENTITY)
    if not r or len(r) < 5 or not r[0]:
        return 0, 0, 0.0
    return int(r[3]), int(r[4]), float(r[1])  # target, decoy, achieved_fdr

n_psm_target,  n_psm_decoy,  fdr_psm = fdr_count(
    results.getThresholdForFDRAboveIdentity, M.DS_COUNT_PSM)
n_seq_target,  n_seq_decoy,  fdr_seq = fdr_count(
    results.getThresholdForFDRAboveIdentity, M.DS_COUNT_SEQUENCE)
n_psm_target_h, n_psm_decoy_h, _ = fdr_count(
    results.getThresholdForFDRAboveHomology, M.DS_COUNT_PSM)
```

Constants: `DS_COUNT_PSM = 0`, `DS_COUNT_SEQUENCE = 1`, `DS_IDENTITY = 0`,
`DS_HOMOLOGY = 1` (the latter is used as a `db_match_type` by some methods but
**not** by `getThresholdForFDR*` — pass `DS_IDENTITY` there).

Why this matters: `setTargetFDR(0.01)` only affects the *threshold function used
during summary creation*. The count APIs always take an explicit sig_level,
and the right one for "1 % FDR" is what `getThresholdForFDR*` returns — not 0.05.

**There is no `pep.isDecoy()` method.** Don't try to detect decoys by walking
peptides and checking accession prefixes; use the API above, which classifies
hits via the result file's embedded decoy flag.

---

## 20. Read Percolator-rescored results: stage cache files in a tmp dir

Setting `MSPEPSUM_PERCOLATOR` on a result-params object alone is **not enough**
— `ms_peptidesummary(resfile, params)` will create with `getNumberOfHits() = 0`
and any `getPeptide()` call will raise
"Attempting to call function ... before createSummary() has completed".

The hash msparser computes locally on `mascot.dat` may differ from the hash
the server used when it wrote the `.target.pop` / `.decoy.pop` / `.pip` files
in `<MASCOT_HOME>/data/cache/YYYY/MM/<server_hash>/`. The fix (mirrors what
the QC dashboard does):

1. Open the resfile pointed at a **fresh tmp dir** as cache_dir.
2. Call `setPercolatorFeatures` with the same `ml_adapter_param` values that
   the search used (e.g. `MS2Rescore.ms2pip_model=HCD2019`).
3. Read the *expected* filenames via `getPercolatorFileNames()` — returns
   tuple `(pip, target.pop, decoy.pop)`.
4. Copy the real server cache files to the expected paths in the tmp dir.
5. Set `MSPEPSUM_PERCOLATOR` and create the summary.

```python
import os, glob, shutil, tempfile
from pathlib import Path

def open_with_percolator(res_path, adapter_params):
    os.chdir(r"C:\inetpub\mascot\cgi")  # see recipe 18

    # Find server-written cache for this resfile
    res_name = os.path.basename(res_path)
    real_target = glob.glob(rf"C:\inetpub\mascot\data\cache\*\*\*\{res_name}.*.target.pop")
    if not real_target:
        return None  # search wasn't Percolator-rescored
    real_dir = os.path.dirname(real_target[0])
    final_hash = os.path.basename(real_target[0]).rsplit('.target.pop', 1)[0].split('.')[-1]
    real_decoy = os.path.join(real_dir, f"{res_name}.{final_hash}.decoy.pop")
    real_pip   = os.path.join(real_dir, f"{res_name}.{final_hash}.pip")

    # Open into tmp cache dir, set features, get expected filenames
    tmp = tempfile.mkdtemp(prefix="perc_cache_")
    df = msparser.ms_datfile(r"C:\inetpub\mascot\config\mascot.dat")
    opts = df.getMascotOptions()
    rf = msparser.ms_mascotresfilebase.createResfile(res_path, 0, "", 0, tmp)
    opts.setPercolatorExeFlags(opts.getPercolatorRtFlags(rf.hasRT(), opts.isPercolatorUseRT()))
    vs = msparser.VectorString()
    for p in adapter_params: vs.append(p)
    rf.setPercolatorFeatures(opts, "", vs)

    expected_pip, expected_target, expected_decoy = rf.getPercolatorFileNames()
    for src, dst in [(real_pip, expected_pip),
                     (real_target, expected_target),
                     (real_decoy, expected_decoy)]:
        Path(dst).parent.mkdir(parents=True, exist_ok=True)
        shutil.copyfile(src, dst)

    # Now build params and summary
    rp = msparser.ms_mascotresults_params()
    rf.get_ms_mascotresults_params(opts, rp)
    flags2 = rp.getFlags2() | msparser.ms_peptidesummary.MSPEPSUM_PERCOLATOR
    rp.setFlags2(flags2)
    rp.setTargetFDR(0.01)
    rp.setTargetFDRType(msparser.ms_mascotresults.DS_COUNT_SEQUENCE)
    return msparser.ms_peptidesummary(rf, rp), rf
```

After this, `rf.getPercolatorFileNames()` returns a 3-tuple
`(pip, target.pop, decoy.pop)` (note pip is FIRST, not last). The QC
dashboard's `stage_percolator_cache` follows exactly this pattern.

## 21. Read the `.target.pop` directly (FDR-passing PSMs, family-member-safe)

When you only need the **FDR-passing PSMs** (and per-protein roll-ups built from
them), you can skip the `ms_peptidesummary` staging dance of recipe 20 entirely
and read the server's `.target.pop` table directly. This is the most robust path
for per-protein identification/quant because grouping by the `proteinIds` column
captures **representatives AND family members** automatically — it sidesteps the
`getHit()`-returns-representatives-only blind spot (see SKILL.md → *Reading all
proteins: family members, not just `getHit()`*).

`.target.pop` is tab-separated with a header row. Key columns: `PSMId`
(`query:N;rank:M` — **N is the msparser query number**), `score`, `q-value`,
`posterior_error_prob`, `peptide` (`X.SEQUENCE.X`), `proteinIds` (space-separated
accessions). msparser is still used — but only for **spectrum/peak access** via
`ms_inputquery`, which is reliable on the SQLite `.msr`.

```python
import os, glob, re
import msparser

PSMID_RE = re.compile(r"query:(\d+);rank:(\d+)")

def find_target_pop(cache_root, res_name):
    hits = glob.glob(os.path.join(cache_root, "*", "*", "*", f"{res_name}.*.target.pop"))
    return hits[0] if hits else None

def proteins_from_target_pop(res_path, cache_root, target_fdr=0.01):
    """Group FDR-passing PSMs by protein — reps AND family members included."""
    pop = find_target_pop(cache_root, os.path.basename(res_path))
    if not pop:
        return None  # search wasn't Percolator-rescored
    resfile = msparser.ms_mascotresfilebase.createResfile(res_path)  # peaks only

    by_protein = {}  # acc -> {"psms": int, "seqs": set, "queries": set}
    with open(pop, encoding="utf-8") as fh:
        next(fh)  # header
        for line in fh:
            f = line.rstrip("\n").split("\t")
            if len(f) < 6:
                continue
            try:
                q = float(f[2])            # q-value column
            except ValueError:
                continue
            if q > target_fdr:
                continue
            m = PSMID_RE.search(f[0])       # PSMId -> query N
            if not m:
                continue
            query = int(m.group(1))         # use with ms_inputquery(resfile, query)
            pepfield = f[4]                 # X.SEQ.X
            seq = pepfield.split(".")[1] if pepfield.count(".") >= 2 else pepfield
            for acc in f[5].split():        # proteinIds, space-separated
                d = by_protein.setdefault(acc, {"psms": 0, "seqs": set(), "queries": set()})
                d["psms"] += 1
                d["seqs"].add(seq)
                d["queries"].add(query)
    return by_protein
```

For TMT/iTRAQ quant, read each passing PSM's spectrum with
`ms_inputquery(resfile, query)`, sum the reporter-ion intensities (nearest peak
within tolerance per channel), and add them to every protein in `proteinIds`. A
complete working extractor is in the project at
`TPP2D/scripts/msr_to_quant_summary_pop.py`.
