# CentOS System Monitor Automation & Fixes Log

Documentation of the fixes applied on October 1, 2026, to resolve automation failures after system reboots and repair Git repository corruption on the CentOS server.

## 1. Root Cause Analysis
* **Token Disappearance:** The `GITHUB_TOKEN` environment variable was originally declared in an active terminal session. A system shutdown wiped it from memory, preventing the automated `monitor_cpu.sh` script from authenticating with GitHub.
* **Cron Environment Limitations:** The `cron` daemon runs tasks in a stripped-down, non-interactive environment that does not automatically load user profile files or environment variables.
* **Git Metadata Corruption:** An abrupt system shutdown mid-write corrupted the local `.git/index` file and emptied the local branch markers, resulting in `fatal: index file smaller than expected` and `fatal: your current branch appears to be broken` errors.

## 2. Changes Applied

### A. Environment & Token Permanence
To ensure the token survives shutdowns and remains accessible to the background automation, it was permanently appended to the user's Bash configuration profile:
```bash
echo 'export GITHUB_TOKEN="[REDACTED_TOKEN]"' >> ~/.bashrc
source ~/.bashrc
```

### B. Crontab Optimization
The crontab configuration file was cleaned of multiple duplicate entries and optimized to explicitly load (`source`) the user environment settings right before triggering the script every minute:
```text
* * * * * /home/paul/sys_check.sh
* * * * * source /home/paul/.bashrc && /home/paul/projects/centos-system-monitor/monitor_cpu.sh > /dev/null 2>&1
```

### C. Repository Repair via Re-Cloning
Because the local Git metadata database was severely scrambled by the unexpected crash, the broken repository was archived, and a fresh tracking copy was downloaded directly from GitHub:
```bash
cd /home/paul/projects
mv centos-system-monitor centos-system-monitor-broken
git clone https://github.com
```

## 3. Operational Integrity
* **System Reboots:** Safe shutdowns (`sudo shutdown -h now`) will not corrupt Git files. 
* **Autonomous Execution:** The background cron automation runs completely independent of active user login sessions. It will continue pushing system stats to GitHub every minute as long as the server is powered on.
