# Linux Bash-Based HIDS

## Overview

This project implements a Host Intrusion Detection System (HIDS) using only native Linux tools and Bash scripting.

The system monitors a Linux host for suspicious activity across multiple domains:
- System health
- User activity
- Processes and network behavior
- File integrity
- Alerting and correlation

The goal is to detect abnormal behavior, reduce noise, and provide actionable alerts similar to real-world tools like Wazuh, OSSEC, and Auditd.

---

## Architecture

The system is built using a **modular architecture**:
Controller → Modules → Raw Events → Correlation → Alerting → Enrichment

- Each module runs independently and detects anomalies in its domain
- Events are written to a centralized log (`raw_events.log`)
- The correlation engine combines weak signals into strong alerts
- Alerts are processed by the alerting module (JSON format)
- Enrichment adds forensic context (process tree, IP resolution, etc.)


### Components

- **Controller**
  - Executes modules periodically  
  - Defines execution frequency  

- **System Health Module (state-based)**
  - Monitors system condition over time  

- **Auditd-based Modules (event-based)**
  - Monitor real-time system activity  

- **Alerting Module**
  - Standardizes alert output  
  - Writes logs  

- **Baselines**
  - Store initial system state for comparison  

---

### Design Approach

The HIDS combines:

- **State-based monitoring**
  - Continuous system condition (CPU, memory, etc.)

- **Event-based monitoring**
  - Discrete actions (process execution, file changes)

This separation improves:
- clarity  
- detection accuracy  
- modularity  

---

## Modules

### Module 1 — System Health

This module provides a real-time assessment of whether the system is operating under normal conditions or experiencing abnormal stress.

#### Purpose
A compromised or misconfigured system often reveals itself through abnormal resource usage. However, high usage alone is not sufficient to trigger an alert — context and persistence matter.

#### What it monitors
- CPU usage (overall load)
- Memory usage
- Disk usage
- System load average

#### Detection approach
This module uses a **state-based monitoring model**:
- It collects system metrics at regular intervals
- Compares them against configurable thresholds (`thresholds.conf`)
- Applies **temporal logic** to avoid false positives

#### Temporal logic
Instead of alerting on a single spike:
- The module tracks whether a condition persists across multiple runs
- Only sustained abnormal behavior triggers an alert

Example:
- CPU spike for 1 second → ignored  
- CPU > threshold for multiple cycles → alert

#### Missing signal detection
The module also detects:
- Missing command outputs
- Failed metric collection

This is important because attackers may interfere with monitoring tools.

#### Output
- Alerts only when abnormal conditions persist
- Reduces noise and avoids alert fatigue

#### Detection type
State-based detection

---

### Module 2 — User Activity

This module monitors authentication behavior, user sessions, privilege escalation, and persistence mechanisms.

#### Purpose
Most attacks begin or evolve through user activity:
- Unauthorized logins
- Privilege escalation
- Persistence via accounts or SSH keys

This module focuses on detecting deviations from expected user behavior.

---

## A. Authentication anomalies

Detects suspicious login behavior:
- Repeated failed logins (brute-force attempts)
- Successful login after multiple failures (compromise indicator)
- Logins outside allowed hours
- Logins from non-whitelisted IPs
- Root login via SSH

#### Detection logic
- Parses authentication logs (`auth.log`)
- Correlates failures and successes per IP
- Applies thresholds and time windows

---

## B. Privilege escalation

Detects attempts to gain elevated privileges:
- New UID 0 accounts (silent root access)
- New members in sudo/wheel groups
- Excessive sudo usage
- Use of `su` (bypasses standard logging)

---

## C. Session anomalies

Detects abnormal session behavior:
- Multiple concurrent sessions for the same user
- Sessions from multiple IPs
- Long idle sessions (potential hijacking)
- Console (TTY) logins on a server

---

## D. Persistence indicators

Detects long-term attacker footholds:
- New user accounts
- Hidden accounts (duplicate UID)
- Changes in user shells
- Unauthorized SSH keys (`authorized_keys`)

---

## E. Log tampering

Detects attempts to hide activity:
- Truncated or deleted log files
- Missing authentication databases

---

## F. Auditd integration

Uses kernel-level audit logs to detect:
- Account modifications
- Privilege escalation attempts
- SSH configuration changes

#### Key strength
- Uses **auid (audit user ID)** → identifies the real user behind sudo

---

#### Detection type
Combination of:
- Event-based detection (log parsing)
- State-based detection (baseline comparison)

---
### Module 3 — Process and Network Audit

This module detects suspicious processes, network activity, and system-level anomalies.

#### Purpose
Attackers operate through processes and network connections:
- Malware execution
- Backdoors
- Reverse shells
- Data exfiltration

---

## Key detections

### 1. Unexpected ports
- Detects listening ports not in the whitelist
- Indicates backdoors or unauthorized services

### 2. New ports vs baseline
- Compares current ports with baseline snapshot
- Detects newly opened services

### 3. Suspicious execution paths
- Detects binaries running from:
  - /tmp
  - /dev/shm
  - /var/tmp
- These are common attacker staging locations

---

### 4. SUID abuse
- Detects new SUID files
- Critical privilege escalation vector

---

### 5. Hidden processes
- Compares `/proc` vs `ps`
- Detects rootkits hiding processes

---

### 6. Advanced attack indicators
- Service accounts spawning shells (webshells)
- Reverse tunnels (SSH -R)
- Shell with active network socket (reverse shell)
- Interpreter with outbound connection (C2 behavior)
- LD_PRELOAD injection
- RWX memory pages (shellcode execution)
- Name spoofing
- Process reparenting (daemonized malware)

---

### 7. Network anomalies
- Connection bursts
- Long-lived connections
- C2 beaconing patterns

---

#### Data sources
- `/proc` → process internals
- `ss` → network sockets
- `ps` → process list
- auditd → execution context

---

#### Detection type
Hybrid:
- State-based (baseline comparison)
- Event-based (real-time inspection)
- Behavioral detection (heuristics)

---

### Module 4 — File Integrity

This module ensures that critical system files have not been modified.

#### Purpose
Attackers modify files to:
- Add backdoors
- Escalate privileges
- Maintain persistence

---

## Detection approach

Uses **cryptographic hashing (SHA-256)**:
- Each file is hashed during baseline creation
- Current hash is compared to baseline

---

## What it detects

- File content changes
- Permission changes
- Ownership changes
- File deletion

---

## Auditd enrichment

When a change is detected:
- Queries audit logs to identify:
  - Who modified the file
  - Which command was used

Example:
- Instead of: "File modified"
- You get: "File modified by user X using command Y"

---

## Design decision

- Baseline is NOT automatically updated
- Admin must manually reinitialize baseline

This prevents attackers from hiding changes.

---

#### Detection type
State-based detection (baseline comparison)

---

### Module 5 — Alerting

This module centralizes all alerts generated by the system.

#### Purpose
A detection system is only useful if alerts are:
- Clear
- Structured
- Actionable
- Not overwhelming

---

## Features

### Structured alerts
- JSON format
- Compatible with SIEM tools (Splunk, ELK, Wazuh)

---

### Severity levels
- INFO → informational
- WARNING → suspicious
- CRITICAL → high-risk

---

### Deduplication
Prevents alert flooding:
- Same alert is suppressed for a configurable time window
- Uses hashing + timestamp tracking

---

## Output
- Stored in `alerts.json`
- Displayed in terminal with color coding

---

#### Detection type
Not a detection module — alert management layer

---

### Module 6 - Baseline

Shows what normal looks like on this machine?

All detection modules rely on this reference to identify anomalies.

---

The baseline module generates 4 reference files:

| File | Description |
|------|------------|
| `file_hashes.txt` | SHA-256 hashes of critical files |
| `users.txt` | list of system users |
| `ports.txt` | list of listening ports |
| `suid.txt` | list of SUID files |

---

This baseline allows other modules to detect:

- file modification → hash change  
- new user → possible backdoor account  
- new port → possible reverse shell  
- new SUID file → privilege escalation  

---

### Data sources used

- `/etc/passwd` → users  
- `ss -tuln` → listening ports  
- `find / -perm -4000` → SUID files  
- `sha256sum` → file integrity  

---

### Execution

The baseline is created using:

```bash`
./baseline.sh --init or via controller with init flag

### Module 7 — Correlation Engine

This module combines multiple weak signals into strong attack indicators.

#### Purpose
Single events are often ambiguous.
Correlation identifies patterns that indicate real attacks.

---

## Core principle
Weak signals → Strong detection

Example:
- Root user → normal
- Execution from /tmp → suspicious
- EXECVE event → normal

Combined:
→ CRITICAL (likely compromise)

---

## Example patterns

- Root execution from /tmp
- Reverse shell indicators
- Download + execution chain
- UID 0 account + root activity
- Permission changes on system binaries

---

## Data source
- `raw_events.log` from all modules

---

#### Detection type
Correlation-based detection (multi-signal analysis)

### Module 8 — Enrichment

This module adds context to alerts to assist investigation.

#### Purpose
Detection tells you WHAT happened.
Enrichment helps you understand:
- HOW it happened
- WHO did it
- WHERE it came from

---

## Enrichment types

### 1. Process ancestry
- Reconstructs execution chain:
  ssh → bash → payload

---

### 2. File attribution
- Uses auditd to identify:
  - User responsible
  - Command used

---

### 3. Reverse DNS
- Converts IP → hostname
- Helps identify attacker infrastructure

---

## Output
- Stored in `enriched.log`

---

#### Detection type
Post-processing (context enhancement)

## Usage

### Initialize baseline:

```bash
sudo bash main.sh --init
```

### Run HIDS

bash main.sh
