# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

vscsiStats Helper is a Python tool for profiling ESXi VM storage workloads and generating synthetic I/O profiles (fio job files) for storage system design. It automates the collection, analysis, and visualization of VMware ESXi storage I/O patterns using the vscsiStats utility.

**Current Phase**: Planning & Architecture Design (implementation has not yet begun)

## Key Documents

- **README.md**: Project vision, use cases, and design decisions
- **TECHNICAL_DESIGN.md**: Complete architecture, data schemas, module structure, and implementation roadmap
- **doc/algonquin.csv**: Real vscsiStats output example (reference for parser implementation)
- **doc/vscsistats-summary.md**: Visualization principles and vscsiStats usage patterns

## Architecture

### Two-Component Design

1. **Background Daemon** (`vscsi-daemon`): User-level daemon managing long-running SSH connections and data collection
2. **CLI Tool** (`vscsi-helper`): User-facing commands for discovery, collection, analysis, and visualization

**IPC**: Unix domain socket for CLI-daemon communication

**Key Design Principle**: User-level operation only - no system-wide installation, no sudo/root required

### Data Flow

```
ESXi Host (vscsiStats)
  → SSH (Paramiko)
  → Daemon (collection loop)
  → Job directory (samples/*.csv)
  → Analysis (pandas/numpy)
  → Outputs (fio job files, 3D charts, CSV)
```

## Critical Data Format: vscsiStats CSV

The vscsiStats output format is **NOT** a single table. It consists of multiple independent histogram sections:

```
Histogram: IO lengths of commands,virtual machine worldGroupID,2099979,...
min,512
max,65536
mean,5332
count,793
Frequency,Histogram Bucket Limit
42,512
608,4096
90,8192
...

Histogram: latency of IOs in Microseconds (us),...
min,40
max,51537
mean,392
count,793
Frequency,Histogram Bucket Limit
403,100
304,500
78,1000
...
```

**Histogram Types** (each has total/read/write variants):
1. IO lengths (block sizes)
2. Distance between successive commands (seek distance - sequential vs random)
3. Distance from closest of previous 16 (spatial locality)
4. Latency in microseconds
5. Number of outstanding IOs (queue depth)
6. IO interarrival time

**Parser Requirements**:
- Each histogram is independent with different bin boundaries
- Bins are upper limits (values ≤ bucket limit)
- Must extract World Group ID and Handle ID from header lines
- Summary statistics (min/max/mean/count) provide quick metrics

## Module Structure

```
src/vscsi_helper/
├── cli/          # Click commands (discover, start, stop, status, analyze, list, delete)
├── daemon/       # Background service (server, job_manager, collector)
├── core/         # SSH, vscsiStats execution, CSV parsing, job model
├── analysis/     # Filtering, metrics calculation, fio generation, visualization
├── config/       # Config loading, built-in profiles, validation
└── utils/        # Path resolution, IPC protocol, logging
```

## Job Directory Structure

Each job is stored in a user-specified directory (default: `~/.vscsiStats-helper/data/`):

```
<job-name>/
├── job.json              # Metadata: status, target, collection progress, cleanup state
├── samples/              # Raw collected samples
│   ├── sample_001.csv
│   ├── sample_002.csv
│   └── ...
├── analysis/             # Created by analyze command
│   ├── summary.json      # Calculated metrics (IOPS, throughput, latency percentiles)
│   ├── filtered_data.csv
│   └── <job-name>.fio    # Generated fio job file
└── visualizations/       # 3D surface charts
    ├── iops-3d.png
    ├── latency-3d.png
    └── blocksize-3d.png
```

## Design Decisions (Finalized)

### Job Management
- **Storage**: User-specified directory only (no system-wide `/var/lib/`)
- **Job IDs**: User-provided friendly names with timestamp fallback (e.g., `sql-server-peak` or `job-20251106-083022`)
- **Retention**: Manual deletion only via `vscsi-helper delete <job-id>`

### Profile Configuration
- **Built-in profiles**: `quick` (10 samples @ 30s), `standard` (30 @ 60s), `deep` (100 @ 120s)
- **Customization**: User config overrides in `~/.vscsiStats-helper/config.yaml`
- **Parameters**: Sample count and interval only (keep it simple)

### Reliability & Error Handling
- **Connection failures**: Retry with exponential backoff (3 attempts), then fail gracefully
- **Partial data**: Always preserved - never lost
- **Critical issue**: Orphaned vscsiStats processes on failed/abandoned jobs
  - Must track cleanup status in `job.json`
  - Provide manual cleanup command if automatic cleanup fails
- **No host protection**: Trust user to manage infrastructure (vscsiStats has minimal overhead)

## Analysis Pipeline

1. **Filtering**: Remove idle/low-activity samples to focus on peak load
   - Percentile-based (default: 75th percentile)
   - Statistical (std dev based)
   - Absolute threshold (user-defined IOPS minimum)

2. **Metrics Calculation**:
   - IOPS (total, read, write)
   - Throughput (MB/s)
   - Block size distribution
   - Sequential vs. random ratio (based on seek distance)
   - Queue depth statistics
   - Latency percentiles (p50, p95, p99)

3. **Output Generation**:
   - fio job file (ready-to-use synthetic workload)
   - 3D visualizations (matplotlib surface charts)
   - CSV export (filtered data)
   - Summary JSON (all calculated metrics)

## Implementation Phases

### Phase 0: Project Setup
- [x] README with project vision
- [x] Design decisions finalized
- [x] Technical design document
- [ ] **Next**: Python project structure (pyproject.toml, src/ layout, dependencies)

### Phase 1: MVP - Core Collection
**Goal**: Start job, collect samples, stop job

Core modules: `core/ssh.py`, `core/vscsi.py`, `core/parser.py`, `core/job.py`, `daemon/collector.py`

### Phase 2: Analysis & Output Generation
**Goal**: Analyze data and generate fio job files

Modules: `analysis/filter.py`, `analysis/metrics.py`, `analysis/fio_generator.py`

### Phase 3: Daemon & IPC
**Goal**: Background daemon with CLI communication

Modules: `utils/ipc.py`, `daemon/server.py`, `daemon/job_manager.py`

### Phase 4: Discovery & Polish
**Goal**: Complete user experience

Features: `discover` command, `delete` command, user config, 3D visualizations

### Phase 5: Hardening
**Goal**: Production readiness

Focus: Error handling, connection recovery, daemon crash recovery, cleanup reliability

## Technology Stack

- **Python**: 3.8+ (no specific version locked yet)
- **SSH**: Paramiko (pure Python, ESXi compatible)
- **CLI**: Click (clean API, good docs)
- **Data**: pandas, numpy (histogram analysis, statistics)
- **Visualization**: matplotlib (3D surface plots)
- **Daemon**: python-daemon (standard library approach)
- **IPC**: Unix domain socket (JSON-RPC over socket)
- **Config**: YAML (human-readable)
- **Serialization**: JSON (job metadata, state)

## Sequential I/O Detection

Sequential I/O is identified by seek distance = 1 LBN (Logical Block Number) in the "distance between successive commands" histogram. This is a key metric for workload characterization:

- Distance = 1: Sequential/linear I/O
- Large positive/negative distances: Random I/O
- Negative values: Backward seeks

## Visualization Strategy

**Color coding** (per vmdamentals.com standard):
- Blue: Total I/O
- Green: Read I/O
- Red: Write I/O

**3D surface charts** show:
- X-axis: Histogram bins (block size, distance, latency, etc.)
- Y-axis: Time (sample number)
- Z-axis: Frequency/count

## Open Implementation Questions

These require decisions during implementation:

1. **vscsiStats cleanup**: Always attempt cleanup on stop/fail? Deploy ESXi-side cronjob as failsafe? Both?
2. **Sequential I/O threshold**: Make configurable beyond distance=1? Some workloads are "mostly sequential" with small gaps.
3. **Large dataset handling**: Streaming analysis for 100+ samples with large histograms?
4. **Multi-LUN collection**: Support multiple LUNs in single job? (Currently: one job = one VM + one LUN)

## ESXi Compatibility

- Requires ESXi 3.5+ (tested on 6.x, 7.x, 8.x)
- SSH access required
- Uses `vscsiStats` with `-w` (World Group ID) and `-i` (Handle ID) options
- Command pattern: `vscsiStats -s -w <WorldGroupID> -i <HandleID>` (start), `-p csv` (print CSV), `-x` (stop)

## Configuration File

Default location: `~/.vscsiStats-helper/config.yaml`

Built-in defaults work without config file. User can override:

```yaml
data_directory: ~/storage-profiling/vscsi-data

profiles:
  standard:
    samples: 50
    interval_seconds: 90

  custom-overnight:
    samples: 200
    interval_seconds: 180

analysis:
  default_filter_method: percentile
  default_threshold: 75

ssh:
  default_user: root
  default_key: ~/.ssh/esxi_custom_key
  retry_attempts: 3
```

## Notes for Implementation

- **Offline analysis**: The `analyze` command works independently without daemon (reads job data directly)
- **Daemon auto-start**: Daemon should auto-start on first CLI command requiring it
- **Job status persistence**: All state written to `job.json` immediately for crash recovery
- **Partial data analysis**: `analyze` can work with incomplete jobs (with warning to user)
- **No empty commits**: Check for actual changes before creating git commits
