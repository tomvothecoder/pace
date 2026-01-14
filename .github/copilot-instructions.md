# Copilot Instructions for PACE

## Project Overview
PACE (Performance Analytics for Computational Experiments) is a web-enabled framework for summarizing and analyzing performance data from E3SM experiments. It consists of a Flask-based web portal, a Python client for uploading/parsing data, and a MySQL backend.

## Architecture & Key Components
- **portal/pace/**: Main Flask web application and backend logic.
  - `webapp.py`: Flask routes, upload/auth logic, main entrypoint.
  - `pace_common.py`: Environment/config/database helpers. Reads `.pacerc` for secrets/paths.
  - `parse.py`: Orchestrates parsing of uploaded experiment data.
  - `e3sm/e3smParser/`: Specialized E3SM data parsers (timing, memory, version, etc).
  - `e3sm/e3smDb/`: Data structures for DB interaction.
  - `static/`, `templates/`: Frontend assets and Jinja2 templates.
- **client/**: Python CLI tool (`pace-upload`) for uploading and parsing E3SM performance data.
- **docker/**: Scripts and templates for containerized development/deployment.
- **utils/**: Shell/Perl scripts for archiving, permissions, and automation.

## Developer Workflows
- **Run Web Portal**: Launch Flask app from `portal/pace/webapp.py`. Requires a valid `.pacerc` config (see `etc/pacerc.tmpl`).
- **Upload Data**: Use `client/pace-upload` CLI. See `client/README.md` for usage and required arguments.
- **Unit Tests**: Run from `portal/pace/e3sm/unit_tests/`:
  ```sh
  cd portal/pace/e3sm/unit_tests
  cp <existing .pacerc> .
  PYTHONPATH=$PYTHONPATH:../../.. python3 ./e3smParserTest.py
  ```
- **Configuration**: All secrets/paths are loaded from `.pacerc` (see `pace_common.py`). File permissions must be `600`.
- **Database**: MySQL, connection string built from `.pacerc`.
- **OAuth**: GitHub OAuth for user login (see `webapp.py`, `pace_common.py`).

## Project-Specific Patterns & Conventions
- **File Uploads**: Only `.zip` files are accepted. Filenames must match `pace-exps-<user>-<timestamp>.zip`.
- **Environment Detection**: Paths and credentials are dynamically loaded based on environment and `.pacerc` location.
- **Parsing**: All parsing logic is routed through `parse.py` and dispatched by project type (currently only `e3sm`).
- **Frontend**: Jinja2 templates in `templates/`, static assets in `static/`.
- **Logging**: Logs are written to the directory specified in `.pacerc` under `[ENVIRONMENT] pace_log_dir`.

## Integration Points
- **MinIO**: Used for object storage (see `pace_common.py`).
- **GitHub**: OAuth for authentication.
- **MySQL**: Main data store, SQLAlchemy ORM.

## Analysis-Only Tasks (Very Important)
For requests involving architecture understanding, parsing logic analysis, or documentation:
- Do NOT modify code
- Do NOT refactor or reformat files
- Do NOT add tests or CI changes
- Read and analyze the existing implementation only
- Produce structured markdown summaries and diagrams in PR comments or issue responses

## E3SM Parsing Analysis Focus
When asked to analyze E3SM parsing:
- Focus on code under `portal/pace/e3sm/e3smParser/`
- Trace how metadata is extracted from:
  - env_case.xml
  - env_run.xml
  - CaseStatus files
  - timing logs
  - machine/compiler configuration
- Identify:
  - input files
  - parsing entrypoints
  - intermediate representations
  - final normalized fields
- Explicitly map parsed fields to a neutral schema suitable for reimplementation outside PACE

## REST API → Parser Call Chain Analysis
When analyzing E3SM parsing behavior:
- Trace how Flask REST endpoints invoke parsing logic
- Identify:
  - Flask routes and HTTP methods
  - Request payloads and validation
  - How uploaded artifacts are staged on disk
  - Where parsing is invoked (functions/classes)
  - What data is passed into the E3SM parsers
- Clearly separate:
  - HTTP concerns
  - Filesystem staging
  - Parsing logic
  - Persistence / side effects
- Highlight which layers are unnecessary for a SimBoard-native implementation


## References
- [client/README.md](../client/README.md): Client usage and options
- [portal/pace/pace_common.py](../portal/pace/pace_common.py): Config/database helpers
- [portal/pace/webapp.py](../portal/pace/webapp.py): Flask app entrypoint
- [portal/pace/e3sm/unit_tests/README.md](../portal/pace/e3sm/unit_tests/README.md): Unit test instructions
- [etc/pacerc.tmpl](../etc/pacerc.tmpl): Example config file

---
For more, see [https://pace-docs.readthedocs.io](https://pace-docs.readthedocs.io) and in-code comments.
