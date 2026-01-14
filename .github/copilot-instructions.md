# Copilot Instructions for PACE

## Project Overview

PACE (Performance Analytics for Computational Experiments) is a
web-enabled framework for summarizing and analyzing performance data
from E3SM experiments. It consists of a Flask-based web portal, a Python
client for uploading/parsing data, and a MySQL backend.

PACE is currently used primarily as an automated ingestion and analysis
service for E3SM experiments executed on DOE HPC systems.

------------------------------------------------------------------------

## Architecture & Key Components

-   **portal/pace/**: Main Flask web application and backend logic.
    -   `webapp.py`: Flask routes, upload/auth logic, main entrypoint.
    -   `pace_common.py`: Environment/config/database helpers. Reads
        `.pacerc` for secrets/paths.
    -   `parse.py`: Orchestrates parsing of uploaded experiment data.
    -   `e3sm/e3smParser/`: Specialized E3SM data parsers (timing,
        memory, version, etc).
    -   `e3sm/e3smDb/`: Data structures for DB interaction.
    -   `static/`, `templates/`: Frontend assets and Jinja2 templates.
-   **client/**: Python CLI tool (`pace-upload`) for uploading and
    parsing E3SM performance data.
-   **docker/**: Scripts and templates for containerized
    development/deployment.
-   **utils/**: Shell/Perl scripts for archiving, permissions, and
    automation.

------------------------------------------------------------------------

## Developer Workflows

-   **Run Web Portal**: Launch Flask app from `portal/pace/webapp.py`.
    Requires a valid `.pacerc` config (see `etc/pacerc.tmpl`).

-   **Upload Data**: Use `client/pace-upload` CLI. See
    `client/README.md` for usage and required arguments.

-   **Unit Tests**: Run from `portal/pace/e3sm/unit_tests/`:

    ``` sh
    cd portal/pace/e3sm/unit_tests
    cp <existing .pacerc> .
    PYTHONPATH=$PYTHONPATH:../../.. python3 ./e3smParserTest.py
    ```

-   **Configuration**: All secrets/paths are loaded from `.pacerc` (see
    `pace_common.py`). File permissions must be `600`.

-   **Database**: MySQL, connection string built from `.pacerc`.

-   **OAuth**: GitHub OAuth for user login (see `webapp.py`,
    `pace_common.py`).

------------------------------------------------------------------------

## Project-Specific Patterns & Conventions

-   **File Uploads**: Only `.zip` files are accepted. Filenames must
    match `pace-exps-<user>-<timestamp>.zip`.
-   **Environment Detection**: Paths and credentials are dynamically
    loaded based on environment and `.pacerc` location.
-   **Parsing**: All parsing logic is routed through `parse.py` and
    dispatched by project type (currently only `e3sm`).
-   **Frontend**: Jinja2 templates in `templates/`, static assets in
    `static/`.
-   **Logging**: Logs are written to the directory specified in
    `.pacerc` under `[ENVIRONMENT] pace_log_dir`.

------------------------------------------------------------------------

## Integration Points

-   **MinIO**: Used for object storage (see `pace_common.py`).
-   **GitHub**: OAuth for authentication.
-   **MySQL**: Main data store, SQLAlchemy ORM.

------------------------------------------------------------------------

## Analysis-Only Tasks (Very Important)

For requests involving architecture understanding, parsing logic
analysis, or documentation:

-   Do NOT modify code
-   Do NOT refactor or reformat files
-   Do NOT add tests or CI changes
-   Do NOT open pull requests
-   Read and analyze the existing implementation only
-   Produce structured markdown summaries and diagrams in issue comments
    or responses
-   Prefer numbered step flows, tables, and bullet lists
-   Reference code locations explicitly (file + function name)

------------------------------------------------------------------------

## E3SM Parsing Analysis Focus

When asked to analyze E3SM parsing:

-   Focus on code under `portal/pace/e3sm/e3smParser/`
-   Trace how metadata is extracted from:
    -   `env_case.xml`
    -   `env_run.xml`
    -   `CaseStatus` files
    -   Timing logs
    -   Machine and compiler configuration
-   Identify:
    -   Required and optional input files
    -   Parsing entrypoints
    -   Intermediate representations
    -   Final normalized fields
-   Identify the minimal invocation contract for the E3SM parsers:
    -   Required filesystem layout
    -   Required files
    -   Arguments and environment assumptions
-   Explicitly map parsed fields to a neutral, backend-independent
    schema suitable for reimplementation outside PACE (e.g., in
    SimBoard)

------------------------------------------------------------------------

## REST API → Parser Call Chain Analysis

When analyzing E3SM parsing behavior:

-   Trace how Flask REST endpoints invoke parsing logic
-   Treat REST APIs as reference plumbing only
-   Do NOT propose new REST APIs or redesign existing ones
-   Focus on:
    -   Request → handler → filesystem staging → parser invocation
-   Identify:
    -   Flask routes and HTTP methods
    -   Request payloads and validation
    -   How uploaded artifacts are staged on disk
    -   Where parsing is invoked (functions/classes)
    -   What data is passed into the E3SM parsers
-   Clearly separate:
    -   HTTP concerns
    -   Filesystem staging
    -   Parsing logic
    -   Persistence / side effects
-   Highlight which layers are unnecessary for a SimBoard-native
    implementation

------------------------------------------------------------------------

## Operational Ingestion Context (Authoritative)

The following describes how PACE is operated in production and should be
treated as ground truth when analyzing API call flow.

-   PACE is hosted at Oak Ridge
-   Cron jobs run on multiple DOE HPC systems
-   These cron jobs:
    -   Possess GitHub credentials
    -   Run nightly
    -   Automatically upload E3SM experiment data
    -   Call PACE REST APIs directly
-   Data ingestion and parsing is NOT user-driven in production

### Ingestion Flow (High Level)

1.  E3SM experiments complete on DOE supercomputers
2.  Cron jobs collect experiment artifacts from E3SM case directories
3.  Cron jobs call PACE REST APIs:
    -   `/upload`: stores a zip of files under directories determined by
        `getDirectories()` and `UPLOAD_FOLDER`
    -   `/fileparse`: receives a POST request and invokes parser scripts
        to extract metadata
4.  Parsed metadata is persisted in PACE

### Analysis Expectations

When analyzing REST APIs and parsers:

-   Treat cron-driven ingestion as the primary production use case
-   Identify how REST endpoints support automated, unattended execution
-   Highlight assumptions made to support cron-based workflows
-   Account for:
    -   Unattended execution
    -   Idempotency and retries
    -   Failure handling and partial ingestion
-   Explicitly separate:
    -   Operational concerns (cron, credentials, scheduling)
    -   API concerns (request/response)
    -   Parsing logic (filesystem → metadata)

------------------------------------------------------------------------

## References

-   [client/README.md](../client/README.md): Client usage and options
-   [portal/pace/pace_common.py](../portal/pace/pace_common.py):
    Config/database helpers
-   [portal/pace/webapp.py](../portal/pace/webapp.py): Flask app
    entrypoint
-   [portal/pace/e3sm/unit_tests/README.md](../portal/pace/e3sm/unit_tests/README.md):
    Unit test instructions
-   [etc/pacerc.tmpl](../etc/pacerc.tmpl): Example config file

------------------------------------------------------------------------

For more context, see https://pace-docs.readthedocs.io and in-code
comments.
