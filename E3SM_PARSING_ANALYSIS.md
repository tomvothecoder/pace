# PACE E3SM Metadata Parsing (SimBoard-Focused)

**Purpose:** Document how PACE parses E3SM experiment archives and how to reimplement the **SimBoard-relevant subset**.  
**Scope:** Configuration, provenance, machine context, version metadata, and run timeline.  
**Out of scope:** Performance analytics, timing profiles, memory, and I/O statistics.

---

## High-Level Flow

```mermaid
flowchart TD
    Start([Experiment Archive]) --> Extract[Extract ZIP/TAR.GZ]
    Extract --> Discover[Discover exp* directories]

    Discover --> FindFiles[Locate Required Files]
    FindFiles --> Check{Required Files Present?}

    Check -->|No| Error[Fail: Missing Required Files]
    Check -->|Yes| ParseCore[Parse Core Metadata]

    ParseCore --> ParseOptional[Parse Optional Metadata]
    ParseOptional --> Normalize[Normalize + Map Fields]
    Normalize --> Dedup[Check for Duplicate Simulation]

    Dedup -->|Exists| ReturnExisting[Return Existing ID]
    Dedup -->|New| Create[Create Simulation Metadata]

    Create --> Output[SimBoard-ready JSON]
    ReturnExisting --> End([Complete])
    Output --> End
````

---

## Required vs Optional Files

### Required (SimBoard ingestion fails without these)

| File             | Purpose                                                  |
| ---------------- | -------------------------------------------------------- |
| `e3sm_timing.*`  | Case name, machine, user, grid, compset, experiment date |
| `README.case.*`  | Short resolution and compset                             |
| `GIT_DESCRIBE.*` | E3SM git version (tag + commit)                          |
| `CaseStatus.*`   | Run start date and run status                            |

> **Note:** SimBoard requires `run_start_date`. Therefore `CaseStatus.*` should be treated as required, even though PACE historically allowed it to be missing.

### Optional (Enhances metadata)

| File                           | Purpose               |
| ------------------------------ | --------------------- |
| `CaseDocs/env_case.xml.*`      | Case group            |
| `CaseDocs/env_build.xml.*`     | Compiler, MPI library |
| `GIT_CONFIG.*`                 | Git repository URL    |
| `GIT_STATUS.*`                 | Git branch            |
| `replay.sh.*`, `run_e3sm.sh.*` | Stored as artifacts   |

### Explicitly Ignored for SimBoard

Performance-only files:

* `timing.*`, `spio_stats.*`, `memory.*`
* `build_times.txt.*`
* `preview_run.log.*`

---

## Expected Directory Layout

```
exp-<name>/
├── e3sm_timing.<case>.<LID>        [required]
├── README.case.<LID>.gz            [required]
├── GIT_DESCRIBE.<LID>.gz           [required]
├── CaseStatus.<LID>.gz             [required]
├── replay.sh.<LID>.gz              [optional]
├── run_e3sm.sh.<LID>.gz            [optional]
└── CaseDocs.<LID>/
    ├── env_case.xml.<LID>.gz       [optional]
    ├── env_build.xml.<LID>.gz      [optional]
    └── other XML / namelist files  [ignored or extra]
```

---

## Core Metadata Extraction

### 1. `e3sm_timing.*`

Extracted fields:

| Field               | SimBoard Mapping          |
| ------------------- | ------------------------- |
| case                | `name`, `case_name`       |
| machine             | `machine_id` (via lookup) |
| user                | `extra.user`              |
| LID                 | `extra.lid`               |
| Curr Date           | `simulation_start_date`   |
| grid                | `grid_resolution`         |
| compset             | `compset_alias`           |
| run_type            | `initialization_type`     |
| stop_option, stop_n | `extra.run_config`        |

Ignored:

* All timing, PE layout, runtime, and performance metrics

---

### 2. `README.case.*`

Extracted fields:

| Source             | SimBoard Mapping      |
| ------------------ | --------------------- |
| `--res`            | `grid_name`           |
| `--compset`        | `compset`             |
| creation timestamp | `extra.creation_date` |

---

### 3. `GIT_DESCRIBE.*`

* Read first non-empty line
* Example: `v2.0.0-beta.3-3091-g3219b44fc`

Parsed into:

* `git_tag = v2.0.0-beta.3`
* `git_commit_hash = 3219b44fc`

---

### 4. `CaseStatus.*` (Required)

**Purpose:** Establish the actual run timeline.

Extracted fields:

| Field           | SimBoard Mapping |
| --------------- | ---------------- |
| `RUN_STARTDATE` | `run_start_date` |

#### Optional: `run_end_date`

`run_end_date` can be **derived** if all of the following are available:

* `run_start_date` (from `CaseStatus.*`)
* `stop_option` (from `e3sm_timing.*`)
* `stop_n` (from `e3sm_timing.*`)

Derivation logic:

* Interpret `stop_option` (e.g., `ndays`, `nmonths`, `nyears`)
* Add `stop_n` units to `run_start_date`
* If derivation fails, leave `run_end_date = NULL`

---

## Optional Metadata

### Case Group (`env_case.xml`)

* Extract `CASE_GROUP`
* Maps to `group_name`

### Build Info (`env_build.xml`)

* Extract:

  * `COMPILER`
  * `MPILIB`

### Git Info

* `GIT_CONFIG.*` → repository URL
* `GIT_STATUS.*` → branch name

### Scripts

* `replay.sh`, `run_e3sm.sh`
* Stored verbatim as artifacts (`RUN_SCRIPT`)

---

## Minimal Parsing Sequence (SimBoard)

1. Parse `e3sm_timing.*` → **required**
2. Parse `README.case.*` → **required**
3. Parse `GIT_DESCRIBE.*` → **required**
4. Parse `CaseStatus.*` → **required**
5. Parse `env_case.xml` → optional
6. Parse `env_build.xml` → optional
7. Parse git config/status → optional
8. Store scripts as artifacts → optional

---

## Error Handling Strategy

| Condition                     | Behavior              |
| ----------------------------- | --------------------- |
| Missing required file         | Fail ingestion        |
| Required parse error          | Fail ingestion        |
| Missing optional file         | Log warning, continue |
| Optional parse error          | Log warning, continue |
| Unable to derive run_end_date | Leave NULL            |

---

## Duplicate Detection

PACE uses `(user, machine, date, case)`.

**Recommended for SimBoard:**

* `(case_name, machine_id, simulation_start_date)`
* If duplicate exists → return existing simulation ID

---

## Direct Field Mappings (PACE → SimBoard)

Fields that PACE can directly populate in SimBoard’s `SimulationCreate` schema:

Nice, this is a good place to tighten the contract between PACE → backend → UI 👍
Below is your **updated table** with two new columns:

* **Required?** — per `SimulationCreate` (required vs optional at create time)
* **Assigned by** — where the value is expected to come from:

  * **PACE** (parsed / inferred during ingestion)
  * **SimBoard UI** (user-entered)
  * **Backend (auto)** (derived, looked up, or set server-side)

I stayed faithful to the FastAPI model and your notes.

---

### Configuration

| SimBoard Field         | PACE Source | PACE Field/File   | Required?    | Assigned by         | Notes                 |
| ---------------------- | ----------- | ----------------- | ------------ | ------------------- | --------------------- |
| `name`                 | e3sm_timing | `case`            | **Required** | Backend (from PACE) | Case name             |
| `case_name`            | e3sm_timing | `case`            | **Required** | Backend (from PACE) | Same as `name`        |
| `description`          | N/A         | -                 | Optional     | SimBoard UI         | Not extracted by PACE |
| `compset`              | README.case | `compset` (short) | **Required** | Backend (from PACE) | e.g., `F2010`         |
| `compset_alias`        | e3sm_timing | `long_compset`    | **Required** | Backend (from PACE) | Full component set    |
| `grid_name`            | README.case | `res` (short)     | **Required** | Backend (from PACE) | e.g., `ne30_ne30`     |
| `grid_resolution`      | e3sm_timing | `long_res`        | **Required** | Backend (from PACE) | Full grid spec        |
| `parent_simulation_id` | N/A         | -                 | Optional     | SimBoard UI         | Not tracked by PACE   |

---

### Model setup / context

| SimBoard Field        | PACE Source  | PACE Field/File | Required?    | Assigned by              | Notes                                              |
| --------------------- | ------------ | --------------- | ------------ | ------------------------ | -------------------------------------------------- |
| `simulation_type`     | N/A          | -               | **Required** | Backend (inferred) or UI | Could be inferred (e.g., production, test, branch) |
| `status`              | N/A          | -               | **Required** | Backend (auto)           | Initial status set on creation                     |
| `campaign_id`         | N/A          | -               | Optional     | SimBoard UI              | Not tracked by PACE                                |
| `experiment_type_id`  | N/A          | -               | Optional     | SimBoard UI              | Not tracked by PACE                                |
| `initialization_type` | e3sm_timing  | `run_type`      | **Required** | Backend (from PACE)      | From “run type”                                    |
| `group_name`          | env_case.xml | `case_group`    | Optional     | Backend (from PACE)      | CASE_GROUP                                         |

---

### Model timeline

| SimBoard Field          | PACE Source   | PACE Field/File | Required?    | Assigned by         | Notes                        |
| ----------------------- | ------------- | --------------- | ------------ | ------------------- | ---------------------------- |
| `machine_id`            | e3sm_timing   | machine name    | **Required** | Backend (lookup)    | Requires mapping name → UUID |
| `simulation_start_date` | e3sm_timing   | `exp_date`      | **Required** | Backend (from PACE) | From “Curr Date”             |
| `simulation_end_date`   | N/A           | -               | Optional     | Backend (derived)   | Often unknown at ingest      |
| `run_start_date`        | CaseStatus    | `RUN_STARTDATE` | Optional     | Backend (from PACE) | Parsed                       |
| `run_end_date`          | Derived       | -               | Optional     | Backend (derived)   | Optional                     |
| `compiler`              | env_build.xml | `COMPILER`      | Optional     | Backend (from PACE) |                              |

---

### Version control

| SimBoard Field       | PACE Source  | PACE Field/File | Required? | Assigned by         | Notes        |
| -------------------- | ------------ | --------------- | --------- | ------------------- | ------------ |
| `git_repository_url` | GIT_CONFIG   | -               | Optional  | Backend (from PACE) |              |
| `git_branch`         | GIT_CONFIG   | -               | Optional  | Backend (from PACE) |              |
| `git_tag`            | GIT_DESCRIBE | `version`       | Optional  | Backend (from PACE) | Git describe |
| `git_commit_hash`    | GIT_DESCRIBE | `version`       | Optional  | Backend (from PACE) | Parsed       |

---

### Provenance & audit

| SimBoard Field    | PACE Source | PACE Field/File | Required? | Assigned by    | Notes                       |
| ----------------- | ----------- | --------------- | --------- | -------------- | --------------------------- |
| `created_by`      | N/A         | -               | Optional* | Backend (auth) | Set from authenticated user |
| `last_updated_by` | N/A         | -               | Optional* | Backend (auth) | Set on update               |

* Optional in schema, but effectively required in authenticated workflows.

---

### Miscellaneous / user annotation

| SimBoard Field   | PACE Source | PACE Field/File | Required? | Assigned by  | Notes             |
| ---------------- | ----------- | --------------- | --------- | ------------ | ----------------- |
| `key_features`   | N/A         | -               | Optional  | SimBoard UI  | User input        |
| `known_issues`   | N/A         | -               | Optional  | SimBoard UI  | User input        |
| `notes_markdown` | N/A         | -               | Optional  | SimBoard UI  | User input        |
| `extra`          | Multiple    | Various         | Optional  | Backend / UI | Non-core metadata |

---
### Absolute minimum required fields

```python
name
case_name
compset
compset_alias
grid_name
grid_resolution
simulation_type
status
initialization_type
machine_id
simulation_start_date
```

---

## Key Takeaways

* **Four files are required** for SimBoard ingestion:

  * `e3sm_timing.*`
  * `README.case.*`
  * `GIT_DESCRIBE.*`
  * `CaseStatus.*`
* `run_end_date` is **derived**, not parsed
* Most PACE complexity is performance-related and ignorable
* SimBoard can implement a **small, deterministic parser**
* Heavy data belongs in `extra` or artifacts, not core schema

---

**Document Version:** 1.3
**Last Updated:** 2026-01-14
**Status:** Simplified, SimBoard-ready
