# PACE REST API → Parser Call Flow Analysis

**Date:** 2026-01-14  
**Purpose:** Reference document for SimBoard to understand PACE's REST-to-parser flow for cron-driven ingestion  
**Status:** ANALYSIS ONLY - No code modifications

---

## Executive Summary

This document analyzes the call flow between PACE's REST APIs and the E3SM parsing logic. PACE uses a two-stage ingestion model:
1. **Upload Stage** (`/upload`): Accepts zip files, validates, and stores them to a filesystem staging area
2. **Parse Stage** (`/fileparse`): Triggers parser orchestration to extract metadata and persist to database

The flow is designed for **automated, cron-driven ingestion** from HPC systems where jobs call the REST APIs directly (non-interactive).

---

## Table of Contents

1. [REST Call Flow](#1-rest-call-flow)
2. [Data Handoff Points](#2-data-handoff-points)
3. [Control Decisions](#3-control-decisions)
4. [Side Effects](#4-side-effects)
5. [Transferable Patterns](#5-transferable-patterns)
6. [SimBoard Relevance](#6-simboard-relevance)

---

## 1. REST Call Flow

### 1.1 Upload Flow (`/upload`)

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

### 1.2 Parse Flow (`/fileparse`)

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

### 1.3 Parser Orchestration Flow

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

### 1.4 E3SM Parser Flow

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

## 2. Data Handoff Points

### 2.1 Client → REST Layer

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

### 2.2 REST Layer → Filesystem

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

### 2.3 Filesystem → Parser Layer

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

### 2.4 Parser Layer → Database

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

### 2.5 Database → Response

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

## 3. Control Decisions

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

### 3.3 Sync vs Async Execution

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

## 4. Side Effects

### 4.1 Filesystem Writes

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

### 4.2 Logging

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

### 4.3 Persistence (Database)

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

### 4.4 External Service Interactions

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

## 5. Transferable Patterns

### 5.1 Generally Reusable Patterns

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

### 5.2 PACE-Specific Patterns (Less Transferable)

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

## 6. SimBoard Relevance

### 6.1 How This Flow Could Be Mimicked

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

### 6.2 Patterns Worth Adopting

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

### 6.3 Patterns to Avoid

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

### 6.4 High-Level SimBoard Adaptation

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

### 6.5 Specific Recommendations

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
