# vscsiStats Helper

A comprehensive tool for profiling ESXi VM storage workloads and generating synthetic I/O profiles for storage system design and optimization.

## Overview

vscsiStats Helper is a Python-based tool that automates the collection, analysis, and visualization of VMware ESXi storage I/O patterns. It transforms time-series histogram data from ESXi's `vscsiStats` utility into actionable insights and synthetic workload profiles (fio job files) that storage designers can use to architect and optimize storage systems.

### The Problem

Traditional storage profiling workflows are painful:
- Manual SSH login to ESXi hosts
- Running command-line tools repeatedly
- Manual data collection over hours or days
- Post-processing in Excel or custom scripts
- Difficulty identifying peak vs. idle workload characteristics
- Challenge translating real workloads into synthetic test profiles

### The Solution

vscsiStats Helper provides an end-to-end pipeline:
1. **Remote collection** via SSH (no manual login required)
2. **Long-running data capture** managed by a supervisor service
3. **Intelligent analysis** that filters idle periods and focuses on peak load
4. **Workload profiling** that generates ready-to-use fio job files
5. **3D visualization** to reveal temporal I/O patterns
6. **CSV export** for further analysis

## Target Audience

Storage engineers and system architects who need to:
- Understand real-world application I/O characteristics
- Design storage systems for peak workload capacity
- Optimize storage configurations based on actual usage patterns
- Generate synthetic workloads for testing and validation
- Analyze I/O behavior across multiple VMs and LUNs

## Architecture

### Two-Component Design

#### 1. Supervisor Service (systemd daemon)
- Manages long-running SSH connections to ESXi hosts
- Handles periodic vscsiStats collection cycles
- Monitors multiple VMs/LUNs simultaneously on a single host
- Resilient to network interruptions
- Stores collected data for later analysis

#### 2. CLI Tool (user-facing)
- **Discovery**: List VMs and LUNs on target ESXi hosts
- **Collection**: Start/stop/status monitoring jobs
- **Analysis**: Process collected data with intelligent filtering
- **Generation**: Create fio job files for synthetic workload reproduction
- **Visualization**: Generate 3D surface charts showing I/O patterns over time
- **Export**: Output raw data in CSV format

### Technology Stack

- **Language**: Python 3
- **Platform**: Linux only
- **Service Management**: systemd
- **SSH**: Paramiko or similar for remote ESXi access
- **Visualization**: matplotlib (3D surface charts)
- **Data Processing**: pandas, numpy

## Key Features

### Remote Data Collection

- **Passwordless authentication**: SSH key-based access to ESXi hosts
- **Multi-target monitoring**: Simultaneously profile multiple VMs and LUNs on a single host
- **Collection profiles**: Predefined and customizable sampling configurations
  - Quick: 10 samples @ 30-second intervals
  - Standard: 30 samples @ 60-second intervals
  - Deep: 100 samples @ 120-second intervals
- **Long-running jobs**: Support for multi-day collection cycles
- **Job persistence**: Data preserved across service restarts

### Intelligent Analysis

- **Activity filtering**: Multiple threshold methods to exclude idle/low-activity periods
  - Percentile-based (e.g., focus on top 75% of IOPS)
  - Statistical (exclude samples > 1 std dev below mean)
  - Absolute threshold (user-defined IOPS minimum)
  - Activity clustering (automatic idle/active boundary detection)
- **Peak load focus**: Generate workload profiles designed for storage system capacity planning
- **Temporal analysis**: Identify patterns, bursts, and behavior evolution over time

### Workload Characterization

The analysis engine extracts key I/O parameters:

**I/O Pattern Metrics**:
- Read/write ratio
- Sequential vs. random I/O percentage
- Block size distribution
- IOPS patterns (total, read, write)
- Throughput calculations per time interval

**Performance Metrics**:
- Latency distributions (read/write)
- Queue depth (outstanding I/O)
- Seek distance patterns

**Spatial Characteristics**:
- Logical block distance analysis
- Sequential I/O detection (distance = +1, or user-defined threshold)
- Random I/O identification

### Output Formats

1. **fio Job Files** (primary output)
   - Ready-to-use synthetic workload configurations
   - Optimized for peak load reproduction
   - Include block size mix, I/O depth, read/write ratios

2. **3D Visualizations** (PNG/PDF)
   - Surface charts showing I/O evolution over time
   - Color-coded: Blue (total), Green (reads), Red (writes)
   - Multiple views: IOPS, latency, block size, seek distance

3. **CSV Export**
   - Raw histogram data for custom analysis
   - Filtered and unfiltered datasets
   - Summary statistics

## Example Workflow

```bash
# Discover VMs and LUNs on target host
$ vscsi-helper discover esxi-host01.example.com --user root --key ~/.ssh/esxi_key

ESXi Host: esxi-host01.example.com
VMs:
  - SQL-Server-01 (WorldGroupID: 12345)
  - Web-App-VM (WorldGroupID: 12346)

LUNs:
  - vmhba1:C0:T0:L0 (500GB, SQL-Server-01)
  - vmhba1:C0:T1:L0 (1TB, Web-App-VM)

# Start collection job
$ vscsi-helper start \
    --host esxi-host01.example.com \
    --vm "SQL-Server-01" \
    --lun vmhba1:C0:T0:L0 \
    --profile standard \
    --key ~/.ssh/esxi_key

Job started: job-20251105-143022
Monitoring: SQL-Server-01 on vmhba1:C0:T0:L0
Profile: 30 samples @ 60-second intervals (30 minutes)
Status: vscsi-helper status job-20251105-143022

# Check job status
$ vscsi-helper status job-20251105-143022

Job ID: job-20251105-143022
Status: Running
Progress: 18/30 samples collected (60%)
Runtime: 18 minutes
Target: SQL-Server-01 (esxi-host01.example.com)

# Stop collection (if needed)
$ vscsi-helper stop job-20251105-143022

# Analyze collected data
$ vscsi-helper analyze job-20251105-143022 \
    --filter percentile \
    --threshold 75 \
    --output sql-peak-workload.fio \
    --visualize \
    --csv

Analysis Summary:
  Total samples: 30
  Filtered samples: 7 (23.3% excluded as low-activity)
  Active analysis period: 23 minutes

Peak Workload Profile:
  IOPS: 4,280 (Read: 2,996 / Write: 1,284)
  Throughput: 145 MB/s
  Read/Write ratio: 70/30
  Sequential: 35%, Random: 65%
  Block sizes: 8K (75%), 64K (20%), 4K (5%)
  Queue depth: avg 24, peak 48
  Read latency: p50=2.1ms, p95=8.4ms, p99=15.2ms
  Write latency: p50=3.2ms, p95=12.1ms, p99=22.8ms

Output files:
  - sql-peak-workload.fio (fio job file)
  - job-20251105-143022-iops-3d.png
  - job-20251105-143022-latency-3d.png
  - job-20251105-143022-blocksize-3d.png
  - job-20251105-143022-data.csv
```

## Use Cases

1. **Application I/O Profiling**
   - Capture real-world database, application server, or VDI workloads
   - Generate synthetic profiles for storage testing

2. **Storage System Design**
   - Understand peak IOPS, throughput, and latency requirements
   - Size storage systems based on actual workload characteristics

3. **Performance Optimization**
   - Identify inefficient I/O patterns (excessive random I/O, poor block sizes)
   - Inform RAID stripe size, cache configuration, and tiering decisions

4. **Capacity Planning**
   - Model future storage needs based on current workload trends
   - Test "what-if" scenarios with fio-generated synthetic loads

5. **Before/After Validation**
   - Compare workload profiles before and after optimization changes
   - Verify that storage changes meet performance goals

## Technical Considerations

### ESXi Compatibility
- Requires ESXi 3.5+ (tested on ESXi 6.x, 7.x, 8.x)
- Requires SSH access to ESXi host
- Uses `vscsiStats` with `-w` (World Group ID) and `-i` (Handle ID) options

### Data Collection
- Sampling creates periodic snapshots of histogram data
- Each sample represents accumulated I/O since last reset
- Typical intervals: 30-120 seconds
- Typical sample counts: 10-100+ depending on workload duration

### Resource Impact
- Minimal impact on ESXi host performance
- vscsiStats overhead is negligible for production use
- Data collection network traffic is minimal (CSV output only)

## Installation

```bash
# Clone repository
git clone https://github.com/yourusername/vscsiStats-helper.git
cd vscsiStats-helper

# Install dependencies
pip install -r requirements.txt

# Install systemd service
sudo ./install.sh

# Verify installation
vscsi-helper --version
systemctl status vscsiStats-helper
```

## Configuration

Configuration file location: `/etc/vscsiStats-helper/config.yaml`

```yaml
# Data storage
data_directory: /var/lib/vscsiStats-helper

# Collection profiles
profiles:
  quick:
    samples: 10
    interval_seconds: 30
  standard:
    samples: 30
    interval_seconds: 60
  deep:
    samples: 100
    interval_seconds: 120
  custom:
    samples: 50
    interval_seconds: 90

# Analysis defaults
analysis:
  default_filter_method: percentile
  default_threshold: 75
  visualization_dpi: 300
  visualization_formats: [png, pdf]

# SSH defaults
ssh:
  default_user: root
  default_key: ~/.ssh/id_rsa
  connection_timeout: 30
  retry_attempts: 3
```

## Design Decisions

The following design decisions have been finalized and will guide implementation:

### 1. Job Management & Storage

**Data Storage Location**
- User-specified directory only (via `--data-dir` CLI flag or config file setting)
- No system-wide `/var/lib/` requirement
- No sudo/root privileges needed
- Each user manages their own job data independently
- Default location: `~/.vscsiStats-helper/data/` if not specified

**Job Identifiers**
- User-provided friendly names with automatic timestamp fallback
- Examples: `sql-server-peak`, `web-app-baseline`, `prod-db-monday`
- Auto-generated fallback: `job-YYYYMMDD-HHMMSS` (e.g., `job-20251105-143022`)
- Job names must be unique within a data directory

**Job Lifecycle & Retention**
- Manual deletion only via `vscsi-helper delete <job-id>`
- No automatic cleanup or archival
- Simple and predictable: users explicitly control what gets deleted
- Job data includes: raw collected data, metadata, analysis results, generated outputs

### 2. Profile Configuration

**Profile Storage**
- Hardcoded defaults (quick, standard, deep) built into the tool
- User can override built-in profiles or define custom profiles in config file
- Config file location: `~/.vscsiStats-helper/config.yaml`
- Built-in profiles work immediately without any configuration

**Customizable Parameters**
- Sample count: Number of vscsiStats samples to collect
- Interval: Seconds between each sample collection
- No advanced vscsiStats CLI options or post-processing automation (keep it simple)

**Built-in Profiles**
```yaml
quick:
  samples: 10
  interval_seconds: 30
  # Total time: 5 minutes

standard:
  samples: 30
  interval_seconds: 60
  # Total time: 30 minutes

deep:
  samples: 100
  interval_seconds: 120
  # Total time: 3 hours 20 minutes
```

### 3. Reliability & Error Handling

**Connection Resilience**
- Retry SSH connection with exponential backoff (e.g., 3 attempts with increasing delays)
- On final failure: preserve partial data collected so far
- Job marked as `failed` status with error details in metadata
- User can analyze partial data or restart collection

**Critical Issue: Orphaned vscsiStats Processes**
- Failed or abandoned jobs may leave `vscsiStats` running indefinitely on ESXi host
- **Planned mitigation approaches** (implementation decision TBD):
  1. Job metadata tracks cleanup status; provide `vscsi-helper cleanup <job-id>` command
  2. Deploy optional ESXi cronjob to run `vscsiStats -x` after safe timeout period
  3. On job start, register cleanup handler; on job stop/fail, attempt cleanup
- This is a critical reliability consideration that must be addressed in MVP

**ESXi Host Protection**
- No built-in safeguards or rate limiting
- Trust the user to manage their infrastructure responsibly
- vscsiStats has minimal performance overhead on ESXi hosts
- User responsible for not starting excessive concurrent collections

### 4. Deferred Decisions (Post-MVP)

The following items are recognized as valuable but will be addressed after MVP:

**Analysis Flexibility**
- Multi-phase analysis (peak hours vs. off-hours profiles)
- Trend analysis across multiple collection jobs
- Interactive filtering threshold adjustment
- Users can re-run `analyze` command with different parameters for now

**Output Customization**
- fio job file granularity options (one per VM/LUN vs. aggregated)
- Selective visualization generation (generate all by default for now)
- HTML/PDF report formats (stdout text summary is sufficient for MVP)

**Multi-Host Support**
- Currently scoped for single host at a time
- Architecture should not preclude future multi-host support
- Job metadata includes host information for future aggregation

**Real-Time Monitoring**
- Current design: batch collection → batch analysis (offline)
- Live dashboards and streaming analysis are phase 2 features
- Focus on batch workflow reliability first

## Project Status

**Current Phase**: Planning & Architecture Design

This README represents the planning outcome and project vision. Implementation has not yet begun.

### Next Steps

1. ✅ ~~Resolve open questions~~ (Complete - see Design Decisions section)
2. Create detailed technical design document
3. Set up Python project structure
4. Implement MVP:
   - Basic SSH connection and vscsiStats execution
   - Simple data collection (single VM, single LUN)
   - CSV parsing and storage
   - Basic analysis with one filtering method
   - fio job file generation
   - Job cleanup mechanism (address orphaned vscsiStats processes)
5. Expand to full feature set
6. Testing and validation

## Contributing

This is currently a private project in planning stages. Contribution guidelines will be established once the MVP is implemented.

## License

TBD

## References

- [VMware vscsiStats Documentation](https://knowledge.broadcom.com/external/article/310655/using-vscsistats-to-collect-io-and-laten.html)
- [vmdamentals.com: vscsiStats 3D Visualization](https://vmdamentals.com/?p=722) (original inspiration)
- [fio - Flexible I/O Tester](https://fio.readthedocs.io/)
- ESXi Performance Best Practices

## Contact

Eric Parent - Project Lead

---

**Note**: This project is in active planning. The architecture and features described here represent the intended design and are subject to change as implementation progresses.
