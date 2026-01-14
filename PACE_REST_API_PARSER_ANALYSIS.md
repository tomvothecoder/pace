# PACE REST API → Parser Call Flow Analysis

**Date:** 2026-01-14 (Updated: 2026-01-14)  
**Purpose:** Reference document for SimBoard to understand PACE's REST-to-parser flow for cron-driven ingestion  
**Status:** ANALYSIS ONLY - No code modifications  
**Revision:** 2 - Incorporated corrected understanding of E3SM parsing entrypoints, filesystem layout, and field-level metadata derivation

---

## Executive Summary

This document analyzes the call flow between PACE's REST APIs and the E3SM parsing logic. PACE uses a two-stage ingestion model:
1. **Upload Stage** (`/upload`): Accepts zip files, validates, and stores them to a filesystem staging area
2. **Parse Stage** (`/fileparse`): Triggers parser orchestration to extract metadata and persist to database

The flow is designed for **automated, cron-driven ingestion** from HPC systems where jobs call the REST APIs directly (non-interactive).

**Key Corrections (Revision 2):**
- Clarified E3SM parser entrypoint functions and their contracts
- Documented required filesystem layout with file location expectations
- Added comprehensive field-level metadata derivation mapping showing which files provide which fields

---

## Table of Contents

1. [E3SM Parser Entrypoints & Contracts](#1-e3sm-parser-entrypoints--contracts) **[NEW]**
2. [Required Filesystem Layout](#2-required-filesystem-layout) **[NEW]**
3. [Field-Level Metadata Derivation](#3-field-level-metadata-derivation) **[NEW]**
4. [REST Call Flow](#4-rest-call-flow)
5. [Data Handoff Points](#5-data-handoff-points)
6. [Control Decisions](#6-control-decisions)
7. [Side Effects](#7-side-effects)
8. [Transferable Patterns](#8-transferable-patterns)
9. [SimBoard Relevance](#9-simboard-relevance)

---

## 1. E3SM Parser Entrypoints & Contracts

This section documents the **public entrypoint functions** for E3SM parsing, their signatures, required inputs, and outputs. These represent the minimal invocation contract that any system (including SimBoard) must satisfy to parse E3SM experiment data.

### 4.1 Primary Entrypoint: `parseE3SM.parseData()`

**Location:** `portal/pace/e3sm/e3smParser/parseE3SM.py:65-184`

**Function Signature:**
```python
def parseData(zipfilename: str, uploaduser: str) -> str:
    """
    Parse E3SM experiments from uploaded zip file.
    
    Args:
        zipfilename: Name of zip file in UPLOAD_FOLDER (e.g., 'pace-exps-user-timestamp.zip')
        uploaduser: GitHub username of uploader (for attribution)
    
    Returns:
        'success' if all experiments parsed successfully
        'fail' if any experiment failed
        'ERROR' if fatal error occurred
    """
```

**Preconditions:**
1. Zip file exists at `{UPLOAD_FOLDER}/{zipfilename}`
2. Zip contains directory structure: `{filename-without-ext}/exp*/`
3. Each `exp*` directory contains required E3SM files (see section 2)
4. Database connection initialized via `pace_common.initDatabase()`
5. Global directories set: `PACE_LOG_DIR`, `EXP_DIR`, `UPLOAD_FOLDER`

**Side Effects:**
1. Extracts zip to `{UPLOAD_FOLDER}/`
2. Creates log files in `PACE_LOG_DIR`
3. Inserts records into 13+ database tables
4. Creates archive zip in `EXP_DIR`
5. Uploads archive to Minio object storage
6. Deletes extracted files from `UPLOAD_FOLDER`

**Orchestration Flow:**
```
parseData()
  ├─ Extract zip and tar.gz files
  ├─ Discover experiment directories (exp*)
  ├─ For each experiment:
  │   └─ insertExperiment()
  │       ├─ insertE3SMTiming() → Core metadata extraction
  │       ├─ insertTiming() → Per-rank timing trees
  │       ├─ insertMemoryFile() → Memory profiling data
  │       ├─ insertScorpioStats() → I/O statistics
  │       ├─ parseCaseDocs.loaddb_casedocs() → XML/namelist files
  │       ├─ insertBuildTimeFile() → Build metrics
  │       ├─ insertPreviewRunFile() → Job parameters
  │       ├─ insertScripts() → Shell scripts
  │       ├─ zipFolder() → Archive experiment
  │       └─ db.session.commit()
  └─ Cleanup staging area
```

### 4.2 Core Metadata Parser: `parseE3SMTiming.parseE3SMtiming()`

**Location:** `portal/pace/e3sm/e3smParser/parseE3SMTiming.py:61-255`

**Function Signature:**
```python
def parseE3SMtiming(filename: str) -> Tuple[bool, dict, list, list]:
    """
    Parse e3sm_timing.* file to extract experiment metadata.
    
    Args:
        filename: Path to e3sm_timing.* file (plain or .gz)
    
    Returns:
        Tuple of:
        - successFlag (bool): True if parsing succeeded
        - timingProfileInfo (dict): Experiment metadata (20+ fields)
        - componentTable (list): PE layout configuration
        - runTimeTable (list): Component timing breakdown
    """
```

**Extracted Fields (20 fields):**
```python
timingProfileInfo = {
    'case': str,              # Case name
    'lid': str,               # Log ID (unique run identifier)
    'machine': str,           # HPC system name
    'caseroot': str,          # Case directory path
    'timeroot': str,          # Timing directory path
    'user': str,              # Username
    'curr': str,              # Current date/time string
    'long_res': str,          # Long grid resolution
    'long_compset': str,      # Long component set
    'stop_option': str,       # Stop criterion (e.g., 'ndays')
    'stop_n': str,            # Stop value (e.g., '20')
    'run_length': str,        # Run length in days
    'total_pes': str,         # Total PEs active
    'mpi_task': str,          # MPI tasks per node
    'pe_count': str,          # PE count for cost estimate
    'model_cost': str,        # Model cost (pe-hrs/simulated_year)
    'model_throughput': str,  # Model throughput (simulated_years/day)
    'actual_ocn': str,        # Actual ocean init wait time
    'init_time': str,         # Initialization time (seconds)
    'run_time': str,          # Run time (seconds)
    'final_time': str         # Finalization time (seconds)
}
```

**Component Table Structure:**
```python
# Extracted from e3sm_timing file's component layout table
componentTable = [
    component_name, comp_pes, root_pe, tasks, threads, instances, stride,
    # Repeated for ~9 components (CPL, ATM, LND, ICE, OCN, ROF, GLC, WAV, ESP)
]
```

**Runtime Table Structure:**
```python
# Extracted from e3sm_timing file's TOT Run Time table
runTimeTable = [
    component, seconds, model_day, model_years,
    # Repeated for each component + communication component
]
```

### 4.3 Resolution & Compset Parser: `parseReadMe.parseReadme()`

**Location:** `portal/pace/e3sm/e3smParser/parseReadMe.py:19-74`

**Function Signature:**
```python
def parseReadme(readmefilename: str) -> Union[dict, bool]:
    """
    Parse README.case.* file to extract resolution and compset.
    
    Args:
        readmefilename: Path to README.case.* file (plain or .gz)
    
    Returns:
        dict with 'res' and 'compset' keys, or False on error
    """
```

**Extracted Fields:**
```python
{
    'res': str,      # Grid resolution (short form, e.g., 'ne30_g16')
    'compset': str,  # Component set (short form, e.g., 'F2010')
    'name': str,     # Script name (create_newcase)
    'date': str,     # Creation date
    # Plus any other flags from create_newcase command line
}
```

**Parsing Strategy:**
- Searches for `create_newcase` command in README
- Extracts command-line arguments: `--res`, `--compset`, etc.
- Returns short-form resolution/compset (contrast with long-form in e3sm_timing)

### 4.4 XML Configuration Parser: `parseCaseDocs.loaddb_casedocs()`

**Location:** `portal/pace/e3sm/e3smParser/parseCaseDocs.py:85-156`

**Function Signature:**
```python
def loaddb_casedocs(casedocpath: str, db: SQLAlchemy, currExpObj: E3SMexp) -> bool:
    """
    Parse XML, namelist, and RC files in CaseDocs directory.
    
    Args:
        casedocpath: Path to CaseDocs.* directory
        db: SQLAlchemy database session
        currExpObj: Current experiment ORM object (for updating)
    
    Returns:
        True on success
        
    Side Effects:
        - Inserts into xml_inputs, namelist_inputs, rc_inputs tables
        - Updates currExpObj.case_group (from env_case.xml)
        - Updates currExpObj.compiler, currExpObj.mpilib (from env_build.xml)
    """
```

**Recognized File Prefixes:**

**Namelists** (29 types):
```python
namelists = (
    "atm_in", "atm_modelio", "cpl_modelio", "drv_flds_in", "drv_in",
    "esp_modelio", "glc_modelio", "ice_modelio", "lnd_in", "lnd_modelio",
    "mosart_in", "mpaso_in", "mpassi_in", "ocn_modelio", "rof_modelio",
    "user_nl_cam", "user_nl_clm", "user_nl_cpl", "user_nl_mosart",
    "user_nl_mpascice", "user_nl_mpaso", "wav_modelio", "iac_modelio",
    "docn_in", "user_nl_docn", "user_nl_cice", "ice_in",
    "user_nl_elm", "user_nl_eam", "user_nl_mali", "mali_in"
)
```

**XML Files** (9 types):
```python
xmlfiles = (
    "env_archive", "env_batch", "env_build", "env_case",
    "env_mach_pes", "env_mach_specific", "env_run", "env_workflow",
    "namelist_scream"
)
```

**RC Files** (1 type):
```python
rcfiles = ("seq_maps",)
```

**Key Metadata Extraction:**

1. **From `env_case.xml`:**
   - Extracts `CASE_GROUP` field
   - Updates `currExpObj.case_group`

2. **From `env_build.xml`:**
   - Extracts `COMPILER` field → `currExpObj.compiler`
   - Extracts `MPILIB` field → `currExpObj.mpilib`

**Code Reference:**
```python
# portal/pace/e3sm/e3smParser/parseCaseDocs.py:121-129
if nameseq[0] == 'env_case':
    case_group = getCaseGroup(json.loads(data))
    currExpObj.case_group = case_group
    db.session.merge(currExpObj)
elif nameseq[0] == 'env_build':
    envBuildData = getEnvBuild(json.loads(data))
    currExpObj.compiler = envBuildData['compiler']
    currExpObj.mpilib = envBuildData['mpilib']
    db.session.merge(currExpObj)
```

### 1.5 Model Timing Tree Parser: `parseModelTiming.parse()`

**Location:** `portal/pace/e3sm/e3smParser/parseModelTiming.py`

**Function Signature:**
```python
def parse(fileInput: TextIOWrapper) -> str:
    """
    Parse GPTL timing output to JSON tree structure.
    
    Args:
        fileInput: Opened file handle to model_timing.* file
    
    Returns:
        JSON string representing timing hierarchy
    """
```

**Output Structure:**
```json
[
  [
    {
      "name": "CPL:RUN_LOOP",
      "multiParent": false,
      "children": [...],
      "values": {
        "count": 240,
        "wallmax": 1234.56,
        "wallmin": 1234.50,
        "walltotal": 296295.00,
        "processes": 256,
        "threads": 1,
        "wallmax_proc": 0,
        "wallmax_thrd": 0,
        "wallmin_proc": 255,
        "wallmin_thrd": 0,
        "on": true
      }
    }
  ]
]
```

### 1.6 Summary: Entrypoint Contract

**For SimBoard Integration:**

To parse E3SM data **without** PACE's REST layer, SimBoard needs:

1. **Filesystem Layout** (see Section 2):
   - Experiment directory with specific files
   - CaseDocs subdirectory with XML/namelist files

2. **Required Functions**:
   ```python
   from pace.e3sm.e3smParser import parseE3SMTiming, parseReadMe, parseCaseDocs
   from pace.e3sm.e3smParser import parseModelTiming, parseMemoryProfile, parseScorpioStats
   
   # Core metadata
   success, metadata, pe_layout, runtime = parseE3SMTiming.parseE3SMtiming(e3sm_timing_file)
   readme_data = parseReadMe.parseReadme(readme_file)
   
   # Timing trees
   for rank_file in timing_tarball:
       timing_json = parseModelTiming.parse(rank_file)
   
   # Configuration
   parseCaseDocs.loaddb_casedocs(casedocs_dir, db, exp_obj)
   ```

3. **No REST Layer Required**:
   - Parsers are pure functions (mostly)
   - Input: File paths
   - Output: Dictionaries, lists, JSON strings
   - Side effects: Only database writes (can be abstracted)

---

## 2. Required Filesystem Layout

This section documents the expected filesystem structure that E3SM parsers require. Understanding this is crucial for:
1. Validating uploaded experiments before parsing
2. Implementing parsers in SimBoard without PACE's specific directory structure
3. Identifying which files are optional vs. required

### 5.1 Upload Archive Structure

**Expected Zip Format:**
```
pace-exps-{user}-{timestamp}.zip
└── pace-exps-{user}-{timestamp}/
    ├── exp0/
    │   ├── e3sm_timing.{LID}[.gz]              [REQUIRED]
    │   ├── timing.{LID}.tar.gz                  [REQUIRED]
    │   ├── README.case.{LID}[.gz]               [REQUIRED]
    │   ├── GIT_DESCRIBE.{LID}[.gz]              [REQUIRED]
    │   ├── CaseDocs.{LID}/                      [REQUIRED]
    │   │   ├── env_case.xml.gz                  [Required for case_group]
    │   │   ├── env_build.xml.gz                 [Required for compiler/mpilib]
    │   │   ├── env_run.xml.gz
    │   │   ├── env_mach_pes.xml.gz
    │   │   ├── atm_in.*.gz                      [Component namelists]
    │   │   ├── lnd_in.*.gz
    │   │   ├── ocn_in.*.gz
    │   │   ├── user_nl_*.*.gz                   [User modifications]
    │   │   └── seq_maps.rc.*.gz                 [Mapping files]
    │   ├── spio_stats.{LID}[.gz]                [OPTIONAL]
    │   ├── memory.{LID}[.gz]                    [OPTIONAL]
    │   ├── build_times.txt.{LID}[.gz]           [OPTIONAL]
    │   ├── preview_run.log.{LID}[.gz]           [OPTIONAL]
    │   ├── replay.sh.{LID}[.gz]                 [OPTIONAL]
    │   └── run_e3sm.sh.{LID}[.gz]               [OPTIONAL]
    ├── exp1/
    │   └── ... (same structure)
    └── expN/
        └── ... (same structure)
```

**Key Observations:**

1. **{LID}** = Log ID, unique identifier for this run (format: `YYYYMMDD-HHMMSS`)
   - Example: `43235257.210608-222102`

2. **File Naming Convention:**
   - Base name + `.` + LID + optional `.gz` extension
   - Examples:
     - `e3sm_timing.43235257.210608-222102.gz`
     - `timing.43235257.210608-222102.tar.gz`
     - `README.case.43235257.210608-222102`

3. **Compression:**
   - Most files are gzip-compressed (`.gz`)
   - Timing files are tar.gz archives
   - Parsers handle both compressed and uncompressed

### 5.2 Required Files (Minimum for Successful Parse)

**Core Metadata Files:**

1. **`e3sm_timing.{LID}[.gz]`**
   - **Purpose:** Primary metadata source
   - **Contains:** Case name, machine, user, PE layout, timing summary, component tables
   - **Fields Extracted:** 20+ fields (see Section 1.2)
   - **Parser:** `parseE3SMTiming.parseE3SMtiming()`
   - **Failure Impact:** Parse fails immediately

2. **`timing.{LID}.tar.gz`**
   - **Purpose:** Per-rank detailed timing trees
   - **Contains:** Tarball with `model_timing.0000`, `model_timing.0001`, ..., `model_timing_stats`
   - **Parser:** `parseModelTiming.parse()`
   - **Failure Impact:** Parse fails immediately

3. **`README.case.{LID}[.gz]`**
   - **Purpose:** Resolution and compset (short forms)
   - **Contains:** create_newcase command with `--res` and `--compset` flags
   - **Fields Extracted:** `res`, `compset`
   - **Parser:** `parseReadMe.parseReadme()`
   - **Failure Impact:** Parse fails immediately

4. **`GIT_DESCRIBE.{LID}[.gz]`**
   - **Purpose:** E3SM version information
   - **Contains:** Git describe output (e.g., `v2.1.0-123-g4a5b6c7`)
   - **Parser:** `parseModelVersion.parseModelVersion()`
   - **Failure Impact:** Parse fails immediately

5. **`CaseDocs.{LID}/`**
   - **Purpose:** XML configurations and namelists
   - **Required Subfiles:**
     - `env_case.xml.gz` (for case_group)
     - `env_build.xml.gz` (for compiler/mpilib)
   - **Parser:** `parseCaseDocs.loaddb_casedocs()`
   - **Failure Impact:** Parse succeeds but misses case_group/compiler/mpilib

### 5.3 Optional Files (Enhance Data but Not Required)

**Performance and Configuration Files:**

1. **`spio_stats.{LID}[.gz]`**
   - **Purpose:** SCORPIO I/O statistics
   - **Contains:** JSON with per-file I/O metrics
   - **Parser:** `parseScorpioStats.loaddb_scorpio_stats()`
   - **Failure Impact:** None (skipped if missing)

2. **`memory.{LID}[.gz]`**
   - **Purpose:** Memory profiling data
   - **Contains:** CSV with memory usage over time
   - **Parser:** `parseMemoryProfile.loaddb_memfile()`
   - **Failure Impact:** None (skipped if missing)

3. **`build_times.txt.{LID}[.gz]`**
   - **Purpose:** Build/compilation timing
   - **Contains:** Text file with per-component build times
   - **Parser:** `parseBuildTime.loaddb_buildTimesFile()`
   - **Failure Impact:** None (skipped if missing)

4. **`preview_run.log.{LID}[.gz]`**
   - **Purpose:** Job submission parameters
   - **Contains:** Preview run output with node count, tasks, threads, etc.
   - **Parser:** `parsePreviewRun.load_previewRunFile()`
   - **Failure Impact:** None (skipped if missing)

5. **`replay.sh.{LID}[.gz]`** and **`run_e3sm.sh.{LID}[.gz]`**
   - **Purpose:** Shell scripts for reproduction
   - **Contains:** Bash scripts
   - **Parser:** `parseReplaysh.load_replayshFile()`, `parseRunE3SMsh.load_rune3smshfile()`
   - **Failure Impact:** None (skipped if missing)

### 5.4 Discovery Logic

**Code Reference:** `portal/pace/e3sm/e3smParser/parseE3SM.py:98-139`

```python
# Discover experiment directories
experimentDirs = []
for dir in os.listdir(os.path.join(fpath, tmpfilename)):
    if dir.startswith('exp'):
        experimentDirs.append(os.path.join(fpath, tmpfilename, dir))

# For each experiment, collect file paths
experimentFiles = []
for root in experimentDirs:
    model = {
        "timingfile": None,      # timing.*
        "allfile": None,         # e3sm_timing.*
        "readmefile": None,      # README.case.*
        "gitdescribefile": None, # GIT_DESCRIBE.*
        "scorpiofile": None,     # spio_stats.*
        "memoryfile": None,      # memory.*
        "casedocs": None,        # CaseDocs.*/ (directory)
        "buildtimefile": None,   # build_times.txt.*
        "previewrunfile": None,  # preview_run.log.*
        "replayshfile": None,    # replay.sh.*
        "run_e3sm_file": None    # run_e3sm.sh.*
    }
    
    # Walk directory tree to find files
    for path, subdirs, files in os.walk(root):
        for name in files:
            if name.startswith("timing."):
                model['timingfile'] = os.path.join(path, name)
            elif name.startswith("e3sm_timing."):
                model['allfile'] = os.path.join(path, name)
            # ... etc for all file types
        for name in subdirs:
            if name.startswith("CaseDocs."):
                model['casedocs'] = os.path.join(path, name)
    
    experimentFiles.append(model)
```

**Discovery Strategy:**
1. Experiments must be in directories starting with `exp`
2. Files are discovered via filename prefix matching (not extension)
3. `CaseDocs` is discovered as a subdirectory
4. Missing optional files result in `None` values (gracefully handled)

### 5.5 SimBoard Implications

**For SimBoard to Parse E3SM Data:**

1. **Accept Flexible Layouts:**
   - Don't require exact PACE directory structure
   - Use file manifest or metadata file instead of filename scanning
   - Allow users to specify file roles explicitly

2. **Validate Before Parsing:**
   ```python
   def validate_e3sm_experiment(exp_dir):
       required = [
           'e3sm_timing.*',
           'timing.*.tar.gz',
           'README.case.*',
           'GIT_DESCRIBE.*',
           'CaseDocs.*/'
       ]
       for pattern in required:
           if not glob.glob(os.path.join(exp_dir, pattern)):
               raise ValidationError(f"Missing required file: {pattern}")
   ```

3. **Support Partial Ingestion:**
   - Parse core metadata even if optional files missing
   - Mark fields as "unavailable" rather than failing entire parse

---

## 3. Field-Level Metadata Derivation

This section provides a comprehensive mapping of **where each metadata field comes from**. This is critical for:
1. Understanding data provenance
2. Implementing parsers without PACE's database schema
3. Validating data quality and completeness

### 3.1 E3SMexp Table Fields (Primary Metadata)

**Database Table:** `e3smexp`  
**ORM Class:** `E3SMexp` in `portal/pace/e3sm/e3smDb/datastructs.py`

| Field | Source File | Source Parser | Extraction Logic | Required |
|-------|-------------|---------------|------------------|----------|
| `expid` | Database | Auto-increment | Generated by database | ✓ |
| `case` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "Case :" | ✓ |
| `lid` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "LID :" | ✓ |
| `machine` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "Machine :" | ✓ |
| `caseroot` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "Caseroot :" | ✓ |
| `timeroot` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "Timeroot :" | ✓ |
| `user` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "User :" | ✓ |
| `exp_date` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "Curr :", converted via `changeDateTime()` | ✓ |
| `long_res` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "grid :" (long form) | ✓ |
| `res` | `README.case.*` | `parseReadMe` | `--res` flag from create_newcase command (short form) | ✓ |
| `compset` | `README.case.*` | `parseReadMe` | `--compset` flag from create_newcase command (short form) | ✓ |
| `long_compset` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "compset :" (long form) | ✓ |
| `stop_option` | `e3sm_timing.*` | `parseE3SMTiming` | From "stop option" line, first part before comma | ✓ |
| `stop_n` | `e3sm_timing.*` | `parseE3SMTiming` | From "stop option" line, after "stop_n =" | ✓ |
| `run_length` | `e3sm_timing.*` | `parseE3SMTiming` | From "run length :" line, numeric part | ✓ |
| `total_pes_active` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "total :", after colon | ✓ |
| `mpi_tasks_per_node` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "mpi :", after colon | ✓ |
| `pe_count_for_cost_estimate` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "pe :", after colon | ✓ |
| `model_cost` | `e3sm_timing.*` | `parseE3SMTiming` | First "Model :" line value (pe-hrs/simulated_year) | ✓ |
| `model_throughput` | `e3sm_timing.*` | `parseE3SMTiming` | Second "Model :" line value (simulated_years/day) | ✓ |
| `actual_ocn_init_wait_time` | `e3sm_timing.*` | `parseE3SMTiming` | Line starting with "Actual :" | ✓ |
| `init_time` | `e3sm_timing.*` | `parseE3SMTiming` | First "Init :" line value (seconds) | ✓ |
| `run_time` | `e3sm_timing.*` | `parseE3SMTiming` | First "Run :" line value (seconds) | ✓ |
| `final_time` | `e3sm_timing.*` | `parseE3SMTiming` | First "Final :" line value (seconds) | ✓ |
| `version` | `GIT_DESCRIBE.*` | `parseModelVersion` | Git describe output (e.g., "v2.1.0-123-g4a5b6c7") | ✓ |
| `upload_by` | REST API | Function argument | Passed from `/fileparse` endpoint | ✓ |
| `case_group` | `env_case.xml` | `parseCaseDocs` | From `<entry id="CASE_GROUP">` in env_case.xml | ○ |
| `compiler` | `env_build.xml` | `parseCaseDocs` | From `<entry id="COMPILER">` in env_build.xml | ○ |
| `mpilib` | `env_build.xml` | `parseCaseDocs` | From `<entry id="MPILIB">` in env_build.xml | ○ |

**Legend:**
- ✓ Required for successful parse
- ○ Optional (filled if files available)

**Code Reference for XML Fields:**
```python
# portal/pace/e3sm/e3smParser/parseCaseDocs.py:41-78

# Extract case_group from env_case.xml
def getCaseGroup(jsondata):
    caseDir = jsondata["file"]["group"]
    for model in caseDir:
        if model.get('@id') == 'case_desc' and 'entry' in model:
            for entry in model['entry']:
                if entry.get('@id') == 'CASE_GROUP':
                    return entry.get('@value')
    return None

# Extract compiler and mpilib from env_build.xml
def getEnvBuild(jsondata):
    output = {'compiler': None, 'mpilib': None}
    buildModel = jsondata["file"]["group"]
    for model in buildModel:
        if model.get('@id') == 'build_macros' and 'entry' in model:
            for entry in model['entry']:
                if entry.get('@id') == 'COMPILER':
                    output['compiler'] = entry.get('@value')
                elif entry.get('@id') == 'MPILIB':
                    output['mpilib'] = entry.get('@value')
    return output
```

### 3.2 Resolution and Compset: Long vs. Short Forms

**Important Distinction:**

E3SM uses two forms for resolution and compset:
1. **Short Form** (user-friendly): Extracted from `README.case.*`
2. **Long Form** (fully expanded): Extracted from `e3sm_timing.*`

**Example:**

| Field | Short Form (README) | Long Form (e3sm_timing) |
|-------|---------------------|-------------------------|
| Resolution | `ne30_g16` | `a%ne30np4_l%null_oi%gx1v7_r%r05_g%null_w%null_m%gx1v7` |
| Compset | `F2010` | `2010_CAM60_CLM50%SP_CICE_POP2%ECO%ABIO-DIC_MOSART_SGLC_SWAV` |

**Why Both?**
- **Short form:** Easy for humans to read, used in UI and queries
- **Long form:** Unambiguous, specifies exact component configurations

**SimBoard Recommendation:**
Store both forms in separate fields for flexibility.

### 9.3 Component Layout (pelayout Table)

**Database Table:** `pelayout`  
**Source File:** `e3sm_timing.*`  
**Parser:** `parseE3SMTiming.parseE3SMtiming()`  
**Extraction:** Component table section of e3sm_timing file

**Fields Extracted Per Component:**

| Field | Description | Example |
|-------|-------------|---------|
| `expid` | Foreign key to e3smexp | 12345 |
| `component` | Component name | CPL, ATM, LND, ICE, OCN, ROF, GLC, WAV, ESP |
| `comp_pes` | Component PEs | 256 |
| `root_pe` | Root PE number | 0 |
| `tasks` | Number of tasks | 256 |
| `threads` | Threads per task | 1 |
| `instances` | Number of instances | 1 |
| `stride` | Stride value | 1 |

**Code Reference:**
```python
# portal/pace/e3sm/e3smParser/parseE3SMTiming.py:214-216
if firstWord == 'component':
    for j in range(9):
        tablelist.append(lines[i+j+2])  # Skip header rows
    componentTableSuccess = True
```

### 3.4 Component Runtime (runtime Table)

**Database Table:** `runtime`  
**Source File:** `e3sm_timing.*`  
**Parser:** `parseE3SMTiming.parseE3SMtiming()`  
**Extraction:** TOT Run Time table section

**Fields Extracted Per Component:**

| Field | Description | Example |
|-------|-------------|---------|
| `expid` | Foreign key to e3smexp | 12345 |
| `component` | Component name (may include _COMM for communication) | CPL, ATM, LND_COMM, ICE |
| `seconds` | Wall clock time (seconds) | 1234.56 |
| `model_day` | Seconds per model day | 0.05 |
| `model_years` | Seconds per model year | 18.25 |

**Code Reference:**
```python
# portal/pace/e3sm/e3smParser/parseE3SMTiming.py:219-228
elif firstWord == 'TOT':
    for j in range(11):  # 11 components
        component = lines[i+j].split()
        if component[1] == 'Run':
            runTimeTable.append(component[0])
        else:
            runTimeTable.append(str(component[0]) + '_COMM')
        runTimeTable.append(component[3])  # seconds
        runTimeTable.append(component[5])  # model_day
        runTimeTable.append(component[7])  # model_years
```

### 3.5 Model Timing Trees (model_timing Table)

**Database Table:** `model_timing`  
**Source File:** `timing.*.tar.gz` (contains `model_timing.0000`, `model_timing.0001`, ..., `model_timing_stats`)  
**Parser:** `parseModelTiming.parse()`  
**Format:** JSON tree structure

**Fields Per Entry:**

| Field | Description | Example |
|-------|-------------|---------|
| `expid` | Foreign key to e3smexp | 12345 |
| `rank` | MPI rank number or "stats" | "0", "1", ..., "255", "stats" |
| `jsonVal` | JSON tree of timing data | (see Section 1.5) |

**JSON Structure:**
```json
{
  "name": "CPL:RUN_LOOP",
  "values": {
    "count": 240,
    "wallmax": 1234.56,
    "wallmin": 1234.50,
    "walltotal": 296295.00,
    "processes": 256,
    "threads": 1
  },
  "children": [...]
}
```

### 3.6 Configuration Files

**XML Inputs (xml_inputs Table):**
- **Source:** `CaseDocs.{LID}/*.xml.gz`
- **Parser:** `parseXML.loaddb_xmlfile()`
- **Format:** XML → JSON via xmltodict
- **Key Files:**
  - `env_case.xml`: Case configuration
  - `env_build.xml`: Build/compiler settings
  - `env_run.xml`: Runtime parameters
  - `env_mach_pes.xml`: PE layout details

**Namelist Inputs (namelist_inputs Table):**
- **Source:** `CaseDocs.{LID}/*_in.*.gz`, `CaseDocs.{LID}/user_nl_*.*.gz`
- **Parser:** `parseNameList.loaddb_namelist()` or `parseText.load_text()` (for user_nl)
- **Format:** Fortran namelist → JSON
- **Key Files:**
  - `atm_in.*`: Atmosphere namelists
  - `lnd_in.*`: Land namelists
  - `ocn_in.*`: Ocean namelists
  - `user_nl_*`: User modifications

**RC Inputs (rc_inputs Table):**
- **Source:** `CaseDocs.{LID}/seq_maps.rc.*.gz`
- **Parser:** `parseRC.loaddb_rcfile()`
- **Format:** RC file → JSON

### 3.7 Optional Performance Data

**SCORPIO I/O Stats (scorpio_stats Table):**
- **Source:** `spio_stats.{LID}[.gz]`
- **Parser:** `parseScorpioStats.loaddb_scorpio_stats()`
- **Fields:**
  - `expid`, `name`, `data` (JSON), `iopercent`, `iotime`, `version`

**Memory Profile (memfile_inputs Table):**
- **Source:** `memory.{LID}[.gz]`
- **Parser:** `parseMemoryProfile.loaddb_memfile()`
- **Fields:**
  - `expid`, `name` ("memory"), `data` (CSV string)

**Build Time (build_time Table):**
- **Source:** `build_times.txt.{LID}[.gz]`
- **Parser:** `parseBuildTime.loaddb_buildTimesFile()`
- **Fields:**
  - `expid`, `data` (JSON), `total_computecost`, `total_elapsed_time`

**Preview Run (preview_run Table):**
- **Source:** `preview_run.log.{LID}[.gz]`
- **Parser:** `parsePreviewRun.load_previewRunFile()`
- **Fields:**
  - `expid`, `nodes`, `total_tasks`, `tasks_per_node`, `thread_count`, `ngpus_per_node`, `mpirun`, `submit_cmd`, `env` (JSON), `omp_threads`

**Scripts (script_files Table):**
- **Source:** `replay.sh.{LID}[.gz]`, `run_e3sm.sh.{LID}[.gz]`
- **Parser:** `parseReplaysh.load_replayshFile()`, `parseRunE3SMsh.load_rune3smshfile()`
- **Fields:**
  - `expid`, `name` ("replay_sh" or "run_e3sm_sh"), `data` (bash script text)

### 3.8 Metadata Derivation Summary

**Key Takeaways for SimBoard:**

1. **Primary Source: `e3sm_timing.*`**
   - Contains 20+ core metadata fields
   - Text-based format (easy to parse)
   - Single source of truth for most experiment metadata

2. **Secondary Source: `README.case.*`**
   - Provides short-form resolution and compset
   - Contains create_newcase command-line arguments
   - Essential for user-friendly display

3. **Tertiary Sources: XML Files**
   - Provide additional context (case_group, compiler, mpilib)
   - Not required for basic ingestion
   - Enable advanced filtering and analysis

4. **Derivation is Deterministic:**
   - No complex inference or heuristics
   - Each field has a single, well-defined source
   - Parsing is straightforward text extraction

5. **SimBoard Can Reuse Parsers:**
   - PACE parsers are mostly pure functions
   - Input: File path
   - Output: Dictionary or JSON
   - Minimal PACE-specific dependencies

---

## 4. REST Call Flow

### 4.1 Upload Flow (`/upload`)

**Location:** `portal/pace/webapp.py` lines 96-115

**Sequence:**

```
1. Client → POST /upload
   ├─ Headers: multipart/form-data
   ├─ Fields: 
   │   ├─ 'file': binary zip data
   │   └─ 'filename': original filename string
   │
2. Flask Handler (upload_file)
   ├─ Extract filename from request.files['filename']
   ├─ Validate file extension via allowed_file()
   │   └─ Only .zip allowed (line 44)
   │
3. Filesystem Staging
   ├─ Check if extraction directory exists → remove it
   ├─ Check if zip file exists → remove it
   ├─ Secure filename via werkzeug.secure_filename()
   ├─ Save to UPLOAD_FOLDER (from getDirectories())
   │
4. Response
   └─ Return 'complete' or error message
```

**Code Reference:**
```python
# portal/pace/webapp.py:96-115
@app.route('/upload', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        file = request.files['file']
        zipfilename = str(request.files['filename'])
        tmpfilename = zipfilename.split('.')[0]
        if file and allowed_file(file.filename):
            # Cleanup existing files/dirs
            if os.path.isdir(os.path.join(UPLOAD_FOLDER,tmpfilename)):
                shutil.rmtree(os.path.join(UPLOAD_FOLDER,tmpfilename))
            if os.path.exists(os.path.join(UPLOAD_FOLDER,zipfilename)):
                os.remove(os.path.join(UPLOAD_FOLDER,zipfilename))
            filename = secure_filename(file.filename)
            file.save(os.path.join(UPLOAD_FOLDER, filename))
            return('complete')
```

### 4.2 Parse Flow (`/fileparse`)

**Location:** `portal/pace/webapp.py` lines 117-131

**Sequence:**

```
1. Client → POST /fileparse
   ├─ Form data:
   │   ├─ 'filename': pace-exps-{user}-{timestamp}.zip
   │   ├─ 'user': GitHub username
   │   └─ 'project': 'e3sm'
   │
2. Flask Handler (fileparse)
   ├─ Validate user (alphanumeric + dash/dot/underscore)
   ├─ Validate filename (regex: pace-exps-[user]-[timestamp].zip)
   │
3. Call parse.parseData()
   └─ Return status/logfilename
```

**Code Reference:**
```python
# portal/pace/webapp.py:117-131
@app.route('/fileparse', methods=['GET','POST'])
def fileparse():
    if request.method == 'POST':
        filename = request.form['filename']
        user = request.form['user']
        project = request.form['project']
        if not bool(re.match('^[a-zA-Z0-9\-._]+$', user)):
            return('ERROR')
        if not bool(re.match('^pace-exps-[a-zA-Z0-9\-_]+.zip$', filename)):
            return('ERROR')
        return(parse.parseData(filename,user,project))
```

### 4.3 Parser Orchestration Flow

**Location:** `portal/pace/parse.py` lines 21-47

**High-Level Sequence:**

```
1. parse.parseData(filename, user, project)
   ├─ Setup logging (redirect stdout/stderr to logfile)
   │   ├─ logfilename: pace-{user}-{timestamp}.log
   │   └─ intlogfilename: internal-{logfilename}
   │
2. Route to project-specific parser
   └─ if project == 'e3sm':
       └─ parseE3SM.parseData(filename, user)
```

**Code Reference:**
```python
# portal/pace/parse.py:21-47
def parseData(zipfilename,uploaduser,project):
    old_stdout = sys.stdout
    old_stderr = sys.stderr
    logfilename = 'pace-'+str(uploaduser)+'-'+str(datetime.now().strftime('%Y-%m-%d-%H:%M:%S'))+'.log'
    intlogfilename = "internal-" + logfilename
    logfilepath = PACE_LOG_DIR + logfilename
    intlogfilepath = PACE_LOG_DIR + intlogfilename
    log_file = open(logfilepath,'w')
    intlog_file = open(intlogfilepath,'w')
    sys.stdout = log_file
    sys.stderr = intlog_file
    
    if project == 'e3sm':
        status = parseE3SM.parseData(zipfilename,uploaduser)
    else:
        status = 'fail'
    
    sys.stdout = old_stdout
    sys.stderr = old_stderr
    log_file.close()
    intlog_file.close()
    return(status+'/'+str(logfilename))
```

### 4.4 E3SM Parser Flow

**Location:** `portal/pace/e3sm/e3smParser/parseE3SM.py` lines 65-184

**Detailed Sequence:**

```
1. parseE3SM.parseData(zipfilename, uploaduser)
   │
2. Extract zip file to UPLOAD_FOLDER
   ├─ Unzip aggregated file
   ├─ Extract any tar.gz files (with safemembers security)
   │
3. Discover experiment directories
   ├─ Find directories starting with 'exp'
   ├─ Walk each directory to collect file paths:
   │   ├─ timing.*
   │   ├─ e3sm_timing.*
   │   ├─ README.case.*
   │   ├─ GIT_DESCRIBE.*
   │   ├─ spio_stats.*
   │   ├─ memory.*
   │   ├─ CaseDocs.*/ (directory)
   │   ├─ build_times.txt.*
   │   ├─ preview_run.log.*
   │   ├─ replay.sh.*
   │   └─ run_e3sm.sh.*
   │
4. For each experiment directory:
   └─ Call insertExperiment()
      │
      ├─ insertE3SMTiming()
      │   ├─ parseE3SMTiming.parseE3SMtiming()
      │   ├─ Check for duplicates (user/machine/date/case)
      │   ├─ parseReadMe.parseReadme()
      │   ├─ parseModelVersion.parseModelVersion()
      │   └─ Insert into e3smexp, exp, pelayout, runtime tables
      │
      ├─ insertTiming()
      │   ├─ Extract model_timing files from timing.* tarball
      │   ├─ Parse each rank file via parseModelTiming.parse()
      │   └─ Insert into model_timing table
      │
      ├─ insertMemoryFile()
      │   ├─ parseMemoryProfile.loaddb_memfile()
      │   └─ Insert into memfile_inputs table
      │
      ├─ insertScorpioStats()
      │   ├─ parseScorpioStats.loaddb_scorpio_stats()
      │   └─ Insert into scorpio_stats table
      │
      ├─ parseCaseDocs.loaddb_casedocs()
      │   ├─ Parse XML files via parseXML
      │   ├─ Parse namelist files via parseNameList
      │   ├─ Parse RC files via parseRC
      │   └─ Insert into xml_inputs, namelist_inputs, rc_inputs tables
      │
      ├─ insertBuildTimeFile()
      │   ├─ parseBuildTime.loaddb_buildTimesFile()
      │   └─ Insert into build_time table
      │
      ├─ insertPreviewRunFile()
      │   ├─ parsePreviewRun.load_previewRunFile()
      │   └─ Insert into preview_run table
      │
      ├─ insertScripts()
      │   ├─ parseReplaysh.load_replayshFile()
      │   ├─ parseRunE3SMsh.load_rune3smshfile()
      │   └─ Insert into script_files table
      │
      ├─ zipFolder() - Archive experiment
      │   ├─ Create zip of CaseDocs directory
      │   ├─ Save to EXP_DIR
      │   └─ Upload to Minio object storage
      │
      └─ db.session.commit()
         └─ Print summary (ExpID, User, Machine, Web Link)
│
5. Cleanup
   ├─ removeFolder() - Delete uploaded files
   │
6. Return status
   └─ 'success' or 'fail'
```

**Code Reference:**
```python
# portal/pace/e3sm/e3smParser/parseE3SM.py:65-184
def parseData(zipfilename,uploaduser):
    try:
        # Extract zip and tar files
        fpath=UPLOAD_FOLDER
        zip_ref=zipfile.ZipFile(os.path.join(fpath,zipfilename),'r')
        zip_ref.extractall(fpath)
        
        # Discover experiment directories
        experimentDirs = []
        for i in range(len(dic1)):
            for dir in os.listdir(os.path.join(fpath,dic1[i])):
                if dir.startswith('exp'):
                    experimentDirs.append(os.path.join(fpath,tmpfilename,dir))
        
        # Parse each experiment
        isSuccess=[]
        for index in range(len(experimentFiles)):
            isSuccess.append(insertExperiment(...))
        
        # Cleanup
        removeFolder(UPLOAD_FOLDER,zipfilename)
        
        if False in isSuccess:
            return('fail')
        else:
            return('success')
```

---

## 5. Data Handoff Points

### 5.1 Client → REST Layer

**Handoff:** HTTP multipart form data

**Data:**
- **Upload Stage:**
  - Binary zip file stream
  - Filename string (expected format: `pace-exps-{user}-{timestamp}.zip`)
  
- **Parse Stage:**
  - `filename`: Must match uploaded filename
  - `user`: GitHub username (alphanumeric + `-._`)
  - `project`: Application identifier (currently only 'e3sm')

**Assumptions:**
- Client is responsible for zip file naming convention
- Client has already authenticated via GitHub OAuth (via pace-upload tool)
- Filename embeds user identity for validation

### 5.2 REST Layer → Filesystem

**Handoff:** File I/O operations

**Paths:**
- `UPLOAD_FOLDER`: Staging area for uploaded files (from `.pacerc` config)
- `PACE_LOG_DIR`: Directory for parsing logs
- `EXP_DIR`: Directory for archived experiment zips

**Path Resolution:**
```python
# portal/pace/pace_common.py:54-64
def getDirectories():
    environment = getEnvironment()
    return environment['pace_log_dir'], environment['exp_dir'], environment['upload_folder']

# Reads from .pacerc:
# [ENVIRONMENT]
# pace_log_dir = /path/to/logs/
# exp_dir = /path/to/data/
# upload_folder = /path/to/upload/
```

**Data Written:**
- Uploaded zip file: `{UPLOAD_FOLDER}/{filename}`
- Extracted directories: `{UPLOAD_FOLDER}/{filename-without-ext}/exp*/`
- Parse logs: `{PACE_LOG_DIR}/pace-{user}-{timestamp}.log`
- Internal logs: `{PACE_LOG_DIR}/internal-pace-{user}-{timestamp}.log`
- Archived experiments: `{EXP_DIR}/exp-{user}-{expid}.zip`

### 5.3 Filesystem → Parser Layer

**Handoff:** File paths passed to parser functions

**Key File Types:**
- `e3sm_timing.*`: High-level timing profile (CSV-like format)
- `timing.*`: Tarball containing per-rank model_timing files
- `README.case.*`: Experiment metadata
- `GIT_DESCRIBE.*`: E3SM version information
- `spio_stats.*`: I/O statistics from SCORPIO library
- `memory.*`: Memory profiling data
- `build_times.txt.*`: Compilation timing
- `preview_run.log.*`: Job submission parameters
- `replay.sh.*`, `run_e3sm.sh.*`: Shell scripts
- `CaseDocs.*/`: Directory with XML, namelist, RC files

**Parser Functions:**
```python
# Mapping of files to parser functions:
parseE3SMTiming.parseE3SMtiming(e3sm_timing)      # → dict with metadata
parseModelTiming.parse(model_timing_file)         # → JSON tree
parseReadMe.parseReadme(readme)                   # → dict with compset/res
parseModelVersion.parseModelVersion(git_describe) # → version string
parseScorpioStats.loaddb_scorpio_stats(spio)      # → list of dicts
parseMemoryProfile.loaddb_memfile(memory)         # → CSV string
parseCaseDocs.loaddb_casedocs(casedocs_dir)       # → multiple tables
parseBuildTime.loaddb_buildTimesFile(buildtimes)  # → dict + totals
parsePreviewRun.load_previewRunFile(preview_run)  # → dict
parseReplaysh.load_replayshFile(replay_sh)        # → string
parseRunE3SMsh.load_rune3smshfile(run_e3sm)       # → string
```

### 5.4 Parser Layer → Database

**Handoff:** SQLAlchemy ORM objects

**Database Tables:**
- `e3smexp`: Main experiment metadata
- `exp`: Simplified experiment view
- `pelayout`: PE layout configuration
- `runtime`: Component timing breakdown
- `model_timing`: Per-rank timing trees (JSON)
- `memfile_inputs`: Memory profile data
- `scorpio_stats`: I/O statistics
- `xml_inputs`, `namelist_inputs`, `rc_inputs`: Configuration files
- `build_time`: Compilation metrics
- `preview_run`: Job parameters
- `script_files`: Shell scripts

**Persistence Pattern:**
```python
# Each parser creates ORM objects:
new_experiment = E3SMexp(case=..., user=..., machine=...)
db.session.add(new_experiment)

# All additions batched, single commit at end:
try:
    db.session.commit()
except SQLAlchemyError:
    db.session.rollback()
    return False
```

### 5.5 Database → Response

**Handoff:** HTTP response string

**Response Format:**
- **Upload:** `'complete'` or `'Error Uploading file, Try again'`
- **Parse:** `'{status}/{logfilename}'`
  - Status: `'success'` or `'fail'` or `'ERROR'`
  - Logfilename: `pace-{user}-{timestamp}.log`

**Client Behavior:**
```python
# client/pace-upload:97-102
req = requests.post(parseurl, data={'filename':uploadfile,'user':uploadUser, 'project':'e3sm'})
flaglist = (req.text).split("/")  # ['success', 'pace-user-2021-02-04-12:34:56.log']
if flaglist[0] == 'success':
    print('Parse Success')
else:
    print('ERROR: Check %s' %flaglist[1])
```

---

## 6. Control Decisions

### 3.1 Validation Gates

#### Gate 1: Upload Request Validation
**Location:** `portal/pace/webapp.py:96-115`

**Checks:**
1. HTTP method is POST
2. File is present in request
3. File extension is `.zip` (only allowed extension)
4. File can be saved successfully

**Action on Failure:**
- Return error message string
- No database impact

#### Gate 2: Parse Request Validation
**Location:** `portal/pace/webapp.py:117-131`

**Checks:**
1. HTTP method is POST
2. User matches regex: `^[a-zA-Z0-9\-._]+$`
3. Filename matches regex: `^pace-exps-[a-zA-Z0-9\-_]+.zip$`
4. Project is 'e3sm' (hardcoded in parse routing)

**Action on Failure:**
- Return `'ERROR'` string
- Parsing never begins

**Security Note:**
These regex checks prevent path traversal and command injection attacks.

#### Gate 3: Duplicate Experiment Check
**Location:** `portal/pace/e3sm/e3smParser/parseE3SM.py:212-218`

**Checks:**
```python
def checkDuplicateExp(euser, emachine, ecurr, ecase):
    eexp_date = changeDateTime(ecurr)
    exp = E3SMexp.query.filter_by(
        user=euser,
        machine=emachine,
        case=ecase,
        exp_date=eexp_date
    ).first()
    return (exp is not None, exp.expid if exp else None)
```

**Action on Duplicate:**
- Log: `"Duplicate of Experiment: {expid}"`
- Skip parsing (return success=True)
- Continue to next experiment in batch

**Rationale:**
Prevents duplicate ingestion from re-runs of cron jobs.

#### Gate 4: Per-Parser Validation
**Location:** Various parser modules

**Examples:**
- Empty file check (memory, scorpio, buildtime parsers)
- File format validation (parsing may raise exceptions)
- Data completeness checks

**Action on Failure:**
- Print error message to log
- Return early (skip rest of parsing for that file type)
- Database transaction rolled back for entire experiment

#### Gate 5: Database Transaction
**Location:** `portal/pace/e3sm/e3smParser/parseE3SM.py:377-389`

**Final Gate:**
```python
try:
    db.session.commit()
    print('    -Complete')
except SQLAlchemyError as e:
    print('    SQL ERROR: %s' %e)
    db.session.rollback()
    return False
```

**Action on Failure:**
- All database insertions for this experiment rolled back
- Error logged
- Continue to next experiment in batch

### 3.2 Error Handling Behavior

**Strategy:** Continue-on-error with logging

**Per-Experiment Errors:**
- Individual experiment failures don't abort the batch
- Each experiment's success tracked in `isSuccess[]` list
- Final status: `'fail'` if any experiments failed, else `'success'`

**Per-File Errors:**
- Missing optional files (memory, scorpio, etc.) logged but not fatal
- Only required files (e3sm_timing, timing, README, GIT_DESCRIBE) cause experiment to fail
- Database session rolled back on any error

**Exception Handling Pattern:**
```python
try:
    # Parse and insert
    successFlag = parseExperiment(...)
except SQLAlchemyError as e:
    print('SQL ERROR: %s' %e)
    db.session.rollback()
    return False
except Exception as e:
    print('ERROR: %s' %e)
    db.session.rollback()
    return False
```

**Filesystem Cleanup:**
- Uploaded files removed after parsing (success or fail)
- Cleanup occurs in `removeFolder()` at end of parseData()

### 9.3 Sync vs Async Execution

**Synchronous (Blocking) Flow:**
- All REST handlers execute synchronously
- Client waits for entire parse operation to complete
- No background job queue or async workers

**Implications:**
- Long-running parsing holds HTTP connection open
- Client timeout if parsing takes too long
- No progress updates during parsing
- Works for cron-driven ingestion (cron jobs are patient)

**Client Behavior:**
```python
# client/pace-upload:97
req = requests.post(parseurl, data={'filename':uploadfile,'user':uploadUser, 'project':'e3sm'})
# Blocks until parsing completes (could be minutes)
```

**Server Execution:**
```python
# portal/pace/webapp.py:131
return(parse.parseData(filename,user,project))
# Blocks Flask worker until all experiments parsed
```

**Scaling Considerations:**
- Limited by number of Flask workers
- Each upload/parse cycle ties up one worker
- Production deployment likely uses gunicorn/uwsgi with multiple workers

---

## 7. Side Effects

### 7.1 Filesystem Writes

#### During Upload (`/upload`)
**Location:** `portal/pace/webapp.py:96-115`

**Writes:**
1. Remove existing extraction directory if present
   ```python
   if os.path.isdir(os.path.join(UPLOAD_FOLDER, tmpfilename)):
       shutil.rmtree(os.path.join(UPLOAD_FOLDER, tmpfilename))
   ```

2. Remove existing zip file if present
   ```python
   if os.path.exists(os.path.join(UPLOAD_FOLDER, zipfilename)):
       os.remove(os.path.join(UPLOAD_FOLDER, zipfilename))
   ```

3. Write uploaded zip file
   ```python
   file.save(os.path.join(UPLOAD_FOLDER, filename))
   ```

#### During Parse (`/fileparse`)
**Location:** `portal/pace/e3sm/e3smParser/parseE3SM.py`

**Writes:**

1. **Extract zip file** (lines 69-70)
   ```python
   zip_ref = zipfile.ZipFile(os.path.join(fpath, zipfilename), 'r')
   zip_ref.extractall(fpath)
   # Creates: {UPLOAD_FOLDER}/pace-exps-{user}-{timestamp}/
   ```

2. **Extract tar.gz files** (lines 78-82)
   ```python
   tar = tarfile.open(dic[i])
   tar.extractall(members=safemembers(tar))
   # Creates experiment directories within upload folder
   ```

3. **Create parse logs** (`portal/pace/parse.py:26-32`)
   ```python
   logfilepath = PACE_LOG_DIR + logfilename
   intlogfilepath = PACE_LOG_DIR + intlogfilename
   log_file = open(logfilepath, 'w')
   intlog_file = open(intlogfilepath, 'w')
   # Creates: {PACE_LOG_DIR}/pace-{user}-{timestamp}.log
   # Creates: {PACE_LOG_DIR}/internal-pace-{user}-{timestamp}.log
   ```

4. **Archive experiment** (lines 596-613)
   ```python
   zipFileFullPath = os.path.join(EXP_DIR, expname)
   shutil.make_archive(zipFileFullPath, 'zip', path)
   # Creates: {EXP_DIR}/exp-{user}-{expid}.zip
   ```

5. **Remove temporary files** (lines 584-592)
   ```python
   shutil.rmtree(os.path.join(removeroot, filename.split('.')[0]))
   os.remove(os.path.join(removeroot, filename))
   # Removes: {UPLOAD_FOLDER}/pace-exps-{user}-{timestamp}/
   # Removes: {UPLOAD_FOLDER}/pace-exps-{user}-{timestamp}.zip
   ```

**Security Considerations:**
- Tarfile extraction uses `safemembers()` to prevent path traversal attacks (lines 39-59)
- Filenames sanitized via `secure_filename()` in upload handler

### 7.2 Logging

**Log Redirection:**
```python
# portal/pace/parse.py:24-33
old_stdout = sys.stdout
old_stderr = sys.stderr
sys.stdout = log_file
sys.stderr = intlog_file
# All print() and exceptions go to log files
```

**Log Content:**
- Parser progress messages
- File-by-file parsing status
- Error messages and stack traces
- Experiment summary (ExpID, User, Machine, URL)

**Log Format Example:**
```
* * * * * * * * * * * * * * PACE Report * * * * * * * * * * * * * *

**************************************************
* Parsing e3sm_timing file: e3sm_timing.43235257.210608-222102
     -Complete
* Parsing README.docs file : README.case.43235257.210608-222102
    -Complete
* Parsing GIT_DESCRIBE file : GIT_DESCRIBE.43235257.210608-222102
    -Complete
* Parsing model timing file : timing.43235257.210608-222102
    -Complete
...
----- Experiment Summary -----
- Experiment ID (ExpID): 12345
- User: johndoe
- Machine: cori-haswell
- Web Link: https://pace.ornl.gov/exp-details/12345
------------------------------
**************************************************
```

**Log Retrieval:**
- Client downloads log via `/downloadlog` endpoint
- Logs persist on server for debugging

### 7.3 Persistence (Database)

**Database Connection:**
```python
# portal/pace/pace_common.py:169-187
def initDatabase():
    configFile = detectPaceRc()
    myuser, mypwd, mydb, myhost = readConfigFile(configFile)
    dburl = 'mysql+pymysql://' + myuser + ':' + mypwd + '@' + myhost + '/' + mydb
    return dburl
```

**Database Tables Modified:**
1. `e3smexp` - Main experiment record
2. `exp` - Simplified experiment view
3. `pelayout` - PE layout configuration (multiple rows)
4. `runtime` - Component timing (multiple rows)
5. `model_timing` - Per-rank timing trees (multiple rows, one per rank + stats)
6. `memfile_inputs` - Memory profile data (optional)
7. `scorpio_stats` - I/O statistics (optional, multiple rows)
8. `xml_inputs` - XML configuration files (multiple rows)
9. `namelist_inputs` - Namelist files (multiple rows)
10. `rc_inputs` - RC configuration files (multiple rows)
11. `build_time` - Build timing (optional)
12. `preview_run` - Job submission parameters (optional)
13. `script_files` - Shell scripts (optional, multiple rows)

**Transaction Behavior:**
- All insertions for one experiment in single transaction
- Rollback on any error during experiment parsing
- Each experiment commits independently
- No cross-experiment transactions

**Duplicate Handling:**
- Duplicate check via query (user+machine+case+exp_date)
- Duplicate detection prevents insertion, returns early
- No database writes for duplicates

### 7.4 External Service Interactions

**Minio Object Storage:**
**Location:** `portal/pace/e3sm/e3smParser/parseE3SM.py:617-632`

**Purpose:** Archive experiment zip files to object storage

**Flow:**
```python
def uploadMinio(EXP_DIR, expname):
    myAkey, mySkey, myMiniourl = getMiniokey()  # From .pacerc
    minioClient = Minio(myMiniourl, access_key=myAkey, secret_key=mySkey)
    
    if not minioClient.bucket_exists("e3sm"):
        minioClient.make_bucket("e3sm")
    
    minioClient.fput_object('e3sm', expname+'.zip', os.path.join(EXP_DIR, expname)+'.zip')
```

**Bucket Structure:**
- Bucket name: `e3sm`
- Object name: `exp-{user}-{expid}.zip`
- Contains CaseDocs and related files

**Error Handling:**
- Failures logged but don't abort parsing
- Local filesystem copy remains in `EXP_DIR`

**Configuration:**
```ini
# .pacerc:
[MINIO]
minio_access_key = <key>
minio_secret_key = <secret>
minio_url = <url>
```

---

## 8. Transferable Patterns

### 8.1 Generally Reusable Patterns

#### Pattern 1: Two-Stage Ingestion
**Concept:** Separate upload from parsing

**Benefits:**
- Upload can fail/retry independently of parsing
- Parsing can be deferred or batched
- Client gets fast upload acknowledgement
- Heavy parsing doesn't block upload

**Applicability:** 
- ✅ Excellent for large file ingestion
- ✅ Works well with cron-driven automation
- ✅ Enables upload progress feedback

**SimBoard Consideration:**
Consider decoupling file upload from metadata extraction if SimBoard processes large files.

#### Pattern 2: Batch Processing with Continue-on-Error
**Concept:** Process multiple items, log failures, don't abort batch

**Implementation:**
```python
isSuccess = []
for item in batch:
    try:
        result = process(item)
        isSuccess.append(result)
    except:
        isSuccess.append(False)
        # Log error but continue

return 'fail' if False in isSuccess else 'success'
```

**Benefits:**
- One bad item doesn't kill entire batch
- User gets partial success feedback
- Maximizes data ingestion per run

**Applicability:**
- ✅ Ideal for cron jobs processing multiple experiments
- ✅ Reduces need for manual intervention

#### Pattern 3: Structured Logging with Redirection
**Concept:** Redirect stdout/stderr to per-upload log files

**Benefits:**
- All parsing output captured
- Client can download logs for debugging
- Server logs stay clean
- Audit trail per upload

**Applicability:**
- ✅ Good for async/background processing
- ✅ Essential for debugging failed ingestion

**SimBoard Consideration:**
If SimBoard parsing is async, structured logging is highly recommended.

#### Pattern 4: Filesystem Staging Area
**Concept:** Use filesystem as temporary workspace

**Benefits:**
- Decouples upload from parsing
- Allows inspection of raw data
- Enables cleanup after processing
- Supports reprocessing if needed

**Drawbacks:**
- Disk I/O overhead
- Cleanup required
- Not suitable for high-concurrency

**Applicability:**
- ✅ Good for batch processing
- ⚠️ May not scale to high request rates

#### Pattern 5: Duplicate Detection via Query
**Concept:** Check database for existing record before insert

**Implementation:**
```python
existing = query.filter_by(
    user=user,
    machine=machine,
    case=case,
    exp_date=date
).first()

if existing:
    return (True, existing.expid)
```

**Benefits:**
- Prevents duplicate data
- Idempotent ingestion (safe to re-run)
- Catches accidental re-uploads

**Applicability:**
- ✅ Essential for automated/cron-driven systems
- ✅ Highly reusable pattern

### 8.2 PACE-Specific Patterns (Less Transferable)

#### Pattern 1: Synchronous REST Parsing
**Issue:** Parsing happens in HTTP request handler

**PACE Implementation:**
- Works because cron jobs are patient
- Limited concurrency via Flask worker pool
- No progress updates during parsing

**Why Not Transfer:**
- Doesn't scale to interactive users
- Long HTTP timeouts
- Poor user experience for slow operations

**Better Alternative for SimBoard:**
- Upload endpoint returns immediately with job ID
- Background worker processes parsing
- Status endpoint for progress polling
- Webhook or notification on completion

#### Pattern 2: GitHub OAuth for Automation
**Issue:** Uses GitHub personal access tokens for non-interactive auth

**PACE Implementation:**
- Client stores GitHub token in `~/.pacecc`
- Token validated via GitHub API
- Username extracted from token
- User validated against whitelist

**Why Not Transfer:**
- GitHub-specific
- Not suited for general API authentication
- No fine-grained permissions

**Better Alternative for SimBoard:**
- API keys or service tokens
- JWT tokens for stateless auth
- OAuth2 client credentials flow

#### Pattern 3: Monolithic Parser Module
**Issue:** Single large parseE3SM.py file with many responsibilities

**PACE Implementation:**
- One function orchestrates all parsing
- Many parser imports at top
- All database logic in same file

**Why Not Transfer:**
- Hard to test
- Difficult to extend
- Tight coupling between parsers

**Better Alternative for SimBoard:**
- Plugin architecture for parsers
- Dependency injection
- Separate orchestration from parsing logic

#### Pattern 4: File-Based Parser Discovery
**Issue:** Parsers triggered by filename patterns

**PACE Implementation:**
```python
if name.startswith("timing."):
    model['timingfile'] = os.path.join(path, name)
elif name.startswith("e3sm_timing."):
    model['allfile'] = os.path.join(path, name)
```

**Why Not Transfer:**
- Brittle (filename changes break system)
- No metadata about what files exist
- Hard to extend to new file types

**Better Alternative for SimBoard:**
- Manifest file describing uploaded content
- Content-type based routing
- Explicit file type declaration in API

#### Pattern 5: Tarball-in-Zip Nesting
**Issue:** Requires extracting zip, then extracting tarballs

**PACE Implementation:**
- Client zips multiple experiments
- Each experiment may contain tar.gz files
- Server extracts both layers

**Why Not Transfer:**
- Double extraction overhead
- Complexity for no clear benefit
- Security risk (tarfile path traversal)

**Better Alternative for SimBoard:**
- Single archive format
- Streaming extraction
- Validate archive structure before extraction

---

## 9. SimBoard Relevance

### 9.1 How This Flow Could Be Mimicked

SimBoard already has REST APIs (built with FastAPI). Here's how to adapt PACE's patterns:

#### Step 1: Two-Stage API Design
**PACE Equivalent:** `/upload` + `/fileparse`

**SimBoard Implementation:**
```python
# FastAPI (SimBoard style)
@app.post("/api/v1/upload")
async def upload_experiment(file: UploadFile = File(...)):
    """Stage uploaded file, return upload ID"""
    upload_id = uuid.uuid4()
    staging_path = STAGING_DIR / f"{upload_id}_{file.filename}"
    
    async with aiofiles.open(staging_path, 'wb') as f:
        await f.write(await file.read())
    
    return {"upload_id": upload_id, "status": "staged"}

@app.post("/api/v1/parse/{upload_id}")
async def trigger_parse(upload_id: str, background_tasks: BackgroundTasks):
    """Trigger parsing of staged upload"""
    background_tasks.add_task(parse_experiment, upload_id)
    return {"job_id": upload_id, "status": "parsing"}
```

**Improvements over PACE:**
- Async file I/O (aiofiles)
- Background task processing (doesn't block request)
- UUIDs instead of timestamp-based IDs

#### Step 2: Background Task Processing
**PACE Gap:** Synchronous parsing blocks HTTP handler

**SimBoard Implementation:**
```python
async def parse_experiment(upload_id: str):
    """Background task for parsing"""
    try:
        # Extract archive
        extract_path = await extract_archive(upload_id)
        
        # Discover experiment files
        experiments = await discover_experiments(extract_path)
        
        # Parse each experiment
        for exp in experiments:
            await parse_single_experiment(exp)
        
        # Update job status
        await update_job_status(upload_id, "complete")
    except Exception as e:
        await update_job_status(upload_id, "failed", str(e))
```

**Benefits:**
- Non-blocking
- Progress updates possible
- Better error handling

#### Step 3: Job Status Endpoint
**PACE Gap:** No way to check parsing status while it runs

**SimBoard Implementation:**
```python
@app.get("/api/v1/parse/{job_id}/status")
async def get_parse_status(job_id: str):
    """Get status of parsing job"""
    job = await db.get_job(job_id)
    return {
        "job_id": job_id,
        "status": job.status,  # "pending", "parsing", "complete", "failed"
        "progress": job.progress,  # e.g., "3/5 experiments parsed"
        "log_url": f"/api/v1/parse/{job_id}/log"
    }
```

#### Step 4: Structured Log Retrieval
**PACE Approach:** Download entire log file

**SimBoard Implementation:**
```python
@app.get("/api/v1/parse/{job_id}/log")
async def get_parse_log(job_id: str, level: str = "INFO"):
    """Get structured log entries"""
    logs = await db.get_logs(job_id, level)
    return {
        "job_id": job_id,
        "entries": [
            {"timestamp": log.timestamp, "level": log.level, "message": log.message}
            for log in logs
        ]
    }
```

### 9.2 Patterns Worth Adopting

#### ✅ 1. Duplicate Detection
**Why:** Prevents duplicate ingestion from automation

**Implementation:**
```python
async def check_duplicate_experiment(
    user: str, machine: str, case: str, exp_date: datetime
) -> Optional[int]:
    """Check if experiment already exists"""
    existing = await db.experiments.find_one({
        "user": user,
        "machine": machine,
        "case": case,
        "exp_date": exp_date
    })
    return existing["exp_id"] if existing else None
```

#### ✅ 2. Batch Processing with Continue-on-Error
**Why:** Maximizes data ingestion, reduces manual intervention

**Implementation:**
```python
async def parse_batch(experiments: List[Path]) -> Dict[str, Any]:
    """Parse multiple experiments, continue on error"""
    results = {"success": [], "failed": []}
    
    for exp in experiments:
        try:
            exp_id = await parse_experiment(exp)
            results["success"].append(exp_id)
        except Exception as e:
            results["failed"].append({"path": str(exp), "error": str(e)})
            logger.error(f"Failed to parse {exp}: {e}")
    
    return results
```

#### ✅ 3. Per-Upload Logging
**Why:** Essential for debugging, audit trail

**Implementation:**
```python
import structlog

logger = structlog.get_logger()

async def parse_experiment(upload_id: str):
    """Parse with structured logging"""
    log = logger.bind(upload_id=upload_id)
    
    log.info("parse_started")
    try:
        exp_id = await do_parsing(upload_id)
        log.info("parse_complete", exp_id=exp_id)
    except Exception as e:
        log.error("parse_failed", error=str(e))
        raise
```

#### ✅ 4. Filesystem Staging
**Why:** Decouples upload from parsing, enables retry

**Consideration:** May want to add TTL-based cleanup

**Implementation:**
```python
async def cleanup_staging_area():
    """Remove old staged files"""
    cutoff = datetime.now() - timedelta(days=7)
    
    for file in STAGING_DIR.glob("*"):
        if file.stat().st_mtime < cutoff.timestamp():
            file.unlink()
```

### 9.3 Patterns to Avoid

#### ❌ 1. Synchronous REST Parsing
**Why:** Blocks HTTP workers, poor scalability

**Alternative:** FastAPI background tasks or Celery

#### ❌ 2. GitHub OAuth for Service Auth
**Why:** Not designed for machine-to-machine

**Alternative:** API keys, service tokens, OAuth2 client credentials

#### ❌ 3. Filename-Based Parser Discovery
**Why:** Brittle, hard to extend

**Alternative:** Manifest file or content-type headers

#### ❌ 4. Stdout Redirection for Logging
**Why:** Not compatible with async, structured logging better

**Alternative:** structlog, Python logging with handlers

#### ❌ 5. Global State (sys.stdout redirect)
**Why:** Thread-unsafe, affects entire process

**Alternative:** Context-aware logging with correlation IDs

### 9.4 High-Level SimBoard Adaptation

**Recommended Architecture:**

```
┌─────────────┐
│   Client    │ (Cron job on HPC)
│  (curl/CLI) │
└──────┬──────┘
       │ POST /api/v1/upload (multipart file)
       ▼
┌──────────────────────────────────┐
│   FastAPI REST Layer             │
│  - Authentication (API key)      │
│  - Validation                    │
│  - File staging                  │
└──────────┬───────────────────────┘
       │ Returns: {"upload_id": "...", "status": "staged"}
       │
       │ POST /api/v1/parse/{upload_id}
       ▼
┌──────────────────────────────────┐
│   Background Task Queue          │
│  - Celery or FastAPI background  │
│  - Parallel processing           │
│  - Progress tracking             │
└──────────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│   Parser Orchestrator            │
│  - Extract archive               │
│  - Discover experiments          │
│  - Route to parsers              │
└──────────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│   Parser Plugins                 │
│  - Timing parser                 │
│  - Metadata parser               │
│  - Config parser                 │
└──────────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│   Database Layer                 │
│  - Duplicate check               │
│  - Batch insert                  │
│  - Transaction management        │
└──────────────────────────────────┘
```

**Key Differences from PACE:**
1. **Async everywhere** (FastAPI native)
2. **Background tasks** (non-blocking)
3. **Status polling** (GET /parse/{job_id}/status)
4. **Structured logging** (JSON logs, correlation IDs)
5. **Plugin architecture** (extensible parsers)
6. **API key auth** (not GitHub OAuth)

### 9.5 Specific Recommendations

#### For Cron-Driven Ingestion:
1. ✅ Keep two-stage upload/parse flow
2. ✅ Implement duplicate detection
3. ✅ Use batch processing with continue-on-error
4. ✅ Provide log download endpoint
5. ✅ Return job ID immediately, parse asynchronously

#### For Scalability:
1. ✅ Use async file I/O (aiofiles)
2. ✅ Background task processing (Celery or FastAPI)
3. ✅ Parallel parsing of experiments
4. ✅ Database connection pooling
5. ✅ Object storage for archives (S3-compatible)

#### For Reliability:
1. ✅ Idempotent operations (duplicate detection)
2. ✅ Transaction per experiment (independent failures)
3. ✅ Structured error logging
4. ✅ Retry logic for transient failures
5. ✅ Health check endpoints

#### For Developer Experience:
1. ✅ OpenAPI/Swagger docs (FastAPI auto-generates)
2. ✅ Structured logging with correlation IDs
3. ✅ Clear error messages
4. ✅ Progress indicators
5. ✅ Webhooks for completion notifications (optional)

---

## Appendix: File Locations Quick Reference

### REST Endpoints
- **Upload:** `portal/pace/webapp.py:96-115`
- **Parse:** `portal/pace/webapp.py:117-131`
- **Download Log:** `portal/pace/webapp.py:133-145`

### Orchestration
- **Main Router:** `portal/pace/parse.py:21-47`
- **E3SM Parser:** `portal/pace/e3sm/e3smParser/parseE3SM.py:65-693`

### Individual Parsers
- **E3SM Timing:** `portal/pace/e3sm/e3smParser/parseE3SMTiming.py`
- **Model Timing:** `portal/pace/e3sm/e3smParser/parseModelTiming.py`
- **README:** `portal/pace/e3sm/e3smParser/parseReadMe.py`
- **Version:** `portal/pace/e3sm/e3smParser/parseModelVersion.py`
- **Memory:** `portal/pace/e3sm/e3smParser/parseMemoryProfile.py`
- **SCORPIO:** `portal/pace/e3sm/e3smParser/parseScorpioStats.py`
- **CaseDocs:** `portal/pace/e3sm/e3smParser/parseCaseDocs.py`
- **Build Time:** `portal/pace/e3sm/e3smParser/parseBuildTime.py`
- **Preview Run:** `portal/pace/e3sm/e3smParser/parsePreviewRun.py`
- **Scripts:** `portal/pace/e3sm/e3smParser/parseReplaysh.py`, `parseRunE3SMsh.py`

### Configuration
- **Common Functions:** `portal/pace/pace_common.py`
- **Config Template:** `etc/pacerc.tmpl`

### Client
- **Upload Tool:** `client/pace-upload`
- **Client README:** `client/README.md`

### Database
- **ORM Models:** `portal/pace/e3sm/e3smDb/datastructs.py`

---

## Conclusion

PACE's REST-to-parser flow is optimized for **automated, cron-driven ingestion** from HPC systems. The two-stage upload/parse model, duplicate detection, and continue-on-error batch processing are excellent patterns for SimBoard to adopt.

However, PACE's synchronous execution and GitHub-based auth are less suitable for modern web applications. SimBoard should prefer:
- **Async/background processing** for scalability
- **API key authentication** for machine-to-machine
- **Status polling endpoints** for progress visibility
- **Structured logging** for debugging

The core orchestration pattern (discover files → route to parsers → batch insert) is highly transferable and provides a solid foundation for SimBoard's ingestion pipeline.

---

**End of Analysis**
