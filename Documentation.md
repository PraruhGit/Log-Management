
# Log Management Script Documentation

## Table of Contents

1. Introduction  
2. System Overview  
3. Pre-requisites  
4. Script Setup and Configuration  
   - 4.1. Directory Structure  
   - 4.2. Configuration File  
   - 4.3. Python Log Management Script  
   - 4.4. Systemd Service Setup  
5. Log Rotation and Archival  
6. Testing the Log Management Script  
7. Troubleshooting  
8. Best Practices and Recommendations  
9. Conclusion  

---

## 1. Introduction

This document provides a detailed guide to set up and manage a **Log Management Script** that automates the monitoring, rotation, archival, and cleanup of log files on a Unix-like system. The script ensures that log files are maintained efficiently, preventing disk space exhaustion and simplifying troubleshooting by keeping logs organized.

---

## 2. System Overview

The Log Management Script is designed with the following key features:

- **Real-time Monitoring:**  
  Watches designated directories for new or modified log files.

- **Log Rotation:**  
  Automatically rotates logs when they reach a specified size or age.

- **Archival and Cleanup:**  
  Compresses older log files and deletes archives beyond a defined retention period.

- **Automation:**  
  Runs continuously as a daemon, managed via `systemd`, ensuring ongoing log maintenance without manual intervention.

- **Configuration Driven:**  
  Uses a JSON configuration file for flexible parameter management including paths, rotation rules, and retention policies.

---

## 3. Pre-requisites

Before deploying the Log Management Script, ensure that your system meets the following requirements:

- **Operating System:**  
  Unix-like OS (e.g., Linux).

- **Python 3:**  
  Installed on the system.

- **Required Packages:**  
  Python modules such as `watchdog` for monitoring, and `logging`, `json`, `subprocess`, and `os` for script functionality.  
  Install with:
  ```bash
  sudo apt-get update
  sudo apt-get install python3-pip -y
  pip3 install watchdog
  ```

- **Disk Space and Permissions:**  
  Adequate disk space for log archives and proper permissions to read, write, and delete log files in the target directories.

- **Systemd:**  
  A systemd-enabled environment to run the script as a service.

---

## 4. Script Setup and Configuration

### 4.1. Directory Structure

Organize your directories as follows:

**Log Files Directory:**

```bash
/var/log/myapp/
├── app.log          # Active log file
└── archive/         # Directory for rotated/compressed logs
```

**Script and Configuration Directory:**

```bash
/etc/log_management/
├── log_management.py       # Main Python script
├── config_log_management.json  # Configuration file
└── log_management.log      # Script log for debugging and status
```

### 4.2. Configuration File

**Path:** `/etc/log_management/config_log_management.json`

This JSON file defines parameters for log rotation, archival, and cleanup. An example configuration:

```json
{
    "log_dir": "/var/log/myapp/",
    "archive_dir": "/var/log/myapp/archive/",
    "active_log": "app.log",
    "rotation": {
        "max_size_mb": 10,
        "max_age_days": 7
    },
    "archival": {
        "compress": true,
        "retention_days": 30
    },
    "logging": {
        "log_file": "/etc/log_management/log_management.log",
        "log_level": "INFO"
    },
    "debounce_time": 5
}
```

**Notes:**

- **`max_size_mb`:** Maximum log file size (in MB) before rotation.
- **`max_age_days`:** Maximum age (in days) before forcing a rotation.
- **`compress`:** If `true`, rotated logs are compressed.
- **`retention_days`:** Number of days to retain archived logs.
- **`debounce_time`:** Time in seconds to wait after a file change before processing.

### 4.3. Python Log Management Script

**Path:** `/etc/log_management/log_management.py`

Below is a sample Python script that demonstrates log monitoring, rotation, archival, and cleanup:

```python
#!/usr/bin/env python3

import os
import json
import time
import logging
import shutil
import gzip
from datetime import datetime, timedelta
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
from threading import Timer, Lock

# ---------------------------
# Load Configuration
# ---------------------------
CONFIG_FILE = "/etc/log_management/config_log_management.json"

def load_config(config_path):
    if not os.path.exists(config_path):
        raise FileNotFoundError(f"Configuration file {config_path} not found.")
    with open(config_path, 'r') as f:
        return json.load(f)

config = load_config(CONFIG_FILE)

LOG_DIR = config.get("log_dir")
ARCHIVE_DIR = config.get("archive_dir")
ACTIVE_LOG = os.path.join(LOG_DIR, config.get("active_log"))
ROTATION_CONFIG = config.get("rotation", {})
ARCHIVAL_CONFIG = config.get("archival", {})
DEBOUNCE_TIME = config.get("debounce_time", 5)

# ---------------------------
# Setup Logging
# ---------------------------
LOGGING_CONFIG = config.get("logging", {})
LOG_FILE = LOGGING_CONFIG.get("log_file", "/etc/log_management/log_management.log")
LOG_LEVEL = LOGGING_CONFIG.get("log_level", "INFO").upper()

logging.basicConfig(
    filename=LOG_FILE,
    level=getattr(logging, LOG_LEVEL, logging.INFO),
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# ---------------------------
# Utility Functions
# ---------------------------
def rotate_log():
    if not os.path.exists(ACTIVE_LOG):
        logging.error("Active log file does not exist.")
        return

    file_size_mb = os.path.getsize(ACTIVE_LOG) / (1024 * 1024)
    file_age_days = (time.time() - os.path.getmtime(ACTIVE_LOG)) / (3600 * 24)

    if file_size_mb < ROTATION_CONFIG.get("max_size_mb", 10) and file_age_days < ROTATION_CONFIG.get("max_age_days", 7):
        logging.info("Rotation conditions not met. Skipping rotation.")
        return

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    rotated_log = os.path.join(ARCHIVE_DIR, f"app_{timestamp}.log")
    try:
        # Move active log to archive location
        shutil.move(ACTIVE_LOG, rotated_log)
        logging.info(f"Rotated log file to {rotated_log}")

        # Re-create the active log file
        open(ACTIVE_LOG, 'w').close()

        # Compress if configured
        if ARCHIVAL_CONFIG.get("compress", True):
            compress_log(rotated_log)
    except Exception as e:
        logging.error(f"Error rotating log: {e}")

def compress_log(file_path):
    try:
        with open(file_path, 'rb') as f_in:
            with gzip.open(file_path + ".gz", 'wb') as f_out:
                shutil.copyfileobj(f_in, f_out)
        os.remove(file_path)
        logging.info(f"Compressed and removed uncompressed file: {file_path}")
    except Exception as e:
        logging.error(f"Error compressing log file {file_path}: {e}")

def cleanup_archives():
    retention_days = ARCHIVAL_CONFIG.get("retention_days", 30)
    cutoff = datetime.now() - timedelta(days=retention_days)
    try:
        for fname in os.listdir(ARCHIVE_DIR):
            if fname.endswith(".gz"):
                fpath = os.path.join(ARCHIVE_DIR, fname)
                ftime = datetime.fromtimestamp(os.path.getmtime(fpath))
                if ftime < cutoff:
                    os.remove(fpath)
                    logging.info(f"Removed archived log: {fpath}")
    except Exception as e:
        logging.error(f"Error cleaning up archives: {e}")

# ---------------------------
# File System Event Handler
# ---------------------------
class LogHandler(FileSystemEventHandler):
    def __init__(self):
        self.lock = Lock()
        self.debounce_timer = None

    def on_modified(self, event):
        if event.src_path == ACTIVE_LOG:
            self._debounced_action(rotate_log)

    def _debounced_action(self, action):
        with self.lock:
            if self.debounce_timer:
                self.debounce_timer.cancel()
            self.debounce_timer = Timer(DEBOUNCE_TIME, action)
            self.debounce_timer.start()

# ---------------------------
# Main Execution
# ---------------------------
def main():
    logging.info("Starting Log Management Script.")

    # Ensure archive directory exists
    if not os.path.exists(ARCHIVE_DIR):
        os.makedirs(ARCHIVE_DIR)
        logging.info(f"Created archive directory: {ARCHIVE_DIR}")

    # Initial rotation check and cleanup
    rotate_log()
    cleanup_archives()

    # Set up watchdog observer for real-time monitoring
    event_handler = LogHandler()
    observer = Observer()
    observer.schedule(event_handler, path=LOG_DIR, recursive=False)
    observer.start()
    logging.info(f"Started monitoring active log: {ACTIVE_LOG}")

    try:
        while True:
            time.sleep(60)
            # Periodically check for cleanup
            cleanup_archives()
    except KeyboardInterrupt:
        observer.stop()
        logging.info("Stopping Log Management Script.")
    observer.join()

if __name__ == "__main__":
    main()
```

**Notes:**

- The script uses **watchdog** to monitor the active log file and triggers log rotation if the file exceeds size or age thresholds.
- After rotation, the script compresses the archived log (if enabled) and periodically cleans up old archives based on the retention policy.
- Debounce logic is applied to prevent multiple rotations in quick succession.

### 4.4. Systemd Service Setup

**Path:** `/etc/systemd/system/log_management.service`

Create a systemd service file to run the log management script as a daemon:

```ini
[Unit]
Description=Log Management Script Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /etc/log_management/log_management.py
Restart=on-failure
RestartSec=30
User=root
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

**Setup Steps:**

1. Create the service file:
   ```bash
   sudo nano /etc/systemd/system/log_management.service
   ```
2. Paste the above content and save the file.
3. Reload systemd and enable the service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable log_management.service
   ```
4. Start the service:
   ```bash
   sudo systemctl start log_management.service
   ```
5. Verify the service status:
   ```bash
   sudo systemctl status log_management.service
   ```

---

## 5. Log Rotation and Archival

The script performs the following log management tasks:

- **Rotation:**  
  Moves the active log file to an archive directory when size or age thresholds are exceeded, and creates a new active log file.

- **Compression:**  
  If enabled in the configuration, rotated logs are compressed to save disk space.

- **Cleanup:**  
  Archives older than the configured retention period are deleted automatically.

---

## 6. Testing the Log Management Script

To verify that the script functions correctly, perform these tests:

### 6.1. Simulate Log Growth

1. Append data to the active log file to exceed the size threshold:
   ```bash
   for i in {1..1000}; do echo "Log entry $i" >> /var/log/myapp/app.log; done
   ```
2. Check that the script rotates the log and creates a new active log file in `/var/log/myapp/`.

### 6.2. Verify Compression and Archival

1. Inspect the archive directory:
   ```bash
   ls -l /var/log/myapp/archive/
   ```
2. Confirm that rotated logs are compressed (ending with `.gz`) if compression is enabled.

### 6.3. Test Cleanup

1. Manually modify the timestamp of an archive file to simulate an old log:
   ```bash
   sudo touch -d "40 days ago" /var/log/myapp/archive/old_log.log.gz
   ```
2. Wait for the periodic cleanup or run the `cleanup_archives()` function manually within the script context.
3. Confirm that the old archive is deleted.

---

## 7. Troubleshooting

If issues arise, follow these steps:

### 7.1. Check Service Status

- Verify that the service is active:
  ```bash
  sudo systemctl status log_management.service
  ```

### 7.2. Review Script Logs

- Examine the log file for the script:
  ```bash
  sudo tail -f /etc/log_management/log_management.log
  ```
- Look for errors related to file access, rotation failures, or cleanup issues.

### 7.3. Validate Configuration

- Ensure that paths in `/etc/log_management/config_log_management.json` are correct and directories exist.
- Check file permissions for both the log directories and the configuration file.

### 7.4. Test Manual Execution

- Run the script manually to observe any immediate errors:
  ```bash
  sudo /usr/bin/python3 /etc/log_management/log_management.py
  ```

---

## 8. Best Practices and Recommendations

- **Regular Monitoring:**  
  Periodically review log management service status and logs to ensure continuous operation.

- **Backup Configurations:**  
  Keep backups of configuration files and archived logs as needed for audit and recovery.

- **Security:**  
  Restrict file permissions for log and configuration directories to prevent unauthorized access.

- **Scalability:**  
  For systems with high log volume, consider integrating with centralized logging solutions (e.g., ELK stack) in addition to local rotation.

- **Testing Updates:**  
  After configuration changes, test the service manually before deploying to production.

---

