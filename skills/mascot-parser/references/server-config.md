# Mascot Server Configuration

## Directory Layout

Mascot Server can be installed on Windows or Linux. The directory structure is the same; only the root path differs.

| Resource | Windows (typical) | Linux (typical) |
|----------|-------------------|-----------------|
| Mascot root (`<MASCOT_HOME>`) | `C:\inetpub\mascot\` | `/usr/local/mascot/` |
| CGI URL | `http://<HOST>/mascot/cgi/` | `http://<HOST>/mascot/cgi/` |

```
<MASCOT_HOME>/
├── bin/           # Mascot executables and tools
├── cgi/           # CGI scripts (nph-mascot.exe, master_results_2.pl, login.pl, etc.)
├── config/        # Configuration files (mascot.dat and versioned backups)
├── data/          # Result files (.dat, .msr) and date subdirectories
├── logs/          # Log files
├── sequence/      # FASTA sequence databases
└── x-cgi/         # Additional CGI scripts
```

## Key Paths

| Resource | Path |
|----------|------|
| mascot.dat (current) | `<MASCOT_HOME>/config/mascot.dat` |
| Result files | `<MASCOT_HOME>/data/` |
| Sequence databases | `<MASCOT_HOME>/sequence/` |
| msparser SDK | `<MSPARSER_SDK>/` (separate from Mascot Server) |
| Example scripts | `<MSPARSER_SDK>/example_python/`, `example_perl/`, etc. |

## Result File Organization

Result files live in `<MASCOT_HOME>/data/`:
- `.dat` files: text-based result format (e.g. `F001316.dat`, `F981139.dat`)
- `.msr` files: binary result format (e.g. `F981142.msr`, `F981143.msr`)
- Named result files (e.g. `benchmark_PXD003791.dat`)

Date subdirectories (e.g. `20240104/`, `20240306/`) contain `task_id` files tracking search provenance.

Special files in data directory:
- `mascot.control` - server control state
- `mascot.job` - job queue
- `getseq.job` - sequence retrieval queue

## Discovering Available Databases

```python
datfile = msparser.ms_datfile("<MASCOT_HOME>/config/mascot.dat")
dbs = datfile.getDatabases()
for i in range(dbs.getNumberOfDatabases()):
    db = dbs.getDatabase(i)
    if db.isActive():
        print(db.getName())
```

Or check the filesystem: each subdirectory under `<MASCOT_HOME>/sequence/` is a database.

## mascot.dat Configuration

The `mascot.dat` file is the central configuration. It has versioned backups in the config directory (e.g. `2024-11-26_113839.mascot.dat`).

Key sections accessible via `ms_datfile`:
- **Databases** (`getDatabases()`) - configured FASTA databases
- **Options** (`getMascotOptions()`) - server options including `getMascotCmdLine()`
- **Parse** (`getParseOptions()`) - accession/description parse rules
- **WWW** (`getWWWOptions()`) - sequence report URL patterns
- **Taxonomy** (`getTaxonomyRules(n)`) - taxonomy configurations
- **Cluster** (`getClusterParams()`) - cluster mode settings
- **Processors** (`getProcessors()`) - CPU configuration
- **Cron** (`getCronOptions()`) - scheduled tasks

## Authentication Flow

**Credentials** should be stored in environment variables or a `.env` file. Never hardcode passwords in scripts.

| Variable | Purpose |
|----------|---------|
| `MASCOT_URL` | Server CGI URL (e.g. `http://localhost/mascot/cgi/`) |
| `MASCOT_USER` | Username |
| `MASCOT_PASS` | Password |

### Direct File Access (No Auth Needed)
Opening local `.dat`/`.msr` files with `createResfile()` does not require authentication:
```python
resfile = msparser.ms_mascotresfilebase.createResfile("<MASCOT_HOME>/data/F001316.dat")
```

### HTTP Access (Auth Required)
For remote operations (search submission, downloading results, getting sequences):

```python
from dotenv import load_dotenv
import os
load_dotenv()

settings = msparser.ms_connection_settings()
settings.setProxyServerType(msparser.ms_connection_settings.PROXY_TYPE_NO_PROXY)

client = msparser.ms_http_client(os.getenv("MASCOT_URL"), settings)
session = msparser.ms_http_client_session()
rc = client.userLogin(os.getenv("MASCOT_USER"), os.getenv("MASCOT_PASS"), session)

if rc == msparser.ms_http_client.L_SUCCESS:
    # Authenticated - can submit searches, download results, etc.
    pass
elif rc == msparser.ms_http_client.L_SECURITYDISABLED:
    # Security not active - session still valid for operations
    pass

# Use session for operations...

# Always logout when done
session.logout()
```

### Reading mascot.dat via URL (Auth Required)
```python
cs = msparser.ms_connection_settings()
cs.setSessionID(session.sessionId())
datfile = msparser.ms_datfile(os.getenv("MASCOT_URL"), 0, cs)
```

## CGI Scripts

Key CGI endpoints at `http://<HOST>/mascot/cgi/`:
- `nph-mascot.exe` - search engine
- `master_results_2.pl` - results viewer (e.g. `master_results_2.pl?file=../data/F001316.dat`)
- `login.pl` - authentication
- `export_dat_2.pl` / `export_dat_v3.pl` - export results
- `get_params.pl` - get search parameters
- `getseq.pl` - sequence retrieval
