# vscsiStats Helper - Technical Design Document

**Version:** 1.0
**Date:** 2025-11-06
**Status:** Draft

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [System Components](#system-components)
3. [Data Formats and Schemas](#data-formats-and-schemas)
4. [Module Structure](#module-structure)
5. [CLI Command Specifications](#cli-command-specifications)
6. [Core Workflows](#core-workflows)
7. [Error Handling and Reliability](#error-handling-and-reliability)
8. [Configuration Management](#configuration-management)
9. [Testing Strategy](#testing-strategy)
10. [Implementation Phases](#implementation-phases)

---

## Architecture Overview

### High-Level Architecture

vscsiStats Helper uses a **background service + CLI tool** architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                         User                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
          ┌────────────────────────┐
          │   CLI Tool (vscsi-helper)│
          │   - discover            │
          │   - start               │
          │   - status              │
          │   - stop                │
          │   - analyze             │
          │   - list                │
          │   - delete              │
          └───────────┬─────────────┘
                      │
                      │ IPC (socket/file-based)
                      │
          ┌───────────▼─────────────┐
          │  Background Service     │
          │  (vscsi-daemon)         │
          │  - Job management       │
          │  - SSH connection mgmt  │
          │  - Data collection      │
          │  - Retry logic          │
          └───────────┬─────────────┘
                      │
                      │ SSH
                      │
          ┌───────────▼─────────────┐
          │   ESXi Host             │
          │   - vscsiStats utility  │
          │   - VM workloads        │
          └─────────────────────────┘
```

### Design Rationale

**Why a background service?**
- Collection jobs run for minutes to hours (5 min - 3+ hours)
- SSH connections must persist across CLI invocations
- Users should be able to start jobs and disconnect
- Status checking and job management require shared state

**Service Model: User-Level Daemon**
- No system-wide installation or sudo required
- Each user runs their own daemon instance
- Daemon auto-starts on first CLI invocation
- Data stored in user-specified directories

### Technology Stack

| Component | Technology | Justification |
|-----------|------------|---------------|
| Language | Python 3.8+ | Rich ecosystem, rapid development |
| SSH Library | Paramiko | Pure Python, robust ESXi support |
| Data Processing | pandas, numpy | Histogram analysis, statistics |
| Visualization | matplotlib | 3D surface plots |
| CLI Framework | Click | Clean API, good documentation |
| IPC | Unix domain socket | Fast, secure, local-only |
| Daemon Management | python-daemon | Standard library approach |
| Config Format | YAML | Human-readable, comments |
| Data Serialization | JSON | Job metadata, state persistence |

---

## System Components

### 1. CLI Tool (`vscsi-helper`)

**Purpose**: User-facing command-line interface

**Responsibilities**:
- Parse user commands and arguments
- Communicate with background daemon via IPC
- Display job status and results
- Perform analysis on collected data (offline, no daemon needed)
- Generate visualizations and fio job files

**Key Features**:
- Minimal logic - mostly delegates to daemon
- `analyze` command runs independently (reads job data directly)
- Can work offline for analysis tasks

### 2. Background Daemon (`vscsi-daemon`)

**Purpose**: Long-running service that manages collection jobs

**Responsibilities**:
- Maintain persistent SSH connections to ESXi hosts
- Execute vscsiStats commands periodically
- Store collected data to disk
- Track job state (running, completed, failed)
- Handle connection failures and retries
- Clean up vscsiStats processes on ESXi hosts

**Key Features**:
- Single-threaded with async I/O (asyncio + asyncssh)
- One daemon instance per user
- Auto-starts on demand
- Graceful shutdown with cleanup

**State Management**:
- In-memory: Active job state, SSH connections
- On-disk: Job metadata, collected samples, persistent state

### 3. Core Modules

See [Module Structure](#module-structure) section for detailed breakdown.

---

## Data Formats and Schemas

### Job Directory Structure

Each job is stored in a dedicated directory:

```
~/.vscsiStats-helper/data/
├── sql-server-peak/               # User-provided job name
│   ├── job.json                   # Job metadata
│   ├── samples/                   # Raw collected samples
│   │   ├── sample_001.csv
│   │   ├── sample_002.csv
│   │   └── ...
│   ├── analysis/                  # Analysis outputs (created on analyze)
│   │   ├── summary.json
│   │   ├── filtered_data.csv
│   │   └── peak-workload.fio
│   └── visualizations/            # Generated charts
│       ├── iops-3d.png
│       ├── latency-3d.png
│       └── blocksize-3d.png
│
└── job-20251106-083022/           # Auto-generated job name
    └── ...
```

### Job Metadata Schema (`job.json`)

```json
{
  "job_id": "sql-server-peak",
  "created_at": "2025-11-06T08:30:22Z",
  "updated_at": "2025-11-06T09:00:45Z",
  "status": "completed",
  "host": {
    "hostname": "esxi-host01.example.com",
    "username": "root",
    "fingerprint": "SHA256:..."
  },
  "target": {
    "vm_name": "SQL-Server-01",
    "world_group_id": "12345",
    "lun": "vmhba1:C0:T0:L0",
    "handle_id": "67890"
  },
  "profile": {
    "name": "standard",
    "samples": 30,
    "interval_seconds": 60
  },
  "collection": {
    "started_at": "2025-11-06T08:30:30Z",
    "completed_at": "2025-11-06T09:00:45Z",
    "samples_collected": 30,
    "samples_expected": 30,
    "errors": []
  },
  "cleanup": {
    "vscsiStats_stopped": true,
    "stopped_at": "2025-11-06T09:00:46Z",
    "cleanup_error": null
  }
}
```

**Status Values**:
- `pending`: Job created but not started
- `running`: Collection in progress
- `completed`: All samples collected successfully
- `failed`: Job failed (see errors array)
- `stopped`: User manually stopped job

### vscsiStats CSV Format

vscsiStats outputs histogram data in CSV format. Example:

```csv
# This is a vscsiStats histogram output
# VM: SQL-Server-01, World Group: 12345, LUN: vmhba1:C0:T0:L0
IOLength(bytes),Seeks(sectors),OutstandingIOs,Latency(ms),Count,Type
512,0,1,0.5,145,Read
4096,0,1,0.8,892,Read
8192,0,1,1.2,3421,Read
8192,8,1,1.5,234,Read
4096,0,2,2.1,112,Write
...
```

**Key Fields**:
- `IOLength(bytes)`: Block size of the I/O operation
- `Seeks(sectors)`: Distance from previous I/O (0 = sequential)
- `OutstandingIOs`: Queue depth at time of I/O
- `Latency(ms)`: Latency bucket (histogram bucket center)
- `Count`: Number of I/Os in this histogram bucket
- `Type`: Read or Write

### Analysis Summary Schema (`analysis/summary.json`)

```json
{
  "job_id": "sql-server-peak",
  "analyzed_at": "2025-11-06T09:15:00Z",
  "filter": {
    "method": "percentile",
    "threshold": 75,
    "samples_filtered": 7,
    "samples_analyzed": 23
  },
  "metrics": {
    "iops": {
      "total": 4280,
      "read": 2996,
      "write": 1284,
      "read_write_ratio": "70/30"
    },
    "throughput_mbps": 145.3,
    "block_size_distribution": {
      "4k": 5,
      "8k": 75,
      "64k": 20
    },
    "sequential_random": {
      "sequential_percent": 35,
      "random_percent": 65
    },
    "queue_depth": {
      "average": 24,
      "p50": 22,
      "p95": 42,
      "p99": 48
    },
    "latency_ms": {
      "read": {
        "p50": 2.1,
        "p95": 8.4,
        "p99": 15.2
      },
      "write": {
        "p50": 3.2,
        "p95": 12.1,
        "p99": 22.8
      }
    }
  }
}
```

### fio Job File Format

Generated fio job files should be valid and ready to use:

```ini
[sql-server-peak]
# Generated from vscsiStats data
# Job: sql-server-peak
# Analyzed: 2025-11-06T09:15:00Z
# Filter: percentile (75th), 23/30 samples

# Workload characteristics
rw=randrw
rwmixread=70
blocksize=8k
iodepth=24
numjobs=1

# Block size distribution
# 8k: 75%, 4k: 5%, 64k: 20%
bssplit=4k/5:8k/75:64k/20

# Sequential vs random
# 35% sequential, 65% random
percentage_random=65

# Runtime
runtime=300
time_based=1

# Target
filename=/dev/sdb
direct=1
ioengine=libaio

# Output
write_bw_log=sql-server-peak
write_lat_log=sql-server-peak
write_iops_log=sql-server-peak
log_avg_msec=1000
```

---

## Module Structure

```
vscsiStats-helper/
├── src/
│   └── vscsi_helper/
│       ├── __init__.py
│       ├── __main__.py              # Entry point for CLI
│       │
│       ├── cli/
│       │   ├── __init__.py
│       │   ├── main.py              # Click CLI definitions
│       │   ├── discover.py          # discover command
│       │   ├── collection.py        # start, stop, status commands
│       │   ├── analysis.py          # analyze command
│       │   └── management.py        # list, delete commands
│       │
│       ├── daemon/
│       │   ├── __init__.py
│       │   ├── main.py              # Daemon entry point
│       │   ├── server.py            # IPC server (socket listener)
│       │   ├── job_manager.py       # Job lifecycle management
│       │   └── collector.py         # SSH + vscsiStats execution
│       │
│       ├── core/
│       │   ├── __init__.py
│       │   ├── ssh.py               # SSH connection wrapper
│       │   ├── vscsi.py             # vscsiStats command builder
│       │   ├── parser.py            # CSV parsing logic
│       │   └── job.py               # Job data model
│       │
│       ├── analysis/
│       │   ├── __init__.py
│       │   ├── filter.py            # Activity filtering logic
│       │   ├── metrics.py           # Workload metric calculations
│       │   ├── fio_generator.py     # fio job file generation
│       │   └── visualizer.py        # 3D chart generation
│       │
│       ├── config/
│       │   ├── __init__.py
│       │   ├── manager.py           # Config file loading
│       │   ├── defaults.py          # Built-in profiles
│       │   └── schema.py            # Config validation
│       │
│       └── utils/
│           ├── __init__.py
│           ├── paths.py             # Path resolution (data dir, config)
│           ├── ipc.py               # IPC protocol (client/server)
│           └── logging.py           # Logging configuration
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│       └── sample_vscsiStats.csv    # Example data
│
├── docs/
│   └── examples/
│
├── pyproject.toml
├── README.md
├── TECHNICAL_DESIGN.md
└── LICENSE
```

### Module Descriptions

#### `cli/` - Command Line Interface
- `main.py`: Click group and global options
- `discover.py`: SSH to ESXi, list VMs and LUNs
- `collection.py`: start/stop/status commands (talk to daemon)
- `analysis.py`: Offline analysis (no daemon dependency)
- `management.py`: list/delete job commands

#### `daemon/` - Background Service
- `main.py`: Daemon initialization, signal handling
- `server.py`: Unix socket server, handles IPC requests
- `job_manager.py`: Manages job queue and state
- `collector.py`: SSH connection, vscsiStats execution loop

#### `core/` - Core Functionality
- `ssh.py`: SSH connection management, retry logic
- `vscsi.py`: vscsiStats command building and execution
- `parser.py`: Parse vscsiStats CSV output
- `job.py`: Job data model (load/save job.json)

#### `analysis/` - Data Analysis
- `filter.py`: Percentile, statistical, absolute threshold filters
- `metrics.py`: Calculate IOPS, throughput, latency, seq/rand ratio
- `fio_generator.py`: Generate fio job files from metrics
- `visualizer.py`: matplotlib 3D surface charts

#### `config/` - Configuration Management
- `manager.py`: Load user config, merge with defaults
- `defaults.py`: Built-in profile definitions
- `schema.py`: Validate config structure

#### `utils/` - Utilities
- `paths.py`: Resolve data directory, config file paths
- `ipc.py`: IPC protocol (JSON-RPC over Unix socket)
- `logging.py`: Configure logging for CLI and daemon

---

## CLI Command Specifications

### Global Options

```bash
vscsi-helper [OPTIONS] COMMAND [ARGS]...

Options:
  --data-dir PATH         Data directory for jobs [default: ~/.vscsiStats-helper/data]
  --config PATH           Config file [default: ~/.vscsiStats-helper/config.yaml]
  --verbose, -v           Enable verbose output
  --help                  Show this message and exit
```

### Commands

#### `discover`

Discover VMs and LUNs on an ESXi host (does not require daemon).

```bash
vscsi-helper discover HOST [OPTIONS]

Arguments:
  HOST                    ESXi hostname or IP address

Options:
  --user, -u TEXT         SSH username [default: root]
  --key, -k PATH          SSH private key [default: ~/.ssh/id_rsa]
  --password, -p TEXT     SSH password (if not using key)
  --port INTEGER          SSH port [default: 22]
```

**Example**:
```bash
$ vscsi-helper discover esxi-host01.example.com --user root --key ~/.ssh/esxi_key

ESXi Host: esxi-host01.example.com
Connected successfully.

Virtual Machines:
  Name: SQL-Server-01
    World Group ID: 12345
    Power State: On

  Name: Web-App-VM
    World Group ID: 12346
    Power State: On

LUNs (vscsiStats Handles):
  vmhba1:C0:T0:L0
    Capacity: 500 GB
    Used by: SQL-Server-01
    Handle ID: 67890

  vmhba1:C0:T1:L0
    Capacity: 1 TB
    Used by: Web-App-VM
    Handle ID: 67891
```

#### `start`

Start a new collection job (requires daemon).

```bash
vscsi-helper start [OPTIONS]

Options:
  --host, -h TEXT         ESXi hostname [required]
  --vm TEXT               VM name or World Group ID [required]
  --lun TEXT              LUN identifier (e.g., vmhba1:C0:T0:L0) [required]
  --profile, -p TEXT      Collection profile [default: standard]
  --name, -n TEXT         Job name (auto-generated if not provided)
  --user, -u TEXT         SSH username [default: root]
  --key, -k PATH          SSH private key [default: ~/.ssh/id_rsa]
  --password TEXT         SSH password
```

**Example**:
```bash
$ vscsi-helper start \
    --host esxi-host01.example.com \
    --vm SQL-Server-01 \
    --lun vmhba1:C0:T0:L0 \
    --profile standard \
    --name sql-peak

Job 'sql-peak' started successfully.

Target: SQL-Server-01 on esxi-host01.example.com
LUN: vmhba1:C0:T0:L0
Profile: standard (30 samples @ 60s intervals)
Expected duration: 30 minutes

Check status: vscsi-helper status sql-peak
```

#### `status`

Check status of a collection job.

```bash
vscsi-helper status JOB_ID

Arguments:
  JOB_ID                  Job identifier
```

**Example**:
```bash
$ vscsi-helper status sql-peak

Job: sql-peak
Status: running
Progress: 18/30 samples (60%)
Runtime: 18 minutes / 30 minutes
Next sample in: 42 seconds

Target:
  Host: esxi-host01.example.com
  VM: SQL-Server-01 (World Group: 12345)
  LUN: vmhba1:C0:T0:L0 (Handle: 67890)

Collection started: 2025-11-06 08:30:22
Expected completion: 2025-11-06 09:00:22
```

#### `stop`

Stop a running collection job.

```bash
vscsi-helper stop JOB_ID [OPTIONS]

Arguments:
  JOB_ID                  Job identifier

Options:
  --cleanup/--no-cleanup  Stop vscsiStats on host [default: cleanup]
```

**Example**:
```bash
$ vscsi-helper stop sql-peak

Stopping job 'sql-peak'...
Collected 18/30 samples.
Stopping vscsiStats on esxi-host01.example.com... done.

Job stopped successfully.
Partial data saved to: ~/.vscsiStats-helper/data/sql-peak/
```

#### `list`

List all jobs.

```bash
vscsi-helper list [OPTIONS]

Options:
  --status TEXT           Filter by status (running, completed, failed)
  --host TEXT             Filter by ESXi host
```

**Example**:
```bash
$ vscsi-helper list

Jobs in ~/.vscsiStats-helper/data/:

sql-peak                    running      18/30 samples   esxi-host01.example.com
web-app-baseline            completed    30/30 samples   esxi-host02.example.com
job-20251105-143022         failed       12/30 samples   esxi-host01.example.com
db-deep-profile             completed   100/100 samples  esxi-host03.example.com
```

#### `analyze`

Analyze collected data and generate outputs (offline, no daemon needed).

```bash
vscsi-helper analyze JOB_ID [OPTIONS]

Arguments:
  JOB_ID                  Job identifier

Options:
  --filter TEXT           Filter method: percentile, statistical, absolute
                          [default: percentile]
  --threshold FLOAT       Threshold value (depends on filter method)
                          [default: 75 for percentile]
  --output, -o PATH       fio job file output path
                          [default: <job_id>-workload.fio]
  --visualize/--no-visualize
                          Generate 3D visualizations [default: no-visualize]
  --csv/--no-csv          Export filtered data to CSV [default: no-csv]
  --format TEXT           Visualization format: png, pdf [default: png]
```

**Example**:
```bash
$ vscsi-helper analyze sql-peak \
    --filter percentile \
    --threshold 75 \
    --output sql-peak.fio \
    --visualize \
    --csv

Analyzing job: sql-peak
Data directory: ~/.vscsiStats-helper/data/sql-peak/

Loading samples... 30 samples found.
Applying filter: percentile (threshold: 75)
Filtered out 7 low-activity samples (23.3%)
Analyzing 23 samples...

Peak Workload Profile:
  IOPS: 4,280 (Read: 2,996 / Write: 1,284)
  Throughput: 145.3 MB/s
  Read/Write ratio: 70/30
  Sequential: 35%, Random: 65%
  Block sizes: 8K (75%), 64K (20%), 4K (5%)
  Queue depth: avg=24, p50=22, p95=42, p99=48
  Read latency:  p50=2.1ms, p95=8.4ms, p99=15.2ms
  Write latency: p50=3.2ms, p95=12.1ms, p99=22.8ms

Output files:
  fio job:       sql-peak.fio
  CSV data:      ~/.vscsiStats-helper/data/sql-peak/analysis/filtered_data.csv
  Summary JSON:  ~/.vscsiStats-helper/data/sql-peak/analysis/summary.json
  Visualizations:
    - ~/.vscsiStats-helper/data/sql-peak/visualizations/iops-3d.png
    - ~/.vscsiStats-helper/data/sql-peak/visualizations/latency-3d.png
    - ~/.vscsiStats-helper/data/sql-peak/visualizations/blocksize-3d.png
```

#### `delete`

Delete a job and its data.

```bash
vscsi-helper delete JOB_ID [OPTIONS]

Arguments:
  JOB_ID                  Job identifier

Options:
  --force, -f             Skip confirmation prompt
```

**Example**:
```bash
$ vscsi-helper delete job-20251105-143022

Delete job 'job-20251105-143022'?
This will remove all collected data and cannot be undone.
Type the job name to confirm: job-20251105-143022

Deleting job... done.
```

---

## Core Workflows

### 1. Discovery Workflow

```
User runs: vscsi-helper discover HOST
                    │
                    ▼
         SSH to ESXi host (no daemon)
                    │
                    ├─► Run: esxcli vm process list
                    │   (Get VM names, World Group IDs)
                    │
                    ├─► Run: esxcli storage core device list
                    │   (Get LUN identifiers)
                    │
                    └─► Run: vscsiStats -l
                        (Get Handle IDs for LUNs)
                    │
                    ▼
         Format and display results
```

### 2. Collection Job Start Workflow

```
User runs: vscsi-helper start --host ... --vm ... --lun ... --profile ...
                    │
                    ▼
         Validate inputs (host reachable, profile exists)
                    │
                    ▼
         Generate job ID (user-provided or timestamp)
                    │
                    ▼
         Create job directory structure
                    │
                    ▼
         Write initial job.json (status: pending)
                    │
                    ▼
         Send START_JOB command to daemon (via IPC)
                    │
                    ▼
              [DAEMON SIDE]
                    │
                    ├─► Load job metadata
                    │
                    ├─► Establish SSH connection
                    │   (with retry logic)
                    │
                    ├─► Discover World Group ID and Handle ID
                    │   (if not provided)
                    │
                    ├─► Start vscsiStats:
                    │   vscsiStats -s -w <WorldGroupID> -i <HandleID>
                    │
                    ├─► Update job.json (status: running)
                    │
                    └─► Enter collection loop:
                        ├─► Sleep for interval seconds
                        ├─► Run: vscsiStats -p csv -w <ID> -i <ID>
                        ├─► Save to samples/sample_NNN.csv
                        ├─► Run: vscsiStats -x -w <ID> -i <ID>
                        ├─► Run: vscsiStats -s -w <ID> -i <ID>
                        └─► Repeat until samples collected
                    │
                    ▼
         Update job.json (status: completed)
                    │
                    ▼
         Cleanup: vscsiStats -x -w <ID> -i <ID>
                    │
                    ▼
         Close SSH connection
```

### 3. Status Check Workflow

```
User runs: vscsi-helper status JOB_ID
                    │
                    ▼
         Read job.json from disk
                    │
                    ├─► If status is "completed" or "failed":
                    │   Display from metadata (no daemon query)
                    │
                    └─► If status is "running":
                        │
                        ▼
                    Send STATUS command to daemon (via IPC)
                        │
                        ▼
                    Daemon returns current progress
                        │
                        ▼
                    Display live status
```

### 4. Analysis Workflow (Offline)

```
User runs: vscsi-helper analyze JOB_ID --filter percentile --threshold 75
                    │
                    ▼
         Read job.json (verify job exists and completed)
                    │
                    ▼
         Load all CSV samples from samples/ directory
                    │
                    ▼
         Parse and combine into pandas DataFrame
                    │
                    ▼
         Apply filtering (percentile, statistical, or absolute)
                    │
                    ▼
         Calculate metrics:
              ├─► IOPS (read/write/total)
              ├─► Throughput (MB/s)
              ├─► Block size distribution
              ├─► Sequential vs Random (based on seeks)
              ├─► Queue depth statistics
              └─► Latency percentiles
                    │
                    ▼
         Generate fio job file
                    │
                    ▼
         (Optional) Generate 3D visualizations
                    │
                    ▼
         (Optional) Export filtered CSV
                    │
                    ▼
         Write summary.json to analysis/ directory
                    │
                    ▼
         Display summary to user
```

---

## Error Handling and Reliability

### SSH Connection Failures

**Scenario**: SSH connection fails during job start or mid-collection.

**Strategy**: Exponential backoff retry

```python
def connect_with_retry(host, max_attempts=3):
    """
    Retry SSH connection with exponential backoff.

    Delays: 5s, 10s, 20s
    """
    for attempt in range(1, max_attempts + 1):
        try:
            connection = ssh.connect(host)
            return connection
        except SSHException as e:
            if attempt == max_attempts:
                raise ConnectionError(f"Failed after {max_attempts} attempts")
            delay = 5 * (2 ** (attempt - 1))
            log.warning(f"Connection failed (attempt {attempt}/{max_attempts}), "
                       f"retrying in {delay}s: {e}")
            time.sleep(delay)
```

**Job Status on Failure**:
- Update `job.json` with status: `failed`
- Record error details in `collection.errors` array
- Preserve all collected samples (partial data)
- Attempt cleanup of vscsiStats on host

### vscsiStats Cleanup Failures

**Scenario**: Job fails/stops but `vscsiStats -x` command fails.

**Strategy**: Record cleanup status and provide manual recovery

```json
{
  "cleanup": {
    "vscsiStats_stopped": false,
    "stopped_at": null,
    "cleanup_error": "SSH connection lost during cleanup",
    "manual_cleanup_required": true,
    "manual_cleanup_command": "vscsiStats -x -w 12345 -i 67890"
  }
}
```

**User notification**:
```bash
$ vscsi-helper status sql-peak

Job: sql-peak
Status: failed
...

⚠️  WARNING: vscsiStats may still be running on esxi-host01.example.com

Manual cleanup required:
  1. SSH to esxi-host01.example.com
  2. Run: vscsiStats -x -w 12345 -i 67890

Or retry cleanup:
  vscsi-helper cleanup sql-peak
```

### Daemon Crash Recovery

**Scenario**: Daemon process dies while jobs are running.

**Strategy**: State persistence and recovery on restart

1. All job state persisted to `job.json` immediately
2. On daemon restart, scan for jobs with status: `running`
3. For each running job:
   - Check if collection time has elapsed
   - If not, attempt to reconnect and resume
   - If reconnect fails, mark as `failed`
4. Never leave jobs in inconsistent state

### Partial Data Handling

**Principle**: Partial data is valuable and should never be lost.

- All samples saved immediately to disk after collection
- Analysis can work with partial datasets
- `analyze` command shows warning if job not completed:
  ```
  ⚠️  Warning: Job 'sql-peak' did not complete (18/30 samples collected)
  Analysis will be based on partial data and may not represent peak workload.
  ```

---

## Configuration Management

### Default Configuration

Built-in defaults (no config file required):

```python
DEFAULT_CONFIG = {
    "data_directory": "~/.vscsiStats-helper/data",
    "profiles": {
        "quick": {
            "samples": 10,
            "interval_seconds": 30,
        },
        "standard": {
            "samples": 30,
            "interval_seconds": 60,
        },
        "deep": {
            "samples": 100,
            "interval_seconds": 120,
        },
    },
    "analysis": {
        "default_filter_method": "percentile",
        "default_threshold": 75,
        "visualization_dpi": 300,
        "visualization_format": "png",
    },
    "ssh": {
        "default_user": "root",
        "default_key": "~/.ssh/id_rsa",
        "connection_timeout": 30,
        "retry_attempts": 3,
    },
    "daemon": {
        "socket_path": "~/.vscsiStats-helper/daemon.sock",
        "log_file": "~/.vscsiStats-helper/daemon.log",
        "pid_file": "~/.vscsiStats-helper/daemon.pid",
    },
}
```

### User Configuration

User config file: `~/.vscsiStats-helper/config.yaml`

Example user config (override defaults):

```yaml
# User config - overrides built-in defaults

data_directory: ~/storage-profiling/vscsi-data

profiles:
  # Override standard profile
  standard:
    samples: 50
    interval_seconds: 90

  # Add custom profile
  overnight:
    samples: 200
    interval_seconds: 180  # 10 hours

analysis:
  default_filter_method: statistical
  default_threshold: 1  # 1 std dev below mean

ssh:
  default_user: admin
  default_key: ~/.ssh/esxi_custom_key
  retry_attempts: 5
```

### Configuration Loading

```python
def load_config(config_path=None):
    """
    Load configuration with precedence:
    1. CLI --config argument
    2. ~/.vscsiStats-helper/config.yaml
    3. Built-in defaults
    """
    config = DEFAULT_CONFIG.copy()

    if config_path is None:
        config_path = Path.home() / ".vscsiStats-helper" / "config.yaml"

    if config_path.exists():
        user_config = yaml.safe_load(config_path.read_text())
        config = deep_merge(config, user_config)

    return config
```

---

## Testing Strategy

### Unit Tests

**Coverage targets**:
- Core parsing logic (vscsiStats CSV → DataFrame)
- Metric calculations (IOPS, throughput, latency percentiles)
- Filtering algorithms (percentile, statistical, absolute)
- fio job file generation
- Configuration loading and merging

**Test fixtures**:
- Sample vscsiStats CSV files (various workload patterns)
- Mock SSH responses
- Job metadata examples

### Integration Tests

**Test scenarios**:
1. **End-to-end collection** (with mock ESXi host)
   - Start job → collect samples → stop job
   - Verify samples saved correctly
   - Verify cleanup executed

2. **Analysis pipeline**
   - Load sample data → filter → calculate metrics → generate fio
   - Validate fio job file syntax
   - Check metric accuracy against known inputs

3. **Daemon IPC**
   - CLI → daemon communication
   - Multiple concurrent jobs
   - Daemon restart with running jobs

### Manual Testing Checklist

- [ ] Discover VMs/LUNs on real ESXi host
- [ ] Start collection job (quick profile)
- [ ] Monitor job status during collection
- [ ] Stop job mid-collection (verify partial data)
- [ ] Complete full collection job
- [ ] Analyze completed job with various filters
- [ ] Generate visualizations
- [ ] Test SSH connection failure recovery
- [ ] Test daemon crash recovery
- [ ] Verify vscsiStats cleanup

---

## Implementation Phases

### Phase 0: Project Setup ✓
- [x] Create README with project vision
- [x] Resolve design decisions
- [x] Create technical design document
- [ ] Set up Python project structure
- [ ] Configure development environment

### Phase 1: MVP - Core Collection (Week 1-2)
**Goal**: Start a job, collect samples, stop a job

- [ ] Module: `core/ssh.py` - SSH connection with retry
- [ ] Module: `core/vscsi.py` - vscsiStats command execution
- [ ] Module: `core/parser.py` - Parse CSV output
- [ ] Module: `core/job.py` - Job data model and persistence
- [ ] Module: `daemon/collector.py` - Collection loop
- [ ] Module: `daemon/main.py` - Basic daemon (no IPC yet)
- [ ] CLI: `start` command (spawn daemon directly)
- [ ] CLI: `stop` command (signal daemon)
- [ ] Manual testing on real ESXi host

**Deliverable**: Can start/stop a single collection job and view raw CSV data

### Phase 2: Analysis & Output Generation (Week 2-3)
**Goal**: Analyze collected data and generate fio job files

- [ ] Module: `analysis/filter.py` - Percentile filtering (other methods later)
- [ ] Module: `analysis/metrics.py` - IOPS, throughput, block size, seq/rand
- [ ] Module: `analysis/fio_generator.py` - Generate fio job files
- [ ] CLI: `analyze` command
- [ ] Unit tests for analysis pipeline
- [ ] Validate fio output on real fio installation

**Deliverable**: Can analyze completed jobs and produce working fio job files

### Phase 3: Daemon & IPC (Week 3-4)
**Goal**: Background daemon with proper CLI communication

- [ ] Module: `utils/ipc.py` - Unix socket IPC protocol
- [ ] Module: `daemon/server.py` - Socket server
- [ ] Module: `daemon/job_manager.py` - Manage multiple jobs
- [ ] Refactor CLI commands to use IPC
- [ ] CLI: `status` command (query daemon)
- [ ] CLI: `list` command
- [ ] Daemon auto-start on first CLI invocation
- [ ] Integration tests for daemon lifecycle

**Deliverable**: Daemon manages multiple concurrent jobs, CLI communicates via IPC

### Phase 4: Discovery & Polish (Week 4-5)
**Goal**: Complete user experience

- [ ] CLI: `discover` command (list VMs/LUNs)
- [ ] CLI: `delete` command
- [ ] Module: `config/manager.py` - User config support
- [ ] Module: `analysis/visualizer.py` - 3D matplotlib charts
- [ ] Enhanced error messages and help text
- [ ] Logging improvements
- [ ] Documentation and examples

**Deliverable**: Full-featured tool ready for real-world use

### Phase 5: Hardening (Week 5-6)
**Goal**: Production readiness

- [ ] Comprehensive error handling
- [ ] Connection failure recovery testing
- [ ] Daemon crash recovery
- [ ] vscsiStats cleanup reliability
- [ ] Performance testing (large datasets)
- [ ] Security review (SSH key handling, permissions)
- [ ] User documentation and tutorials

**Deliverable**: Robust, production-ready tool

### Future Enhancements (Post-MVP)
- Multi-phase analysis (peak hours vs off-hours)
- Statistical and absolute threshold filters
- Seek distance analysis improvements
- HTML/PDF report generation
- Multi-host aggregate analysis
- Real-time monitoring dashboard
- Web UI for threshold adjustment

---

## Open Implementation Questions

The following items need decisions during implementation:

1. **Daemon lifecycle management**:
   - Should daemon auto-start on first CLI command? (Recommended: yes)
   - Should daemon auto-stop when no jobs running? (Recommended: no, keep running)
   - How to handle daemon upgrades with running jobs?

2. **vscsiStats cleanup approaches**:
   - Option A: Always attempt cleanup on job stop/fail
   - Option B: Deploy ESXi-side cronjob as failsafe
   - Option C: Both (cleanup on stop + cronjob as backup)
   - **Decision needed during Phase 1**

3. **Sequential I/O detection threshold**:
   - Currently: seek distance = 0 or +1 block
   - Should we make this configurable?
   - Some workloads may have "mostly sequential" with small gaps

4. **Large dataset handling**:
   - What if 100+ samples × large histograms = GB of data?
   - Should we implement streaming analysis?
   - Should we compress old samples?

5. **Multi-LUN collection**:
   - Currently: one job = one VM + one LUN
   - Should we support multiple LUNs in a single job?
   - Deferred to Phase 5 or post-MVP?

---

## Appendix

### Glossary

- **World Group ID**: VMware ESXi process identifier for a VM
- **Handle ID**: vscsiStats identifier for a specific LUN
- **LUN**: Logical Unit Number (storage device identifier)
- **vscsiStats**: ESXi utility for collecting I/O statistics
- **Histogram**: Statistical distribution of I/O characteristics
- **fio**: Flexible I/O Tester - synthetic workload generator

### References

- [VMware vscsiStats Documentation](https://knowledge.broadcom.com/external/article/310655/using-vscsistats-to-collect-io-and-laten.html)
- [Paramiko Documentation](https://www.paramiko.org/)
- [Click CLI Framework](https://click.palletsprojects.com/)
- [fio Documentation](https://fio.readthedocs.io/)
- [Python daemon Library](https://pypi.org/project/python-daemon/)

---

**Document Status**: Draft - Ready for review and implementation

**Next Action**: Review technical design → Set up Python project structure
