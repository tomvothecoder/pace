# PACE REST API → Parser Call Flow Analysis

**Date:** 2026-01-14
**Purpose:** Reference document for SimBoard to understand PACE's REST-to-parser flow for cron-driven ingestion
**Status:** Analysis only – no code modifications

---

## Executive Summary

PACE ingests E3SM experiments using a **two-stage REST model**:

1. **Upload (`/upload`)** – Store zip archive in a staging directory
2. **Parse (`/fileparse`)** – Extract, parse, insert into database, archive, and clean up

The system is designed for **automated, cron-driven ingestion** from HPC systems.

Key characteristics:

* Batch processing of multiple experiments per upload
* Continue-on-error behavior
* Duplicate detection for idempotency
* Synchronous parsing (blocks HTTP request)
* Heavy reliance on filesystem staging

This document summarizes:

* Parser entry points
* Required filesystem layout
* Metadata derivation sources
* REST call flow
* Control decisions
* Side effects
* Transferable patterns for SimBoard

---

## Table of Contents

1. [E3SM Parser Entrypoints & Contracts](#1-e3sm-parser-entrypoints--contracts)
2. [Required Filesystem Layout](#2-required-filesystem-layout)
3. [Field-Level Metadata Derivation](#3-field-level-metadata-derivation)
4. [REST Call Flow](#4-rest-call-flow)
5. [Data Handoff Points](#5-data-handoff-points)
6. [Control Decisions](#6-control-decisions)
7. [Side Effects](#7-side-effects)
8. [Transferable Patterns](#8-transferable-patterns)
9. [SimBoard Relevance](#9-simboard-relevance)

---

# 1. E3SM Parser Entrypoints & Contracts

## 1.1 Primary Entrypoint

```
parseE3SM.parseData(zipfilename: str, uploaduser: str) -> str
```

Returns:

* `"success"`
* `"fail"`
* `"ERROR"`

### Responsibilities

* Extract zip and tar files
* Discover `exp*` experiment directories
* Parse each experiment
* Insert into database
* Archive experiment
* Clean staging directory

---

## 1.2 Core Metadata Parser

```
parseE3SMTiming.parseE3SMtiming(filename)
```

Extracts ~20 core metadata fields from `e3sm_timing.*`, including:

* case
* lid
* machine
* user
* resolution (long form)
* compset (long form)
* run length
* PE layout
* cost and throughput
* timing summary

This is the **primary metadata source**.

---

## 1.3 Resolution & Compset Parser

```
parseReadMe.parseReadme(readme_file)
```

Extracts:

* short-form resolution
* short-form compset

Important distinction:

| Source      | Form                              |
| ----------- | --------------------------------- |
| README.case | Short form (e.g., F2010)          |
| e3sm_timing | Long form (fully expanded config) |

PACE stores both.

---

## 1.4 CaseDocs Parser

```
parseCaseDocs.loaddb_casedocs(directory)
```

Parses:

* XML files
* Namelists
* RC files

Additional metadata derived:

* `case_group` (env_case.xml)
* `compiler`, `mpilib` (env_build.xml)

---

## 1.5 Timing Tree Parser

```
parseModelTiming.parse(file)
```

Parses per-rank GPTL timing output into JSON trees.

Inserted into `model_timing` table (one per MPI rank + stats).

---

# 2. Required Filesystem Layout

Expected archive structure:

```
pace-exps-USER-TIMESTAMP.zip
└── pace-exps-USER-TIMESTAMP/
    └── exp0/
        e3sm_timing.*
        timing.*.tar.gz
        README.case.*
        GIT_DESCRIBE.*
        CaseDocs.*/
```

## 2.1 Required Files

| File            | Purpose             |
| --------------- | ------------------- |
| e3sm_timing.*   | Core metadata       |
| timing.*.tar.gz | Timing trees        |
| README.case.*   | Short-form metadata |
| GIT_DESCRIBE.*  | Version             |
| CaseDocs/       | Config + XML        |

Missing required files → experiment fails.

---

## 2.2 Optional Files

* memory.*
* spio_stats.*
* build_times.*
* preview_run.*
* replay.sh / run_e3sm.sh

Missing optional files do not fail ingestion.

---

# 3. Field-Level Metadata Derivation

## 3.1 Primary Metadata (e3smexp)

| Field Type               | Source        |
| ------------------------ | ------------- |
| Case, machine, user      | e3sm_timing   |
| Long resolution/compset  | e3sm_timing   |
| Short resolution/compset | README.case   |
| Version                  | GIT_DESCRIBE  |
| Compiler / MPILIB        | env_build.xml |
| Case group               | env_case.xml  |

Derivation is deterministic.
Each field has a single source file.

---

## 3.2 PE Layout & Runtime Tables

Extracted from tables inside `e3sm_timing.*`:

* pelayout table
* runtime table

One row per component.

---

## 3.3 Timing Trees

From `timing.*.tar.gz`:

* One JSON tree per MPI rank
* Plus aggregate stats

---

# 4. REST Call Flow

## 4.1 Upload Stage (`/upload`)

* Accept zip
* Validate extension
* Save to `UPLOAD_FOLDER`
* Return `"complete"`

No parsing occurs here.

---

## 4.2 Parse Stage (`/fileparse`)

* Validate user (regex)
* Validate filename (pattern)
* Call `parse.parseData()`
* Return `"success/logfile"` or `"fail/logfile"`

This call blocks until parsing finishes.

---

## 4.3 Orchestration Flow

```
/fileparse
  → parse.parseData
    → parseE3SM.parseData
        → discover experiments
        → for each experiment:
            parse metadata
            parse timing
            parse config
            insert DB rows
            commit
        → cleanup staging
```

Batch behavior:

* Multiple experiments allowed per upload
* Continue-on-error

---

# 5. Data Handoff Points

## Client → REST

* Zip file
* filename
* user
* project

## REST → Filesystem

* Save zip
* Extract archive
* Create logs
* Archive final experiment

## Filesystem → Parsers

File paths passed to parser functions.

## Parsers → Database

ORM objects inserted across 13+ tables.

Transaction per experiment.

---

# 6. Control Decisions

## 6.1 Validation Gates

1. Zip file extension
2. User regex
3. Filename regex
4. Duplicate experiment check
5. Required file presence
6. Successful DB commit

---

## 6.2 Duplicate Detection

Duplicate if:

```
(user, machine, case, exp_date)
```

If duplicate:

* Skip
* Continue batch
* Not treated as fatal

Ensures idempotent cron ingestion.

---

## 6.3 Execution Model

* Fully synchronous
* No background jobs
* No progress updates
* HTTP request blocks until completion

Acceptable for cron, not ideal for interactive use.

---

# 7. Side Effects

## Filesystem

* Extract archive
* Write logs
* Archive experiment zip
* Delete staging directory

## Database

* Insert into multiple related tables
* Commit per experiment
* Rollback on failure

## Object Storage

* Upload archived experiment to Minio

---

# 8. Transferable Patterns

## Recommended to Reuse

* Two-stage upload/parse model
* Duplicate detection
* Batch continue-on-error
* Transaction per experiment
* Per-upload logging
* Idempotent ingestion

---

## Patterns to Avoid

* Synchronous parsing inside REST handler
* GitHub OAuth for machine authentication
* Filename-based parser discovery
* Global stdout redirection
* Tight coupling between orchestration and parsing

---

# 9. SimBoard Relevance

PACE is optimized for:

* Nightly HPC cron ingestion
* Large batch processing
* Idempotency
* Filesystem-based workflows

SimBoard should:

* Reuse parser contracts and duplicate logic
* Move parsing to background workers
* Provide job status endpoint
* Use API-key or service authentication
* Use structured logging instead of stdout redirection
* Support async processing for scalability

---

## Conclusion

PACE’s ingestion pipeline is well-suited for automated HPC workflows:

* Deterministic parsing
* Clear file contracts
* Batch-safe design
* Robust duplicate handling

However, its synchronous REST model and tight filesystem coupling limit scalability.

SimBoard can adopt the ingestion logic while modernizing:

* Async execution
* Background jobs
* Status visibility
* Structured logging
* Extensible parser architecture
