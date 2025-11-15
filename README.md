# Dropbox Consumer

[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)](https://hub.docker.com/r/trusmith/dropbox-consumer)
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://www.python.org/)

File monitoring service that copies new files from a source directory to a destination. Built for document management systems like Paperless-NGX.

## Why I Built This

I created this out of a need to use [paperless-ngx](https://github.com/paperless-ngx/paperless-ngx) without having it devour my documentation. I needed a way to continue using my existing organization structure while also using paperless. It does create two sources for the same information, yes, but that's what I needed. And since there was nothing else like it out there, I chose to build this small, efficient, and simple tool.

## Features

Real-time monitoring, duplicate detection, file stability checks, persistent state, file filtering.

## Quick Start

Complete docker-compose.yml example:

```yaml
version: "3.8"
services:
  dropbox_consumer:
    image: trusmith/dropbox-consumer:latest
    container_name: dropbox_consumer
    user: "${PUID:-1000}:${PGID:-1000}"
    volumes:
      - /your/source/path:/source:ro
      - /your/destination/path:/consume:rw
      - ./state:/app/state:rw
    environment:
      - SOURCE=/source
      - DEST=/consume
      - RECURSIVE=true
      - PRESERVE_DIRS=false
      - COPY_EMPTY_DIRS=false
      - STATE_DIR=/app/state
      - STATE_CLEANUP_DAYS=30
      - STATE_BACKUP_COUNT=3
      - LOG_LEVEL=INFO
      - DEBOUNCE_SECONDS=1.0
      - STABILITY_INTERVAL=0.5
      - STABILITY_STABLE_ROUNDS=2
      - COPY_TIMEOUT=60
      - MAX_WORKERS=4
      - FILE_INCLUDE_PATTERNS=
      - FILE_EXCLUDE_PATTERNS=
      - DRY_RUN=false
      - DELETE_SOURCE=false
      - MAX_FILE_SIZE_MB=0
      - COMPRESS_FILES=false
      - WEBHOOK_URL=
      - RETRY_ATTEMPTS=3
      - RETRY_DELAY=2.0
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    restart: unless-stopped
```

Start the service:

```bash
docker-compose up -d
```

Monitor activity:

```bash
docker-compose logs -f dropbox_consumer
```

## Configuration

Environment variables control behavior. Set in `docker-compose.yml` or via command line.

### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `SOURCE` | `/source` | Directory to monitor |
| `DEST` | `/consume` | Target directory for copied files |
| `RECURSIVE` | `true` | Monitor subdirectories |
| `PRESERVE_DIRS` | `false` | Maintain source directory structure |
| `STATE_DIR` | `/app/state` | State persistence location |
| `LOG_LEVEL` | `INFO` | Logging level (DEBUG, INFO, WARNING, ERROR) |

### Performance Tuning

| Variable | Default | Description |
|----------|---------|-------------|
| `DEBOUNCE_SECONDS` | `1.0` | Wait time for event debouncing |
| `STABILITY_INTERVAL` | `0.5` | Interval between file stability checks |
| `STABILITY_STABLE_ROUNDS` | `2` | Consecutive stable checks required |
| `COPY_TIMEOUT` | `60` | Maximum seconds to wait for file stability |
| `MAX_WORKERS` | `4` | Concurrent processing threads |

### File Filtering

| Variable | Default | Description |
|----------|---------|-------------|
| `FILE_INCLUDE_PATTERNS` | _(empty)_ | Comma-separated patterns to include (e.g., `*.pdf,*.jpg`) |
| `FILE_EXCLUDE_PATTERNS` | _(empty)_ | Comma-separated patterns to exclude (e.g., `*.tmp,*.swp`) |

When include patterns are specified, only files matching at least one pattern are processed. Exclude patterns are applied first and take precedence.

Examples:
```yaml
# Only copy PDF and image files
environment:
  - FILE_INCLUDE_PATTERNS=*.pdf,*.jpg,*.png,*.jpeg

# Exclude temporary and system files
environment:
  - FILE_EXCLUDE_PATTERNS=*.tmp,*.swp,.DS_Store,Thumbs.db

# Combine both
environment:
  - FILE_INCLUDE_PATTERNS=*.pdf,*.docx
  - FILE_EXCLUDE_PATTERNS=*draft*,*temp*
```

### Advanced Options

| Variable | Default | Description |
|----------|---------|-------------|
| `COPY_EMPTY_DIRS` | `false` | Copy empty directories (requires PRESERVE_DIRS=true) |
| `DRY_RUN` | `false` | Simulate operations without copying files |
| `DELETE_SOURCE` | `false` | Delete source files after successful copy |
| `MAX_FILE_SIZE_MB` | `0` | Maximum file size in MB (0 = unlimited) |
| `COMPRESS_FILES` | `false` | Compress files with gzip during copy |
| `WEBHOOK_URL` | _(empty)_ | HTTP endpoint for copy notifications |
| `RETRY_ATTEMPTS` | `3` | Number of retry attempts for failed operations |
| `RETRY_DELAY` | `2.0` | Initial retry delay in seconds (exponential backoff) |
| `STATE_CLEANUP_DAYS` | `30` | Days to retain old state entries |
| `STATE_BACKUP_COUNT` | `3` | Number of state file backups to keep |
| `PUID` | `1000` | User ID for file operations |
| `PGID` | `1000` | Group ID for file operations |

## How It Works

On startup, the service snapshots all existing files in the source directory and marks them as processed. Only new files added after startup are copied.

File stability checks prevent copying incomplete files. The service monitors file size and waits for it to remain unchanged for the configured stability period.

Duplicate detection uses SHA-256 hashes. Files with identical content are skipped regardless of filename.

State is persisted to disk and survives container restarts. Files processed before a restart will not be reprocessed.

## Directory Structure Preservation

When `PRESERVE_DIRS=true`, the service maintains the relative path structure:

```
source/2024/invoices/document.pdf → destination/2024/invoices/document.pdf
```

When disabled, all files are copied to the destination root:

```
source/2024/invoices/document.pdf → destination/document.pdf
```

## Permissions

The service runs as the user specified by `PUID` and `PGID`. Set these to match your destination directory ownership:

```bash
id $(whoami)
```

Update `docker-compose.yml`:

```yaml
user: "${PUID:-1000}:${PGID:-1000}"
```

The state directory must be writable by this user.

## Logging

Logs are written to stdout and managed by Docker. Configure rotation in `docker-compose.yml`:

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

## Development

Run locally without Docker:

```bash
python3 -m pip install -r requirements.txt
export SOURCE=/path/to/source
export DEST=/path/to/destination
python3 main.py
```

## Troubleshooting

### Files Not Being Copied

Check logs for errors:

```bash
docker-compose logs -f dropbox_consumer
```

Verify permissions:

```bash
docker-compose exec dropbox_consumer ls -la /source
docker-compose exec dropbox_consumer ls -la /consume
```

### Permission Errors

Ensure PUID and PGID match the destination directory owner:

```bash
ls -la /your/destination/path
id $(whoami)
```

Set correct values in `docker-compose.yml`.

### State Persistence Issues

Verify state directory is writable:

```bash
ls -la ./state/
docker-compose exec dropbox_consumer ls -la /app/state
```

Fix permissions if needed:

```bash
sudo chown -R $(id -u):$(id -g) ./state/
```

### High CPU Usage

Reduce worker threads or increase debounce time:

```yaml
environment:
  - MAX_WORKERS=1
  - DEBOUNCE_SECONDS=5.0
```

## Docker Hub

Image: `trusmith/dropbox-consumer:latest`

Pull the image:

```bash
docker pull trusmith/dropbox-consumer:latest
```

Available tags:
- `latest` - Most recent stable release
- `v3.0` - Version 3.0 release
- `main` - Latest development build

## License

Licensed under the MIT License.

See the [LICENSE](LICENSE) file for details.

## Support

If this project helped you, please consider supporting my work:

<a href="https://www.buymeacoffee.com/trusmith" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="60" width="217"></a>