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

Storage designers and architects who need to:
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
- Sequential I/O detection (distance = +1)
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

## Open Questions & Design Decisions

The following items require further discussion and decision-making:

### 1. Job Management & Storage
- **Data storage location**: Where should collected data be stored?
  - System-wide: `/var/lib/vscsiStats-helper/`
  - User-specified directory
  - Both with configurable default
- **Job identifiers**: How are jobs uniquely identified?
  - Timestamp-based: `job-20251105-143022`
  - UUID-based: `job-a1b2c3d4-e5f6...`
  - User-provided friendly names with fallback
- **Job lifecycle**: How long are completed jobs retained?
  - Manual deletion only
  - Auto-cleanup after N days
  - User-configurable retention policy

### 2. Profile Configuration
- **Profile storage**: Built-in vs. user-configurable?
  - Hardcoded defaults with config file overrides (recommended)
  - Fully user-defined in config file
- **Profile parameters**: What can users customize?
  - Sample count, interval (yes, definitely)
  - vscsiStats command-line options (probably yes)
  - Post-collection processing options (maybe)

### 3. Reliability & Error Handling
- **Connection resilience**: What happens if SSH drops mid-collection?
  - Auto-reconnect and resume (complex but valuable)
  - Fail gracefully and preserve partial data (simpler)
  - Alert/notify user (via what mechanism?)
- **ESXi host protection**: Safeguards to prevent overwhelming the host?
  - Limit simultaneous VM monitoring per host
  - Rate limiting on vscsiStats operations
  - Health checks before starting collection

### 4. Analysis Flexibility
- **Filtering UI/UX**: How do users adjust filtering after initial analysis?
  - Re-run analyze command with different parameters
  - Interactive mode that prompts for threshold adjustments
  - Web UI for visual threshold adjustment (future enhancement?)
- **Multi-phase analysis**: Should tool support?
  - Separate "peak hours" vs. "off-hours" profiles
  - Workload comparison across different time ranges
  - Trend analysis over multiple collection jobs

### 5. Output Customization
- **fio job file granularity**:
  - One fio job per VM/LUN
  - One fio job per collection job (multiple VMs aggregated)
  - User choice with CLI flag
- **Visualization generation**:
  - Auto-generate all visualizations by default
  - On-demand via `--visualize` flag (recommended)
  - Selective visualization (e.g., `--visualize-iops --visualize-latency`)
- **Report format**:
  - Text summary to stdout (current design)
  - HTML report with embedded graphs
  - PDF report (future enhancement?)

### 6. Multi-Host Support (Future)
- Currently scoped for single host at a time
- Should we design with multi-host in mind?
  - Database schema that supports it
  - Job namespace separation
  - Aggregate analysis across hosts

### 7. Real-Time Monitoring (Future)
- Current design: batch collection → batch analysis
- Should we consider real-time dashboard?
  - Live IOPS/latency graphs during collection
  - Early warning of anomalous behavior
  - Likely a phase 2 feature

## Project Status

**Current Phase**: Planning & Architecture Design

This README represents the planning outcome and project vision. Implementation has not yet begun.

### Next Steps

1. Resolve open questions (prioritize items 1-3)
2. Create detailed technical design document
3. Set up Python project structure
4. Implement MVP:
   - Basic SSH connection and vscsiStats execution
   - Simple data collection (single VM, single LUN)
   - CSV parsing and storage
   - Basic analysis with one filtering method
   - fio job file generation
5. Expand to full feature set
6. Testing and validation

## Contributing

This is currently a private project in planning stages. Contribution guidelines will be established once the MVP is implemented.

## License

TBD

## References

- [VMware vscsiStats Documentation](https://kb.vmware.com/s/article/1002440)
- [vmdamentals.com: vscsiStats 3D Visualization](https://vmdamentals.com) (original inspiration)
- [fio - Flexible I/O Tester](https://fio.readthedocs.io/)
- ESXi Performance Best Practices

## Contact

Eric - Project Lead

---

**Note**: This project is in active planning. The architecture and features described here represent the intended design and are subject to change as implementation progresses.
