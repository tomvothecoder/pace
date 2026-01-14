# PACE E3SM Simulation Metadata Parsing Analysis

**Date:** 2026-01-14  
**Purpose:** Analyze PACE's E3SM simulation metadata parsing logic for reimplementation in SimBoard  
**Scope:** Parser logic only (no Flask web app, database ORM, or authentication)

---

## 1. Parsing Entrypoints

### 1.1 Main Entry Function

**Function:** `parseData(zipfilename, uploaduser)` in `portal/pace/e3sm/e3smParser/parseE3SM.py`

**Responsibilities:**
- Extracts uploaded zip/tar.gz archives
- Discovers experiment directories (prefixed with `exp`)
- Identifies required files within each experiment
- Orchestrates parsing of all file types
- Handles database insertion and error management

### 1.2 Top-Level Dispatcher

**Function:** `parseData(zipfilename, uploaduser, project)` in `portal/pace/parse.py`

**Responsibilities:**
- Entry point from web application
- Routes to E3SM-specific parser based on `project` parameter
- Sets up logging infrastructure
- Returns status and log filename

### 1.3 Case Directory Discovery

**Algorithm (in parseE3SM.py lines 65-96):**

1. Extract zip file to upload folder
2. Extract any `.tar.gz` files found
3. Look for directories starting with `exp` (e.g., `exp-username-12345`)
4. Walk through each experiment directory to locate specific files

### 1.4 File Discovery Pattern

The parser walks each experiment directory looking for files with specific prefixes (lines 114-139):

```python
Expected file patterns:
- timing.*                  → timingfile
- e3sm_timing.*            → allfile
- README.case.*            → readmefile
- GIT_DESCRIBE.*           → gitdescribefile
- spio_stats.*             → scorpiofile
- memory.*                 → memoryfile
- build_times.txt.*        → buildtimefile
- preview_run.log.*        → previewrunfile
- replay.sh.*              → replayshfile
- run_e3sm.sh.*            → run_e3sm_file
- CaseDocs.*/ (directory)  → casedocs
```

---

## 2. Input Artifacts

### 2.1 Required Files

**Critical Files (parsing fails without these):**
- `e3sm_timing.*` - Main timing summary file
- `README.case.*` - Case creation metadata
- `GIT_DESCRIBE.*` - E3SM version information

**Optional Files (gracefully skipped if missing):**
- `timing.*` - Detailed GPTL timing data (tar archive)
- `spio_stats.*` - Scorpio I/O statistics (tar.gz containing JSON files)
- `memory.*` - Memory profiling data (CSV format)
- `build_times.txt.*` - Build time breakdown
- `preview_run.log.*` - Job submission preview information
- `replay.sh.*` - Replay script
- `run_e3sm.sh.*` - Run script
- `CaseDocs.*/` - Configuration files directory

### 2.2 Expected Directory Layout

```
upload_folder/
└── experiment_name/         # e.g., "exp-username-12345"
    ├── e3sm_timing.case_name.LID
    ├── README.case.LID.gz
    ├── GIT_DESCRIBE.LID.gz
    ├── timing.case_name.LID.tar.gz
    ├── spio_stats.LID.tar.gz
    ├── memory.N.M.log.LID.gz
    ├── build_times.txt.LID.gz
    ├── preview_run.log.LID.gz
    ├── replay.sh.LID.gz
    ├── run_e3sm.sh.timestamp.LID.gz
    └── CaseDocs.LID/
        ├── env_case.xml.LID.gz
        ├── env_build.xml.LID.gz
        ├── env_run.xml.LID.gz
        ├── env_batch.xml.LID.gz
        ├── env_mach_pes.xml.LID.gz
        ├── env_mach_specific.xml.LID.gz
        ├── env_archive.xml.LID.gz
        ├── env_workflow.xml.LID.gz
        ├── namelist_scream.xml.LID.gz  (optional)
        ├── atm_in.*.gz
        ├── lnd_in.*.gz
        ├── ice_in.*.gz
        ├── ocn_in.*.gz  (or mpaso_in, mpassi_in)
        ├── drv_in.*.gz
        ├── mosart_in.*.gz
        ├── user_nl_*.*.gz  (multiple files)
        ├── *_modelio.nml.*.gz  (multiple files)
        └── seq_maps.rc.*.gz
```

**Notes:**
- `LID` = Local ID timestamp (e.g., `43235257.210608-222102`)
- Most files are gzipped (`.gz` extension)
- Files may or may not be compressed depending on upload process
- CaseDocs directory name includes LID: `CaseDocs.{LID}`

### 2.3 File Format Details

| File Type | Format | Compression | Parser Module |
|-----------|--------|-------------|---------------|
| e3sm_timing | Text | Optional .gz | parseE3SMTiming.py |
| README.case | Text | .gz | parseReadMe.py |
| GIT_DESCRIBE | Text | .gz | parseModelVersion.py |
| timing | Tar archive | .tar.gz | parseModelTiming.py |
| spio_stats | Tar w/ JSON files | .tar.gz | parseScorpioStats.py |
| memory | CSV | .gz | parseMemoryProfile.py |
| build_times.txt | Text (tabular) | .gz | parseBuildTime.py |
| preview_run.log | Text | .gz | parsePreviewRun.py |
| replay.sh | Bash script | .gz | parseReplaysh.py |
| run_e3sm.sh | Bash script | .gz | parseRunE3SMsh.py |
| XML files | XML | .gz | parseXML.py |
| Namelist files | Fortran namelist | .gz | parseNameList.py |
| RC files | Key:value pairs | .gz | parseRC.py |
| user_nl_* | Text | .gz | parseText.py |

---

## 3. Metadata Extraction

### 3.1 Core Experiment Metadata (from e3sm_timing.*)

**Source:** `parseE3SMTiming.parseE3SMtiming(filename)`  
**File:** `portal/pace/e3sm/e3smParser/parseE3SMTiming.py`

**Extracted Fields:**

| Field | Source Pattern | Description |
|-------|----------------|-------------|
| case | `Case:` | Case name |
| lid | `LID:` | Local ID (timestamp) |
| machine | `Machine:` | HPC machine name |
| caseroot | `Caseroot:` | Case directory path |
| timeroot | `Timeroot:` | Timing tools directory |
| user | `User:` | Username who ran simulation |
| curr | `Curr Date:` | Current date/time of run |
| long_res | `grid:` | Full grid specification |
| long_compset | `compset:` | Full component set |
| stop_option | `stop option:` | Stop criterion (e.g., "ndays") |
| stop_n | `stop_n =` | Stop value |
| run_length | `run length:` | Duration in days |
| total_pes | `total pes active:` | Total active PEs |
| mpi_task | `mpi tasks per node:` | MPI tasks per node |
| pe_count | `pe count for cost estimate:` | PE count for cost |
| model_cost | `Model Cost:` | Cost in pe-hrs/simulated_year |
| model_throughput | `Model Throughput:` | Simulated years/day |
| actual_ocn | `Actual Ocn Init Wait Time:` | Ocean init wait time |
| init_time | `Init Time:` | Initialization time (seconds) |
| run_time | `Run Time:` | Run time (seconds) |
| final_time | `Final Time:` | Finalization time (seconds) |

**Component Table (PE Layout):**

Extracted from section starting with `component comp_pes root_pe tasks...`

For each component (cpl, atm, lnd, ice, ocn, rof, glc, wav, iac, esp):
- component name
- comp_pes
- root_pe
- tasks
- threads
- instances
- stride

**Runtime Table:**

Extracted from section starting with `TOT Run Time:`

For each component and its COMM counterpart (11 entries):
- component name (e.g., "TOT", "ATM", "ATM_COMM")
- seconds
- model_day (seconds/model-day)
- model_years (model-years/wall-day)

### 3.2 Case Creation Metadata (from README.case.*)

**Source:** `parseReadMe.parseReadme(readmefilename)`  
**File:** `portal/pace/e3sm/e3smParser/parseReadMe.py`

**Extracted Fields:**

| Field | Source | Description |
|-------|--------|-------------|
| res | `--res` argument | Resolution (e.g., "ne30_ne30") |
| compset | `--compset` argument | Component set (e.g., "F2010") |
| date | Line prefix | Creation timestamp |

**Parsing Logic:**
- Searches for line containing `create_newcase`
- Extracts command-line arguments
- Handles both `--arg=value` and `--arg value` formats

### 3.3 Version Information (from GIT_DESCRIBE.*)

**Source:** `parseModelVersion.parseModelVersion(gitfile)`  
**File:** `portal/pace/e3sm/e3smParser/parseModelVersion.py`

**Extracted Field:**
- version: First non-empty line (e.g., "v2.0.0-beta.3-3091-g3219b44fc")

### 3.4 Model Timing Data (from timing.*)

**Source:** `parseModelTiming.parse(source)`  
**File:** `portal/pace/e3sm/e3smParser/parseModelTiming.py`

**Format:** Tar archive containing GPTL (General Purpose Timing Library) output files

**Extracted per rank/thread:**
- Hierarchical timing tree structure (JSON)
- For each timer:
  - name
  - on (boolean, if active)
  - called, recurse counts
  - wallClock, max, min times
  - processes, threads, count
  - walltotal, wallmax, wallmin
  - multiParent flag

**Parser Configurations:**

Three GPTL format versions supported:
1. Old format: `Stats for thread` files
2. GPTL v1: `GLOBAL STATISTICS` without 'on' column
3. GPTL v2: `GLOBAL STATISTICS` with 'on' column

### 3.5 Scorpio I/O Statistics (from spio_stats.*)

**Source:** `parseScorpioStats.loaddb_scorpio_stats(spiofile, runTime)`  
**File:** `portal/pace/e3sm/e3smParser/parseScorpioStats.py`

**Format:** Tar.gz archive containing JSON files (one per component)

**Extracted per component:**
- name: Component name (derived from max tot_time model)
- data: Raw JSON data
- iopercent: I/O time as % of total runtime
- iotime: Total I/O time in seconds
- version: Scorpio stats version

**Logic:**
- Extracts JSON files from tar archive
- Identifies component with maximum tot_time
- Calculates I/O percentage relative to run_time
- Filters invalid entries (iopercent > 100, old unsupported versions)

### 3.6 Memory Profile (from memory.*)

**Source:** `parseMemoryProfile.loaddb_memfile(memfile)`  
**File:** `portal/pace/e3sm/e3smParser/parseMemoryProfile.py`

**Format:** CSV data (gzipped)

**Extraction:** Stores entire CSV as-is (no parsing)

### 3.7 Build Times (from build_times.txt.*)

**Source:** `parseBuildTime.loaddb_buildTimesFile(buildfile)`  
**File:** `portal/pace/e3sm/e3smParser/parseBuildTime.py`

**Format:** Tab-separated values

**Extracted:**
- data: Dictionary of component → elapsed time
- total_computecost: Sum of all build times (excluding Total_Elapsed_Time)
- total_elapsed_time: Overall build duration

**Logic:**
- Skips header line
- Aggregates repeated component names
- Separates Total_Elapsed_Time from compute cost

### 3.8 Preview Run Information (from preview_run.log.*)

**Source:** `parsePreviewRun.load_previewRunFile(previewfile)`  
**File:** `portal/pace/e3sm/e3smParser/parsePreviewRun.py`

**Extracted Fields:**

| Field | Source Pattern | Description |
|-------|---------------|-------------|
| nodes | `nodes:` | Number of nodes |
| total_tasks | `total tasks:` | Total MPI tasks |
| tasks_per_node | `tasks per node:` | Tasks per node |
| thread_count | `thread count:` | Thread count |
| ngpus_per_node | `ngpus per node:` | GPUs per node |
| env | `ENV:` section | Environment variables dict |
| omp_threads | From env['OMP_NUM_THREADS'] | OpenMP threads |
| submit_cmd | `SUBMIT CMD:` | Job submission command |
| mpirun | `MPIRUN CMD:` | MPI run command |

**Logic:**
- State machine parsing with flags
- Reads next line after marker
- Parses environment variable assignments

### 3.9 Script Files (replay.sh.*, run_e3sm.sh.*)

**Source:** `parseReplaysh.load_replayshFile()`, `parseRunE3SMsh.load_rune3smshfile()`  
**Files:** `portal/pace/e3sm/e3smParser/parseReplaysh.py`, `parseRunE3SMsh.py`

**Extraction:** Stores entire script content as-is

### 3.10 CaseDocs Configuration Files

**Source:** `parseCaseDocs.loaddb_casedocs(casedocpath, db, currExpObj)`  
**File:** `portal/pace/e3sm/e3smParser/parseCaseDocs.py`

**Categories:**

#### A. XML Files (env_*.xml, namelist_scream.xml)

**Parser:** `parseXML.loaddb_xmlfile(xmlpath)`

**Accepted prefixes:**
- env_archive, env_batch, env_build, env_case
- env_mach_pes, env_mach_specific, env_run, env_workflow
- namelist_scream

**Format:** XML → JSON via xmltodict

**Special Processing:**

1. **env_case.xml:**
   - Extracts `case_group` from `CASE_GROUP` entry
   - Updates E3SMexp.case_group field

2. **env_build.xml:**
   - Extracts `compiler` from `COMPILER` entry
   - Extracts `mpilib` from `MPILIB` entry
   - Updates E3SMexp.compiler and E3SMexp.mpilib fields

#### B. Namelist Files

**Parser:** `parseNameList.loaddb_namelist(nmlpath)`

**Accepted prefixes:**
- atm_in, atm_modelio, cpl_modelio, drv_flds_in, drv_in
- esp_modelio, glc_modelio, ice_modelio, lnd_in, lnd_modelio
- mosart_in, mpaso_in, mpassi_in, ocn_modelio, rof_modelio
- user_nl_cam, user_nl_clm, user_nl_cpl, user_nl_mosart
- user_nl_mpascice, user_nl_mpaso, wav_modelio, iac_modelio
- docn_in, user_nl_docn, user_nl_cice, ice_in
- user_nl_elm, user_nl_eam, user_nl_mali, mali_in

**Format:**
- Most: Fortran namelist → JSON via f90nml library
- user_nl_*: Plain text (via parseText.py)

#### C. RC Files (seq_maps.rc.*)

**Parser:** `parseRC.loaddb_rcfile(rcpath)`

**Format:** Key:value pairs → JSON object

---

## 4. Control Flow

### 4.1 Overall Parsing Sequence

**Order of operations (from parseE3SM.py:insertExperiment):**

1. **Parse e3sm_timing file** (`insertE3SMTiming`)
   - Parses e3sm_timing.*
   - Parses README.case.*
   - Parses GIT_DESCRIBE.*
   - Checks for duplicates (user + machine + exp_date + case)
   - Creates E3SMexp, Exp, Pelayout, Runtime records
   - Returns early if duplicate found

2. **Parse model timing** (`insertTiming`)
   - Extracts files from timing.* tar archive
   - Parses GPTL output per rank
   - Creates ModelTiming records

3. **Parse memory file** (`insertMemoryFile`)
   - Stores CSV data
   - Creates MemfileInputs record

4. **Parse Scorpio stats** (`insertScorpioStats`)
   - Extracts JSON files from tar.gz
   - Processes each component
   - Creates ScorpioStats records

5. **Parse CaseDocs** (`parseCaseDocs.loaddb_casedocs`)
   - Iterates through all files in directory
   - Routes to appropriate parser based on prefix
   - Extracts case_group, compiler, mpilib
   - Creates NamelistInputs, XMLInputs, RCInputs records

6. **Parse build times** (`insertBuildTimeFile`)
   - Creates BuildTime record

7. **Parse preview run** (`insertPreviewRunFile`)
   - Creates PreviewRun record

8. **Parse scripts** (`insertScripts`)
   - Parses replay.sh and run_e3sm.sh
   - Creates ScriptsFile records

9. **Archive raw data**
   - Zips experiment directory
   - Uploads to MinIO object storage

10. **Commit to database**
    - Single transaction commit
    - Rollback on any error

### 4.2 Error Handling Behavior

**Strategy:** Fail-safe with skip-on-error

- **Missing optional files:** Prints warning, continues parsing
- **Parse errors:** Prints error, returns False, skips experiment
- **Duplicate experiments:** Prints note, returns True, skips experiment
- **Database errors:** Prints error, rolls back transaction, returns False

**Error Messages Format:**
```
print("Empty memory profile file")              # Warning
print("ERROR: %s" % e)                          # Error
print("Insertion is discarded due to duplication")  # Info
```

**Return Values:**
- `True` → Success or duplicate (skip remaining files)
- `False` → Failure (abort this experiment)

### 4.3 Optional vs Required Metadata

**Required (parsing fails without these):**
- e3sm_timing.* file
- README.case.* file
- GIT_DESCRIBE.* file
- Valid experiment metadata (case, user, machine, etc.)

**Optional (gracefully skipped):**
- timing.* (model timing)
- spio_stats.* (Scorpio I/O stats)
- memory.* (memory profile)
- build_times.txt.* (build times)
- preview_run.log.* (preview run info)
- replay.sh.* (replay script)
- run_e3sm.sh.* (run script)
- Any CaseDocs files

**Behavior on Missing Files:**
- Parser checks `if file:` before calling parse function
- Prints `"No <file_type> file"` message
- Returns `True` to continue processing

---

## 5. Implicit Assumptions

### 5.1 HPC-Specific Paths

**Assumptions about caseroot and timeroot:**
- Paths are stored as-is from e3sm_timing file
- Examples: `/global/cscratch1/sd/<user>/<case>`
- No validation or normalization performed
- Assumes Unix-style paths

### 5.2 Machine/Compiler Assumptions

**Supported machines (examples from data):**
- cori-knl
- cori-haswell
- Other NERSC/DOE machines

**Compilers (from env_build.xml):**
- intel, gnu, etc.

**No validation** - stores whatever is in the files

### 5.3 E3SM Version Coupling

**Version String Format:**
- Expects Git describe format: `v{major}.{minor}.{patch}-{commits}-g{hash}`
- Example: `v2.0.0-beta.3-3091-g3219b44fc`
- No version-specific parsing logic (version-agnostic)

**GPTL Format Versions:**
- Parser auto-detects three GPTL output formats
- Backwards compatible with old formats

**Scorpio Stats Versions:**
- Checks for `spio_stats_version` field
- Filters out very old unsupported formats
- Limits to 10 old format files per experiment

### 5.4 File Naming Conventions

**Implicit patterns:**
- Files use `.` as separator: `{prefix}.{case}.{LID}`
- LID format: `{numbers}.{YYMMDD-HHMMSS}`
- Gzipped files end with `.gz`
- Tar archives use `.tar.gz`
- CaseDocs directory: `CaseDocs.{LID}`

### 5.5 Component Names

**Expected components (from PE layout table):**
- cpl (coupler)
- atm (atmosphere): eam, cam
- lnd (land): elm, clm
- ice (sea ice): cice, mpassi
- ocn (ocean): docn, mpaso
- rof (runoff): mosart
- glc (land ice): sglc, mali
- wav (wave): swav
- iac (integrated assessment): siac
- esp (earth system processes): sesp

### 5.6 Date/Time Format

**Expected format (from e3sm_timing):**
```
Curr Date: Day Mon DD HH:MM:SS YYYY
Example: Wed Jun  9 01:07:55 2021
```

**Conversion function:** `changeDateTime(c_date)` in parseE3SM.py
- Converts to SQL format: `YYYY-MM-DD HH:MM:SS`

### 5.7 Database Dependencies

**Not relevant for SimBoard** - but current code assumes:
- MySQL/MariaDB (LONGTEXT, MEDIUMTEXT, DECIMAL types)
- SQLAlchemy ORM
- Single database transaction per experiment
- Auto-incrementing expid

---

## 6. Suggested Neutral Schema

### 6.1 Design Principles

For SimBoard reimplementation, suggest:

1. **Flat JSON structure** for core metadata
2. **Nested objects** for component-specific data
3. **Arrays** for tabular data (PE layout, runtime)
4. **Raw strings** for file contents (scripts, memory CSV)
5. **Optional fields** clearly marked (nullable)
6. **Version field** to allow future schema evolution

### 6.2 Core Experiment Schema

```json
{
  "schema_version": "1.0",
  "experiment": {
    "case": "e3sm_v1.2_ne30_noAgg-60",
    "lid": "43235257.210608-222102",
    "machine": "cori-knl",
    "caseroot": "/global/cscratch1/sd/blazg/e3sm_scratch/e3sm_v1.2_ne30_noAgg-60",
    "timeroot": "/global/cscratch1/sd/blazg/e3sm_scratch/e3sm_v1.2_ne30_noAgg-60/Tools",
    "user": "blazg",
    "exp_date": "2021-06-09T01:07:55Z",
    "upload_by": "simboard_user",
    "upload_date": "2026-01-14T22:32:22Z",
    
    "resolution": {
      "short": "ne30_ne30",
      "long": "a%ne30np4_l%ne30np4_oi%ne30np4_r%r05_g%null_w%null_z%null_m%gx1v6"
    },
    
    "compset": {
      "short": "F2010",
      "long": "2010_EAM%CMIP6_ELM%SPBC_CICE%PRES_DOCN%DOM_MOSART_SGLC_SWAV_SIAC_SESP"
    },
    
    "case_group": "my_experiment_group",
    "compiler": "intel",
    "mpilib": "mvapich2",
    "version": "v2.0.0-beta.3-3091-g3219b44fc",
    
    "run_config": {
      "stop_option": "nmonths",
      "stop_n": 6,
      "run_length_days": 184,
      "run_type": "startup",
      "continue_run": true
    },
    
    "performance": {
      "total_pes_active": 5400,
      "mpi_tasks_per_node": 32,
      "pe_count_for_cost": 1376,
      "model_cost_pe_hrs_per_sim_year": 7473.60,
      "model_throughput_sim_years_per_day": 4.42,
      "init_time_seconds": 81.132,
      "run_time_seconds": 9856.861,
      "final_time_seconds": 0.325,
      "actual_ocn_init_wait_time_seconds": 39.855
    }
  },
  
  "pe_layout": [
    {
      "component": "cpl",
      "comp_pes": 5400,
      "root_pe": 0,
      "tasks": 1350,
      "threads": 4,
      "instances": 1,
      "stride": 1
    },
    {
      "component": "atm",
      "comp_pes": 5400,
      "root_pe": 0,
      "tasks": 1350,
      "threads": 4,
      "instances": 1,
      "stride": 1
    }
    // ... additional components
  ],
  
  "component_runtime": [
    {
      "component": "TOT",
      "seconds": 9856.861,
      "seconds_per_model_day": 53.570,
      "model_years_per_wall_day": 4.42
    },
    {
      "component": "ATM",
      "seconds": 8757.355,
      "seconds_per_model_day": 47.594,
      "model_years_per_wall_day": 4.97
    }
    // ... additional components
  ],
  
  "job_config": {
    "nodes": 169,
    "total_tasks": 5400,
    "tasks_per_node": 32,
    "thread_count": 4,
    "ngpus_per_node": 0,
    "omp_threads": 4,
    "mpirun_cmd": "srun -n 5400 --cpu-bind=cores",
    "submit_cmd": "sbatch run.sh",
    "environment": {
      "OMP_NUM_THREADS": "4",
      "OMP_PLACES": "cores"
      // ... additional env vars
    }
  },
  
  "build_info": {
    "total_compute_cost_seconds": 1234.56,
    "total_elapsed_time_seconds": 234.56,
    "component_times": {
      "cam": 500.0,
      "clm": 300.0,
      "cice": 100.0
      // ... additional components
    }
  },
  
  "io_stats": {
    "scorpio": [
      {
        "name": "ATM-pe000000",
        "iotime_seconds": 123.45,
        "iopercent": 1.25,
        "version": "2.5",
        "raw_data": { /* full JSON from spio_stats */ }
      }
      // ... additional components
    ]
  },
  
  "timing_profiles": [
    {
      "rank": "0",
      "data": [ /* hierarchical timing tree JSON */ ]
    },
    {
      "rank": "stats",
      "data": [ /* aggregate statistics */ ]
    }
    // ... additional ranks
  ],
  
  "configuration": {
    "namelists": {
      "atm_in": { /* parsed namelist JSON */ },
      "lnd_in": { /* parsed namelist JSON */ },
      "user_nl_cam": "! Custom atmospheric namelist\nnco_opt = 'fv'\n"
      // ... additional namelists
    },
    
    "xml_configs": {
      "env_case": { /* parsed XML → JSON */ },
      "env_build": { /* parsed XML → JSON */ },
      "env_run": { /* parsed XML → JSON */ }
      // ... additional XML files
    },
    
    "rc_files": {
      "seq_maps": { /* parsed RC file → JSON */ }
    }
  },
  
  "scripts": {
    "replay_sh": "#!/bin/bash\n# Replay script content...",
    "run_e3sm_sh": "#!/bin/bash\n# Run script content..."
  },
  
  "diagnostics": {
    "memory_profile_csv": "Time,Rank,RSS,VMS\n0.0,0,1234,5678\n...",
    "memory_profile_format": "csv"
  }
}
```

### 6.3 Schema Field Descriptions

| Section | Field | Type | Source | Required | Notes |
|---------|-------|------|--------|----------|-------|
| experiment | case | string | e3sm_timing | Yes | Case name |
| experiment | lid | string | e3sm_timing | Yes | Local ID timestamp |
| experiment | machine | string | e3sm_timing | Yes | HPC machine identifier |
| experiment | user | string | e3sm_timing | Yes | Username |
| experiment | exp_date | ISO8601 | e3sm_timing (curr) | Yes | Experiment date/time |
| experiment | resolution.short | string | README.case (--res) | Yes | Short resolution |
| experiment | resolution.long | string | e3sm_timing (grid) | Yes | Full grid spec |
| experiment | compset.short | string | README.case (--compset) | Yes | Short compset |
| experiment | compset.long | string | e3sm_timing (compset) | Yes | Full compset |
| experiment | case_group | string | env_case.xml | No | Case grouping |
| experiment | compiler | string | env_build.xml | No | Compiler used |
| experiment | mpilib | string | env_build.xml | No | MPI library |
| experiment | version | string | GIT_DESCRIBE | Yes | E3SM version |
| pe_layout | Array | array | e3sm_timing (component table) | Yes | PE configuration |
| component_runtime | Array | array | e3sm_timing (runtime table) | Yes | Runtime by component |
| job_config | Object | object | preview_run.log | No | Job submission config |
| build_info | Object | object | build_times.txt | No | Build time breakdown |
| io_stats.scorpio | Array | array | spio_stats.tar.gz | No | I/O statistics |
| timing_profiles | Array | array | timing.tar.gz | No | Detailed GPTL timings |
| configuration.namelists | Object | object | CaseDocs/*.nml | No | Namelist files |
| configuration.xml_configs | Object | object | CaseDocs/*.xml | No | XML configuration |
| configuration.rc_files | Object | object | CaseDocs/*.rc | No | RC files |
| scripts | Object | object | *.sh files | No | Shell scripts |
| diagnostics.memory_profile_csv | string | memory.log | No | Raw CSV data |

### 6.4 Alternative: Separate Files Approach

Instead of a monolithic JSON, SimBoard could store:

1. **Core metadata:** `experiment.json` (required fields only)
2. **Configuration bundle:** `casedocs.tar.gz` (all CaseDocs files)
3. **Timing data:** `timing.tar.gz` (GPTL output files)
4. **I/O stats:** `spio_stats.tar.gz` (Scorpio JSON files)
5. **Memory profile:** `memory.csv.gz` (raw CSV)
6. **Scripts:** `scripts.tar.gz` (shell scripts)

**Advantages:**
- Smaller JSON payload for queries
- Preserve original file formats
- Lazy loading of large datasets
- Easier to extend

**Disadvantages:**
- More complex retrieval logic
- Multiple storage locations

---

## 7. Implementation Recommendations

### 7.1 Parser Modularity

Reimplement as independent functions:

```python
def parse_e3sm_timing(file_path: str) -> dict:
    """Parse e3sm_timing file, return core experiment metadata"""
    
def parse_readme(file_path: str) -> dict:
    """Parse README.case file, return case creation metadata"""
    
def parse_git_describe(file_path: str) -> str:
    """Parse GIT_DESCRIBE file, return version string"""
    
def parse_pe_layout(e3sm_timing_data: dict) -> list[dict]:
    """Extract PE layout from parsed e3sm_timing data"""
    
def parse_model_timing(tar_path: str) -> list[dict]:
    """Parse timing tar archive, return list of rank timings"""
    
def parse_scorpio_stats(tar_gz_path: str, run_time: float) -> list[dict]:
    """Parse Scorpio stats, return list of component I/O stats"""
    
def parse_casedocs(casedocs_dir: str) -> dict:
    """Parse all CaseDocs files, return structured config"""
    
def parse_build_times(file_path: str) -> dict:
    """Parse build times, return component breakdown"""
    
def parse_preview_run(file_path: str) -> dict:
    """Parse preview_run.log, return job config"""
```

### 7.2 Error Handling Strategy

```python
class ParseResult:
    success: bool
    data: dict | None
    errors: list[str]
    warnings: list[str]

def parse_experiment(exp_dir: str) -> ParseResult:
    """
    Parse entire experiment directory
    Returns ParseResult with:
    - success: True if required files parsed successfully
    - data: Merged metadata dictionary
    - errors: List of critical errors
    - warnings: List of missing optional files
    """
```

### 7.3 Testing Considerations

**Test data sources:**
- Use files from `portal/pace/e3sm/unit_tests/` as baseline
- Create minimal test cases for each parser
- Test gzipped and non-gzipped variants
- Test with missing optional files

**Critical test cases:**
1. Valid complete experiment
2. Minimal experiment (only required files)
3. Malformed e3sm_timing file
4. Multiple GPTL format versions
5. Old vs new Scorpio stats format
6. Missing CaseDocs files
7. Duplicate experiment detection

### 7.4 Dependency Management

**Required libraries (from current implementation):**
- Standard library: `gzip`, `tarfile`, `zipfile`, `json`, `os`, `sys`
- Third-party: `xmltodict`, `f90nml`

**Recommended changes for SimBoard:**
- Replace SQLAlchemy with plain Python classes
- Remove Flask dependencies
- Replace MinIO with generic object storage interface
- Add type hints (Python 3.9+)

### 7.5 Security Considerations

**Already implemented in PACE:**
- Path traversal protection (`badpath`, `badlink` in parseE3SM.py)
- Safe tar extraction (`safemembers` filter)
- Filename validation before file operations

**Additional recommendations:**
- Validate file sizes before extraction
- Set extraction timeout limits
- Sanitize user-provided case names
- Validate machine/compiler names against allowlist

---

## 8. Code Reference Summary

### 8.1 Main Parser Files

| File | Purpose | Key Functions |
|------|---------|---------------|
| `portal/pace/parse.py` | Top-level dispatcher | `parseData(zipfilename, uploaduser, project)` |
| `portal/pace/e3sm/e3smParser/parseE3SM.py` | E3SM orchestrator | `parseData(zipfilename, uploaduser)` |
| | | `insertExperiment(...)` |
| | | `insertE3SMTiming(...)` |
| `portal/pace/e3sm/e3smParser/parseE3SMTiming.py` | Main timing parser | `parseE3SMtiming(filename)` |
| `portal/pace/e3sm/e3smParser/parseReadMe.py` | README parser | `parseReadme(readmefilename)` |
| `portal/pace/e3sm/e3smParser/parseModelVersion.py` | Version parser | `parseModelVersion(gitfile)` |
| `portal/pace/e3sm/e3smParser/parseModelTiming.py` | GPTL timing parser | `parse(fileIn)` |
| | | `getData(src)` |
| | | `detectMtFile(fileObj)` |
| `portal/pace/e3sm/e3smParser/parseScorpioStats.py` | Scorpio I/O parser | `loaddb_scorpio_stats(spiofile, runTime)` |
| `portal/pace/e3sm/e3smParser/parseMemoryProfile.py` | Memory parser | `loaddb_memfile(memfile)` |
| `portal/pace/e3sm/e3smParser/parseBuildTime.py` | Build time parser | `loaddb_buildTimesFile(buildfile)` |
| `portal/pace/e3sm/e3smParser/parsePreviewRun.py` | Preview run parser | `load_previewRunFile(previewfile)` |
| `portal/pace/e3sm/e3smParser/parseCaseDocs.py` | CaseDocs dispatcher | `loaddb_casedocs(casedocpath, db, currExpObj)` |
| `portal/pace/e3sm/e3smParser/parseXML.py` | XML parser | `loaddb_xmlfile(xmlpath)` |
| `portal/pace/e3sm/e3smParser/parseNameList.py` | Namelist parser | `loaddb_namelist(nmlpath)` |
| `portal/pace/e3sm/e3smParser/parseRC.py` | RC file parser | `loaddb_rcfile(rcpath)` |
| `portal/pace/e3sm/e3smParser/parseText.py` | Text file parser | `load_text(file)` |
| `portal/pace/e3sm/e3smParser/parseReplaysh.py` | Replay script parser | `load_replayshFile(replayshfile)` |
| `portal/pace/e3sm/e3smParser/parseRunE3SMsh.py` | Run script parser | `load_rune3smshfile(rune3smshfile)` |

### 8.2 Database Schema

| File | Purpose |
|------|---------|
| `portal/pace/e3sm/e3smDb/datastructs.py` | ORM models for all tables |

**Tables defined:**
- Exp (base experiment table)
- E3SMexp (E3SM-specific metadata)
- Pelayout (PE layout per component)
- Runtime (runtime metrics per component)
- ModelTiming (GPTL timing data)
- NamelistInputs, XMLInputs, RCInputs (configuration files)
- ScorpioStats (I/O statistics)
- BuildTime (build time data)
- MemfileInputs (memory profile)
- PreviewRun (job configuration)
- ScriptsFile (shell scripts)

### 8.3 Test Files

| File | Purpose |
|------|---------|
| `portal/pace/e3sm/unit_tests/e3smParserTest.py` | Parser unit tests |
| `portal/pace/e3sm/unit_tests/test_modelTiming_parser.py` | Model timing tests |

**Test data files in `unit_tests/`:**
- `e3sm_timing.e3sm_v1.2_ne30_noAgg-60.43235257.210608-222102`
- `README.case.43235257.210608-222102.gz`
- `GIT_DESCRIBE.43235257.210608-222102.gz`
- `env_batch.xml.63117.210714-233452.gz`
- `atm_modelio.nml.303313.220628-152730.gz`
- `build_times.txt.303313.220628-152730.gz`
- `memory.3.86400.log.61012576.220711-001958.gz`
- `preview_run.log.303313.220628-152730.gz`
- `replay.sh.303313.220628-152730.gz`
- `run_e3sm.sh.20220506-193932.556061.220506-194028.gz`
- `seq_maps.rc.63117.210714-233452.gz`
- `spio_stats.303313.220628-152730.tar.gz`

---

## 9. Key Takeaways for SimBoard

### 9.1 Critical Success Factors

1. **File discovery is flexible:** Walk directory, find files by prefix
2. **Most files are optional:** Only e3sm_timing, README, GIT_DESCRIBE are required
3. **Formats are well-defined:** Each file type has clear structure
4. **Parsing is independent:** Each parser can run standalone
5. **Error handling is lenient:** Missing/malformed files don't abort entire upload

### 9.2 Simplification Opportunities

1. **Remove database coupling:** Return plain Python dicts/dataclasses
2. **Remove Flask dependencies:** Pure Python functions
3. **Simplify file storage:** Use object storage or filesystem
4. **Add type hints:** Modern Python 3.9+ typing
5. **Async parsing:** Parse multiple files concurrently
6. **Streaming extraction:** Don't extract entire archive to disk

### 9.3 Extension Points

SimBoard can add:

1. **Validation layer:** Check metadata consistency
2. **Metadata enrichment:** Derive additional fields
3. **Comparison API:** Compare experiments side-by-side
4. **Query optimization:** Index key fields for fast lookup
5. **Visualization hooks:** Pre-compute charts/tables
6. **Machine learning:** Predict performance from config

### 9.4 Compatibility Notes

To maintain compatibility with PACE exports:

1. **Preserve file naming conventions**
2. **Support same compression formats (gzip, tar)**
3. **Handle same XML/namelist formats**
4. **Support same GPTL versions**
5. **Accept same directory structure**

However, SimBoard can:

1. **Use different storage backend** (not MinIO required)
2. **Use different database** (not MySQL required)
3. **Add additional metadata fields**
4. **Support additional E3SM versions**

---

## 10. Next Steps

For SimBoard implementation:

1. **Phase 1: Core parsers**
   - Implement parseE3SMTiming, parseReadMe, parseModelVersion
   - Create experiment metadata extraction
   - Return structured dictionary

2. **Phase 2: Optional parsers**
   - Implement model timing, Scorpio stats parsers
   - Implement CaseDocs parsers
   - Handle missing files gracefully

3. **Phase 3: Integration**
   - Create orchestration layer
   - Implement file discovery
   - Add error handling and logging

4. **Phase 4: Storage**
   - Design storage schema (JSON, files, or hybrid)
   - Implement persistence layer
   - Add query interface

5. **Phase 5: Validation**
   - Test with PACE test data
   - Test with production experiments
   - Verify metadata accuracy

---

**Document Version:** 1.0  
**Last Updated:** 2026-01-14  
**Author:** GitHub Copilot Analysis  
**Status:** Complete
