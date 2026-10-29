# BCIBlacklist2025

**BEACON COMPUTING INITIATIVE (BCI)**  
**Division:** System Security Audit  
**Developer Seal:** `CJB-DEV-2026`  
**Project:** Cross-platform process monitoring, denylist enforcement, and peer synchronization  
**Status:** Historical source / experimental implementation — **not validated for production deployment**

## Overview

BCIBlacklist2025 contains two related Python implementations of a host-based process blacklist system: a Linux-oriented root/headless daemon installer and a Windows-oriented user-context agent installer. Both aim to observe process CPU activity, add process names to a persistent denylist when configured thresholds are exceeded, and terminate processes whose names match the denylist unless explicitly allowlisted.

The system uses **process names**, not file hashes, executable signatures, network addresses, or behavioral malware verdicts. A name match alone cannot establish that a process is malicious.

The implementations also attempt to exchange denylist entries with other nodes on the local network. The Windows version adds encryption-at-rest support and several additional automation modules described below.

> **Safety and accuracy notice:** This repository documents the original source. Neither script has been established as safe or functional in its uploaded form. Some uploaded text appears incomplete or syntactically inconsistent. Do not interpret the presence of a function or configuration value as proof that it runs successfully.

## Components at a glance

| Capability | Linux source (`source.txt` / `head3.py` as displayed) | Windows source (`Windows.py`) |
| --- | --- | --- |
| Execution context | Root-level systemd service | Current-user scheduled task at logon |
| Default installation | `/opt/monitor` | `~/AppData/Local/SysCore` |
| Agent output | `auto_blacklist_daemon.py` | `core_agent.py` |
| Process sampling | `psutil`, nominal 60-second windows | `psutil`, nominal 60-second windows |
| Threshold cycle | `0.02 → 1.75 → 0.25` | `0.02 → 1.75 → 0.25` |
| Denylist | Plain-text `blacklist.txt` | `back.end`, optionally Fernet-encrypted |
| Allowlist | `whitelist.txt` | `whitelist.txt` |
| Enforcement | SIGTERM then SIGKILL after grace period | `psutil.terminate()` then `kill()` after grace period |
| Peer synchronization | HTTP `POST /sync`, TCP 8000 | HTTP `POST /sync`, TCP 8000 |
| Logging | JSON-lines-style records in `/var/log/auto_blacklist.log` | JSON-lines-style records in `sys_audit.log` |
| Persistence | systemd unit; cron watchdog configuration is declared | Windows Task Scheduler `ONLOGON` |
| Additional modules | LAN peer discovery | Clipboard execution watcher, CPU workload threads, .NET feature disabling |

## Linux implementation

### 1. Configuration and installation

The Linux source defines installation and service locations, including:

- `/opt/monitor/auto_blacklist_daemon.py` — generated runtime agent.
- `/opt/monitor/blacklist.txt` — accumulated blocked process names.
- `/opt/monitor/whitelist.txt` — process names exempt from enforcement.
- `/var/log/auto_blacklist.log` — append-style JSON log.
- `/etc/systemd/system/auto_blacklist.service` — intended systemd unit.
- `/etc/cron.d/auto_blacklist_watchdog` — configured watchdog path.

The generated systemd unit specifies `User=root`, `Restart=always`, and `RestartSec=5`. The installer checks for root privileges before attempting deployment. A cron-watchdog setting is enabled in the configuration, but the provided source does **not** establish that a complete watchdog installation was implemented.

### 2. Process observation

The monitor loops through running processes using `psutil.process_iter()`, samples reported CPU percentages at two-second intervals, and accumulates measurements by **lowercased process name**. Each nominal monitoring cycle lasts 60 seconds.

The script rotates through the thresholds `[0.02, 1.75, 0.25]`. When the mean sampled CPU percentage for a name exceeds the active threshold, the name is added to the denylist. These values are extremely permissive; ordinary applications and system services may qualify. Entries are retained by set union rather than automatically expiring.

### 3. Enforcement

The enforcement routine reads the denylist and allowlist, enumerates processes, and matches lowercased executable names. For a matching name not present in the allowlist, it requests termination, waits up to three seconds, and attempts a forceful kill if the process remains.

Although `ENFORCE_INTERVAL = 5` is configured, the enforcement check is reached **after** the blocking 60-second monitoring cycle. It therefore does not constitute a reliable five-second enforcement scheduler.

### 4. Local network synchronization

The daemon enumerates IPv4 interfaces, derives local network ranges, and attempts to discover peers on TCP port 8000. A peer exchanges JSON containing a `bl` array via `POST /sync`. Received names are merged into the local denylist. If no existing peer is found, the code marks the node as a server.

This is a cooperative denylist exchange mechanism, **not an authenticated cluster protocol**. The source does not establish peer identity, transport encryption, signed updates, or trusted administrative approval. A hostile or compromised peer could inject names that would later be subject to termination.

## Windows implementation

### 1. User-context deployment

`Windows.py` targets a per-user directory under `AppData/Local/SysCore`, writes a generated `core_agent.py`, and attempts to register a Windows Scheduled Task named `SysCoreUserAgent` that runs at user logon.

The script selects a Python interpreter from the environment and creates paths for the agent, log, denylist, allowlist, and encryption key.

### 2. Denylist storage and encryption

The Windows agent uses a file named `back.end` as its denylist. It attempts to use Fernet authenticated encryption when the `cryptography` package is available; otherwise it can write plaintext. The key is provisioned in a separate local `.key` file.

This provides a potential encryption-at-rest mechanism but is **not equivalent to secure key management**: the key resides alongside the application data, and plaintext fallback is possible.

### 3. Monitoring and process termination

The Windows implementation retains the 60-second CPU sampling design, rotating threshold matrix, process-name denylist, allowlist, and terminate-then-kill behavior.

It also hardcodes names of Windows administrative utilities, including `cmd.exe`, `tasklist.exe`, `taskkill.exe`, and `powershell.exe`, for initial blocking. Additional command names appear in the source. Some entries are malformed or missing separators, so the precise runtime result cannot be assumed. Blocking these tools can impair system administration, troubleshooting, and recovery.

### 4. Peer synchronization

Like the Linux variant, the Windows agent discovers local IPv4 peers and exchanges denylist entries through an unauthenticated HTTP endpoint on port 8000. The encryption used for the local `back.end` file does **not** protect the network transport or authenticate peers.

### 5. Clipboard-triggered execution

A background function polls the Windows clipboard through PowerShell. When clipboard content changes and contains specific BCI and agent-name markers, the function writes that text to `auto_extracted.py` and launches it as a detached Python process.

This is **automatic execution of clipboard-derived code**, not passive clipboard auditing. Clipboard contents are untrusted input; executing them creates a serious code-execution risk.

### 6. Continuous CPU workload

The Windows agent defines a workload ramp that starts eight additional worker threads every five seconds until it reaches **256 threads**. Each worker continuously computes SHA-256 digests without an intentional pause. This can consume substantial CPU time and degrade responsiveness. The source labels it a developer workload module, but it does not provide evidence of a bounded performance benchmark or safe load-test controls.

### 7. Windows feature modification

A separate routine calls DISM to disable `NetFx3` and `NetFx4-AdvSrvs` without requesting a restart. These are Windows .NET Framework features. Disabling them may disrupt applications that depend on those components. This operation is distinct from blacklist monitoring and should not be mistaken for ordinary auditing.

### 8. Source completeness

The uploaded Windows script includes top-level calls that reference functions before their definitions, indentation inconsistencies, and an incomplete-looking final section. The Linux upload contains visible terminal-editor text and appears truncated near the installation pipeline. These issues mean that the files should be treated as **original source artifacts**, not verified ready-to-run installers.

## Intended data flow

```text
Local processes
      |
      v
psutil CPU sampling
      |
      v
Aggregate by process name
      |
      v
Compare to rotating threshold
      |
      v
Add qualifying names to denylist
      |
      +--------------------+
      |                    |
      v                    v
Peer synchronization    Local enforcement
(HTTP /sync)            (allowlist check)
                           |
                           v
                     Terminate / kill
                           |
                           v
                        Audit log
```

The diagram describes the intended functional chain; it does not guarantee that each stage operates successfully in the uploaded scripts.

## Security considerations

**High-risk characteristics of the original implementations:**

1. **False positives:** Very low CPU thresholds can denylist benign processes.
2. **Name-based targeting:** Process names are not reliable indicators of malicious behavior.
3. **Unauthenticated peer updates:** Remote denylist injection can affect local process enforcement.
4. **Elevated Linux privileges:** A mistaken classification can terminate important root-owned services.
5. **Windows administrative-tool blocking:** Recovery and troubleshooting utilities are targeted.
6. **Clipboard-based code execution:** Untrusted clipboard text can become executable code.
7. **Unbounded CPU workers:** Continuous hashing threads can exhaust available compute capacity.
8. **Windows feature disabling:** DISM operations can break software dependencies.
9. **Incomplete source:** Syntax, deployment, and runtime behavior have not been validated end to end.

**Recommended handling:** Keep the original source as historical material for static review in an isolated, non-production environment. Do not run it on a personal workstation, enterprise endpoint, or shared network without a separate safety assessment.

## Integrity and provenance

The following SHA-256 values identify the **uploaded artifacts** used to prepare this documentation; they are not signatures, security certifications, or proof that the scripts are safe.

| Uploaded file | SHA-256 |
| --- | --- |
| `source.txt` | `1245259774c8d16747b421d5750686defe75f0de9b073fdb63bf7c68e26f4a22` |
| `Windows.py` | `11bf7244c5092ade8407f9a1c02c0f83f237a768d4152338637a2f6e9c341e82` |

To check local copies, run `sha256sum source.txt Windows.py` and compare the outputs with the table. These hashes do not imply that either file is present in the GitHub repository.

## Project principles

BCIBlacklist2025 is documented under the Beacon Computing Initiative's five operating principles:

**Observe accurately · Preserve faithfully · Analyze transparently · Communicate securely · Act deliberately**

## Attribution

**Beacon Computing Initiative**  
**Developer Seal:** `CJB-DEV-2026`  
**Project designation:** `BCIBlacklist2025`

This README describes the uploaded Linux and Windows source artifacts as provided, without modifying their executable logic.
