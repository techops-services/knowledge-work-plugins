---
name: service-config
description: Manages KeeperHub service configuration including auto-detection and manual setup
---

# Service Configuration

This skill manages the mapping between KeeperHub services and their external identifiers (Jira projects, GitHub repos). It enables auto-detection of which service you're working on based on your current git repository.

## Configuration Location

**Path:** `~/.claude/kh/services.yaml`

Created on first `/kh:setup` run. Persists across sessions and workspaces.

## Services YAML Schema

Minimal configuration per service - just what's needed to identify the service:

```yaml
services:
  keeperhub-api:
    jira_project: KH
    github_repo: techops/keeperhub-api
  keeperhub-web:
    jira_project: KH
    github_repo: techops/keeperhub-web
  keeperhub-worker:
    jira_project: KH
    github_repo: techops/keeperhub-worker
```

### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `jira_project` | Jira project key | `KH` |
| `github_repo` | GitHub owner/repo | `techops/keeperhub-api` |

### Optional Fields (Advanced Users)

```yaml
services:
  keeperhub-api:
    jira_project: KH
    github_repo: techops/keeperhub-api
    aws_region: us-east-1
    eks_cluster: keeperhub-prod
    namespace: keeperhub-api
```

| Field | Description | Default |
|-------|-------------|---------|
| `aws_region` | AWS region for this service | `us-east-1` |
| `eks_cluster` | EKS cluster name | Derived from service name |
| `namespace` | Kubernetes namespace | Derived from service name |

## Auto-Detection Logic

When a command needs service context, detect the current service automatically:

### Step 1: Get Current Git Remote

```bash
git remote get-url origin
```

Returns one of:
- HTTPS: `https://github.com/techops/keeperhub-api.git`
- SSH: `git@github.com:techops/keeperhub-api.git`

### Step 2: Parse Owner/Repo

Extract `owner/repo` from the URL:

```python
import re

def parse_github_repo(url):
    """Extract owner/repo from GitHub URL (HTTPS or SSH)."""
    # HTTPS: https://github.com/owner/repo.git
    # SSH: git@github.com:owner/repo.git
    patterns = [
        r'github\.com[:/]([^/]+)/([^/]+?)(?:\.git)?$',
    ]
    for pattern in patterns:
        match = re.search(pattern, url)
        if match:
            return f"{match.group(1)}/{match.group(2)}"
    return None
```

### Step 3: Match Against Configuration

```python
def detect_service(remote_url, config):
    """Find service matching current git remote."""
    current_repo = parse_github_repo(remote_url)
    if not current_repo:
        return None

    for service_name, service_config in config.get('services', {}).items():
        if service_config.get('github_repo') == current_repo:
            return service_name

    return None
```

### Step 4: Use or Continue Without

- **If match found:** Use that service context for commands
- **If no match:** Continue without service context (many commands still work)

## Derived Configuration

From the minimal config, derive additional values:

```python
def get_derived_config(service_name, service_config):
    """Derive optional config from minimal fields."""
    return {
        'aws_region': service_config.get('aws_region', 'us-east-1'),
        'eks_cluster': service_config.get('eks_cluster', f'keeperhub-{service_name.split("-")[-1]}'),
        'namespace': service_config.get('namespace', service_name),
    }
```

These defaults work for standard KeeperHub naming conventions. Override explicitly if needed.

## Reading Configuration

### Load Config File

```python
import yaml
from pathlib import Path

def load_services_config():
    """Load services configuration from ~/.claude/kh/services.yaml."""
    config_path = Path.home() / '.claude' / 'kh' / 'services.yaml'

    if not config_path.exists():
        return None

    with open(config_path) as f:
        return yaml.safe_load(f)
```

### Get Current Service

```python
import subprocess

def get_current_service():
    """Detect current service based on git remote."""
    config = load_services_config()
    if not config:
        return None, "No services configured. Run /kh:setup to configure."

    try:
        result = subprocess.run(
            ['git', 'remote', 'get-url', 'origin'],
            capture_output=True,
            text=True
        )
        if result.returncode != 0:
            return None, "Not in a git repository or no remote configured."

        remote_url = result.stdout.strip()
        service = detect_service(remote_url, config)

        if service:
            return service, None
        else:
            return None, f"Repository not in services config. Run /kh:setup to add it."

    except Exception as e:
        return None, f"Error detecting service: {e}"
```

## Graceful Degradation

### When No Service Detected

Commands handle missing service context gracefully:

```python
def require_service_context():
    """Check for service context, fail gracefully if missing."""
    service, error = get_current_service()
    if not service:
        print(f"[No service context] {error}")
        print("Commands that require service context won't work.")
        print("Other commands (like /kh:status --team) still work.")
        return None
    return service
```

### When Config File Missing

```python
def check_config_exists():
    """Check if services.yaml exists, suggest setup if not."""
    config_path = Path.home() / '.claude' / 'kh' / 'services.yaml'
    if not config_path.exists():
        print("No services configured yet.")
        print("Run /kh:setup to configure your KeeperHub services.")
        return False
    return True
```

### Command Behavior by Context

| Scenario | Behavior |
|----------|----------|
| Service detected | Full command functionality |
| No service match | Warn, continue with non-service commands |
| Config missing | Suggest `/kh:setup`, continue with tool-only commands |
| In non-git directory | Warn, continue with non-service commands |

## Service Context in Commands

Commands can check for service context:

```python
# Command that requires service context
service = require_service_context()
if not service:
    return  # Graceful exit with message already shown

# Command that works with or without service context
service, _ = get_current_service()
if service:
    # Show service-specific info
    pass
else:
    # Show general info
    pass
```

## Manual Override

Users can override auto-detection with `--service` flag:

```
/kh:status --service=keeperhub-api
```

This is useful when:
- Working in a directory without git
- Working on multiple services
- Testing against a different service
