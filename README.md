# Ansible Role: Node Exporter

| Source | Version | CI | License |
| :--- | :--- | :--- | :--- |
| [![Source Code](https://img.shields.io/badge/source-github-blue.svg)](https://github.com/grzegorzfranus/ansible-role-node-exporter) | [![Version](https://img.shields.io/github/v/release/grzegorzfranus/ansible-role-node-exporter)](https://github.com/grzegorzfranus/ansible-role-node-exporter/releases) | [![CI](https://github.com/grzegorzfranus/ansible-role-node-exporter/actions/workflows/ci.yml/badge.svg)](https://github.com/grzegorzfranus/ansible-role-node-exporter/actions/workflows/ci.yml) | [![Repository License](https://img.shields.io/badge/license-apache2.0-brightgreen.svg)](LICENSE) |

Enterprise-grade Ansible role that installs, configures, and manages Prometheus `node_exporter` as a native binary systemd service on Linux hosts (Ubuntu, Debian, RHEL/Rocky Linux).

---

## ✨ Features

- 📦 **Native Binary Deployment**: Deploys static Go binary under systemd with full host `/proc` and `/sys` visibility.
- 🔄 **Versioned Directory Symlinking**: Installs into versioned `/opt/node_exporter/node_exporter-<version>.linux-<arch>/` with atomic `/usr/local/bin/node_exporter` symlink swaps for zero-downtime upgrades.
- 🔒 **Upstream SHA256 Verification**: Verifies release archives against official `sha256sums.txt` automatically, with support for explicit checksum pinning.
- 🛡️ **Systemd Security Sandboxing**: Enforces `ProtectSystem=strict`, `ProtectHome=read-only`, `NoNewPrivileges=true`, `MemoryDenyWriteExecute=true`, empty `CapabilityBoundingSet`, and system call filtering.
- 🧪 **Dual-Layer Validation**: Declarative `meta/argument_specs.yml` paired with runtime `tasks/assert.yml` assertions.
- 📝 **Textfile Collector Integration**: Automated directory creation and systemd `ReadWritePaths` configuration for custom metrics.
- 🧹 **Automated Version Cleanup**: Retains configurable number of past release directories (`node_exporter_retain_versions: 2`) for instant offline rollbacks.
- 🧪 **Container Testing**: Full Molecule test suite covering multiple scenarios (`default`, `textfile`, `uninstall`) for CI/CD integration.

---

## 🎯 Architecture

Upstream Prometheus explicitly discourages running `node_exporter` inside containers because mounted container filesystems mask host `/proc`, `/sys`, and PID namespaces. This role deploys `node_exporter` as a single static Go binary managed directly by systemd. Host isolation and security boundaries are enforced using systemd service sandboxing directives instead of container namespaces.

```
Prometheus Server ← (Scrape /metrics HTTP GET) → node_exporter (Host Systemd Service)
                                                      ├── Host /proc
                                                      ├── Host /sys
                                                      └── Textfile Metrics Directory
```

---

## 📋 Requirements

- **Ansible**: 2.15 or higher
- **Python**: 3.9 or higher on target hosts
- **Network**: Internet access to download official Prometheus release archives from GitHub Releases
- **Privileges**: sudo/root access on target hosts

### Supported operating systems

List of officially supported operating systems for this role:

| OS Family | Version | Status |
|---|---|---|
| Ubuntu | 26.04 (Resolute) | ![✓](https://img.shields.io/badge/✓-brightgreen.svg) |
| Ubuntu | 24.04 (Noble) | ![✓](https://img.shields.io/badge/✓-brightgreen.svg) |
| Ubuntu | 22.04 (Jammy) | ![✓](https://img.shields.io/badge/✓-brightgreen.svg) |
| Debian | 13 (Trixie) | ![✓](https://img.shields.io/badge/✓-brightgreen.svg) |
| Debian | 12 (Bookworm) | ![✓](https://img.shields.io/badge/✓-brightgreen.svg) |
| Debian | 11 (Bullseye) | ![✓](https://img.shields.io/badge/✓-brightgreen.svg) |
| EL (RHEL, Rocky, Alma, Oracle) | 9 | ![✓](https://img.shields.io/badge/✓-brightgreen.svg) |

### Ansible version

Ansible core version `>= 2.15` is required.

### Python version

Python 3.9+ on managed target nodes.

### Setup module

Gathering facts (`gather_facts: true`) is required to populate `ansible_facts['architecture']`, `ansible_facts['os_family']`, and `ansible_facts['distribution']`. If you disable the Setup module in your playbook, the role will not work properly.

### Root access

This role requires root access (`become: true`) for system package installation, service user/group management, directory creation under `/opt`, and systemd unit placement.

---

## 🚀 Quick Start

### 1. Basic Setup

Add the role to your playbook requirements or role path and apply default configuration:

```yaml
---
- name: Deploy Node Exporter
  hosts: all
  become: true
  roles:
    - role: grzegorzfranus.node_exporter
```

### 2. Custom Collectors and Textfile Directory

Configure custom enabled collectors and textfile directory for custom metrics:

```yaml
---
- name: Deploy Node Exporter with Textfile Collector
  hosts: monitoring_targets
  become: true
  roles:
    - role: grzegorzfranus.node_exporter
      vars:
        node_exporter_enabled_collectors:
          - "systemd"
          - "processes"
        node_exporter_textfile_directory: "/var/lib/node_exporter/textfile_collector"
        node_exporter_read_write_paths:
          - "/var/lib/node_exporter/textfile_collector"
```

### 3. Run the playbook

```bash
ansible-playbook -i inventory/hosts site.yml --tags node_exporter
```

---

## ⚙️ Configuration

### Default Configuration

The role comes with production-ready defaults:

```yaml
node_exporter_version: "1.9.1"
node_exporter_port: 9100
node_exporter_listen_address: ""
node_exporter_telemetry_path: "/metrics"
node_exporter_service_enabled: true
node_exporter_retain_versions: 2
```

### Advanced Configuration

To restrict listening to a specific internal management interface (e.g., `192.0.2.10` per RFC 5737):

```yaml
---
- name: Advanced Node Exporter Setup
  hosts: all
  become: true
  vars:
    node_exporter_listen_address: "192.0.2.10"
    node_exporter_port: 9100
    node_exporter_disabled_collectors:
      - "bcache"
      - "infiniband"
  roles:
    - role: grzegorzfranus.node_exporter
```

---

## 📊 Variables

### General Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `node_exporter_role_action` | `str` | `"all"` | Action phase: `all`, `prerequisites`, `install`, `configure`, `service`, `remove`, `test`. |
| `node_exporter_state` | `str` | `"present"` | State of service and assets: `present` or `absent`. |
| `node_exporter_service_enabled` | `bool` | `true` | Whether systemd service is enabled and started. |
| `node_exporter_run_test` | `bool` | `false` | Whether to run post-installation health check. |

### Installation Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `node_exporter_version` | `str` | `"1.9.1"` | Release version string to install. |
| `node_exporter_base_dir` | `path` | `"/opt/node_exporter"` | Base directory for versioned release subdirectories. |
| `node_exporter_bin_link` | `path` | `"/usr/local/bin/node_exporter"` | Symlink path to active binary. |
| `node_exporter_retain_versions` | `int` | `2` | Number of previous releases retained for rollback. |
| `node_exporter_download_dir` | `path` | `"/tmp"` | Temporary directory for downloads. |
| `node_exporter_download_url` | `str` | `""` | Optional custom download URL override. |
| `node_exporter_checksum` | `str` | `""` | Optional explicit SHA256 checksum string. |
| `node_exporter_verify_checksum` | `bool` | `true` | Enable SHA256 verification against `sha256sums.txt`. |
| `node_exporter_download_timeout` | `int` | `60` | HTTP download timeout in seconds. |

### Service Account Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `node_exporter_user` | `str` | `"node_exporter"` | Unprivileged system service user. |
| `node_exporter_group` | `str` | `"node_exporter"` | Unprivileged system service group. |

### Network Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `node_exporter_listen_address` | `str` | `""` | Bind IP address (`""` binds to all interfaces). |
| `node_exporter_port` | `int` | `9100` | TCP listener port (1-65535). |
| `node_exporter_telemetry_path` | `str` | `"/metrics"` | HTTP metrics path. |

### Collector Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `node_exporter_enabled_collectors` | `list[str]` | `[]` | Explicitly enabled collectors (`--collector.<name>`). |
| `node_exporter_disabled_collectors` | `list[str]` | `[]` | Explicitly disabled collectors (`--no-collector.<name>`). |
| `node_exporter_textfile_directory` | `str` | `""` | Directory for textfile collector metrics. |
| `node_exporter_extra_args` | `list[str]` | `[]` | Additional flags for binary. |

### Systemd Hardening & Ordering Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `node_exporter_after_units` | `list[str]` | `["network-online.target"]` | Units for `After=` ordering. |
| `node_exporter_wants_units` | `list[str]` | `["network-online.target"]` | Units for `Wants=` dependency. |
| `node_exporter_restart_policy` | `str` | `"on-failure"` | Systemd service restart policy. |
| `node_exporter_restart_sec` | `int` | `5` | Restart delay in seconds. |
| `node_exporter_limit_nofile` | `int` | `65536` | LimitNOFILE file descriptor ceiling. |
| `node_exporter_read_write_paths` | `list[str]` | `[]` | Paths granted write access under `ProtectSystem=strict`. |

### Grafana Integration Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `node_exporter_dashboard_urls` | `list[str]` | `[]` | List of Grafana dashboard download URLs. Recommended: Dashboard ID `1860` ("Node Exporter Full"). |

---

## 📌 Role Properties

| Property | Value | Description |
| :--- | :--- | :--- |
| **Idempotent** | ✅ Yes | Running the role multiple times with the same parameters produces the exact same state. |
| **Atomic** | ❌ No | The role can be partially applied. Binary extraction and systemd unit rendering occur sequentially. |
| **Check Mode** | ✅ Supported | Dry-run mode is fully supported. Health checks and mutating commands are safely skipped. |
| **Diff Mode** | ✅ Supported | Template rendering tasks support Ansible diff mode for visual change preview. |

---

## 📤 Role Output

This role configures the following system components:

- Systemd service unit `/etc/systemd/system/node_exporter.service`.
- Versioned release directory `/opt/node_exporter/node_exporter-<version>.linux-<arch>/`.
- Active binary symlink `/usr/local/bin/node_exporter`.
- Running HTTP metric endpoint on TCP port 9100 exposing host telemetry.

---

## 🔍 Verification

After deployment, verify metrics collection is working:

### Check Service Status

```bash
sudo systemctl status node_exporter.service
```

### Verify Metrics Endpoint

```bash
curl -s http://127.0.0.1:9100/metrics | grep node_cpu_seconds_total
```

### Check Systemd Logs

```bash
sudo journalctl -u node_exporter.service -f
```

---

## 🔄 Upgrade Procedure

### Upgrade Steps

1. Update `node_exporter_version` in your inventory or group variables (e.g. `node_exporter_version: "1.9.2"`).
2. Execute the playbook:
   ```bash
   ansible-playbook -i inventory/hosts site.yml --tags node_exporter
   ```

### What Happens During Upgrade

| Step | Action | Impact on scraping |
| :--- | :--- | :--- |
| 1. Download & Verify | New binary downloaded into `/opt/node_exporter/node_exporter-1.9.2.linux-amd64/` | Zero impact. |
| 2. Symlink Swap | `/usr/local/bin/node_exporter` updated to point to 1.9.2 | Zero impact. |
| 3. Service Restart | Systemd restarts `node_exporter` | Single scrape interval miss (< 1s). |

### Upgrade Safety Guarantees

The previous release directory remains intact on disk. If network connectivity fails during download, the upgrade halts in the rescue block without touching the active symlink or running service.

### Old Version Cleanup

The task `cleanup.yml` automatically retains `node_exporter_retain_versions` (default `2`) release directories and never removes the directory currently targeted by the active symlink.

### Manual Rollback

If a new version exhibits issues, repoint the symlink and restart systemd manually:

```bash
sudo ln -sfn /opt/node_exporter/node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/node_exporter
sudo systemctl restart node_exporter
```

---

## 🛡️ Security Features

- ✅ **Secure Default Configuration**: Minimal attack surface with unprivileged system user `node_exporter`
- ✅ **Binary Sanitization**: Ownership fixed strictly to `root:root 0755` so service user cannot modify binary
- ✅ **Service Sandboxing**: Enforces `ProtectSystem=strict`, `ProtectHome=read-only`, `PrivateTmp=true`, `PrivateDevices=true`, `NoNewPrivileges=true`, `MemoryDenyWriteExecute=true`, empty `CapabilityBoundingSet=`
- ✅ **Checksum Verification**: Validates release archives against official `sha256sums.txt` before extraction

### Enhanced Security Configuration

```yaml
# Strict systemd sandboxing with dedicated textfile write directory
node_exporter_textfile_directory: "/var/lib/node_exporter/textfile_collector"
node_exporter_read_write_paths:
  - "/var/lib/node_exporter/textfile_collector"
```

### Uninstall

Set `node_exporter_state: "absent"` and run the playbook to purge systemd units, symlinks, binary directories, service user accounts, and groups:

```yaml
---
- name: Uninstall Node Exporter
  hosts: all
  become: true
  roles:
    - role: grzegorzfranus.node_exporter
      vars:
        node_exporter_state: "absent"
```

### Roll-back Capabilities

Because releases are stored in versioned subdirectories, rolling back requires no internet connection or archive re-download.

---

## 🔒 Security considerations

- Binary ownership is strictly `root:root` with permissions `0755`.
- Service runs under dedicated system user `node_exporter` with `/sbin/nologin` shell and `create_home: false`.
- `ProtectHome=read-only` is deliberately chosen over `ProtectHome=true` so host filesystem collectors can gather metrics without failing.

---

## 🧪 Check mode behavior

- Most validation and status checks run normally in Check Mode.
- Mutating commands (such as package installation and service management) are safely skipped.
- Health check assertions (`test.yml`) are skipped in check mode.

---

## 🏷️ Tags usage

Use `--tags` to run selective parts of the role:

```bash
ansible-playbook -i inventory/hosts site.yml --tags node_exporter_install
```

---

## 🌐 Network resilience

Archive downloads use `ansible.builtin.get_url` configured with `retries: 3`, `delay: 5`, and configurable `node_exporter_download_timeout`.

---

## 🧰 Repository management

> [!IMPORTANT]
> Dependabot watches `.github/workflows/**` actions only. It does **not** track `node_exporter_version` in `defaults/main.yml`. Exporter version updates must be bumped manually in `defaults/main.yml` when new upstream releases occur.

---

## 🔧 Troubleshooting

### Service Issues

Inspect systemd status and journal logs:
```bash
sudo systemctl status node_exporter.service
sudo journalctl -u node_exporter.service --no-pager -n 50
```

### Checksum Issues

If upstream checksum verification fails, verify node_exporter version tag and SHA256 checksum in upstream `sha256sums.txt`.

### Missing Metrics

If custom textfile collector metrics do not surface in `/metrics`:
1. Verify `node_exporter_textfile_directory` is configured.
2. Confirm the path is included in `node_exporter_read_write_paths`.
3. Check file extension ends with `.prom`.

---

## 📁 File Structure

```text
ansible-role-node-exporter/
├── .ansible-lint                     # Ansible lint configuration
├── .gitignore                        # Git ignore patterns
├── .release-please-manifest.json     # Release Please manifest tracking
├── .yamllint                         # YAML lint configuration
├── CHANGELOG.md                      # Change history
├── LICENSE                           # Apache-2.0 license file
├── README.md                         # Role documentation
├── release-please-config.json        # Release Please release configuration
├── .github/
│   ├── ISSUE_TEMPLATE/                # Issue templates for bug, feature, task
│   │   ├── bug_report.yml
│   │   ├── config.yml
│   │   ├── feature_request.yml
│   │   └── task.yml
│   ├── PULL_REQUEST_TEMPLATE/         # Pull request description template
│   │   └── pull_request_template.md
│   ├── dependabot.yml                 # Dependabot configuration for GitHub Actions
│   └── workflows/
│       ├── ci.yml                     # CI pipeline workflow
│       └── release.yml                # Release Please + Galaxy publish workflow
├── defaults/
│   └── main.yml                       # Default configuration variables
├── handlers/
│   └── main.yml                       # Service reload and restart handlers
├── meta/
│   ├── argument_specs.yml             # Native argument specification schema
│   └── main.yml                       # Role metadata and Galaxy information
├── molecule/                          # Molecule testing scenarios
│   ├── default/                       # Default scenario (installation & metrics test)
│   ├── textfile/                      # Textfile collector scenario
│   └── uninstall/                     # Uninstallation scenario
├── tasks/
│   ├── main.yml                       # Main task orchestrator
│   ├── assert.yml                     # Variable assertion & validation tasks
│   ├── cleanup.yml                    # Old version release cleanup tasks
│   ├── configure.yml                  # Flag assembly & systemd hardening tasks
│   ├── install.yml                    # Binary download, extraction & symlinking
│   ├── prerequisites.yml              # Package prerequisites & directory setup
│   ├── remove.yml                     # Role teardown & asset removal tasks
│   ├── service.yml                    # Systemd unit rendering & service state
│   ├── test.yml                       # Metric endpoint health check tasks
│   └── user.yml                       # System user and group setup
├── templates/
│   └── systemd/
│       └── node_exporter.service.j2   # Systemd service unit Jinja2 template
└── vars/
    ├── debian.yml                     # Debian family package variables
    ├── default.yml                    # Default fallback package variables
    ├── main.yml                       # Architecture mapping & internal facts
    └── redhat.yml                     # RedHat family package variables
```

---

## 🏷️ Tags

Summary of all available Ansible tags:

| Tag | Description |
|---|---|
| `node_exporter_setup` | Setup tasks including OS-specific variables, user, group, and directory creation |
| `node_exporter_install` | Binary download, extraction, symlinking, and version cleanup |
| `node_exporter_configure` | Command flag assembly and systemd unit rendering |
| `node_exporter_service` | Systemd service unit deployment and management |
| `node_exporter_remove` | Purge all role resources (`state: absent`) |
| `node_exporter_test` | Metric endpoint verification health check |

---

## CI/CD Pipeline

This repository uses centralized, reusable GitHub Actions workflows from [grzegorzfranus/github-workflows](https://github.com/grzegorzfranus/github-workflows) (`v3.0.1`) for quality assurance, security scanning, and release automation.

### CI Pipeline (`ansible-ci.yml@v3.0.1`)

Runs on every Pull Request in a two-tier gate pattern:

1. **Branch Name Lint** — enforces naming conventions (`feature/`, `bugfix/`, `fix/`, `hotfix/`, `release/`, `chore/`, `docs/`, `refactor/`, `test/`, `build/`, `ci/`, `perf/`, `revert/`)
2. **PR Title Lint** — enforces [Conventional Commits](https://www.conventionalcommits.org/) format (`feat:`, `fix:`, `ci:`, etc.)
3. **YAML Syntax Lint** — validates YAML formatting via `yamllint`
4. **Ansible Lint** — checks Ansible best practices and role standards
5. **Galaxy Metadata Validation** — verifies `meta/main.yml` schema and requirements (`ansible-meta-validate.yml`)
6. **Security Scanning** — TruffleHog secret detection and Trivy IaC scanning (`ansible-security.yml`)
7. **Molecule Integration Tests** — executes Molecule test matrix across Ubuntu 26.04, Ubuntu 24.04, Ubuntu 22.04, Debian 13, Debian 12, Debian 11, and Rocky Linux 9 (`ansible-molecule.yml`)
8. **Merge Check Gate** — single authoritative status check aggregating all results for branch protection

### Release & Publish Pipeline (`ansible-publish.yml@v3.0.1`)

Automated via [Release Please](https://github.com/googleapis/release-please):

1. **Push to `main`** → Release Please creates or updates a Release PR with automated changelog generation
2. **Release PR Validation** → validates YAML syntax and actions schema before setting `Merge Check` status
3. **Merge Release PR** → creates Git version tag and GitHub Release automatically
4. **Ansible Galaxy Publish** → publishes tagged release to Ansible Galaxy via `ansible-publish.yml@v3.0.1` with exponential backoff retry logic

---

## Example Playbooks

```yaml
---
- name: Deploy Prometheus Node Exporter
  hosts: all
  become: true
  roles:
    - role: grzegorzfranus.node_exporter
      vars:
        node_exporter_version: "1.9.1"
        node_exporter_port: 9100
        node_exporter_enabled_collectors:
          - "systemd"
          - "processes"
        node_exporter_service_enabled: true
```

---

## 🤝 Contributing

Contributions, bug reports, and feature requests are welcome!

- Fork the repository and create your branch from `main`
- Branch naming must match: `^(feature|bugfix|fix|hotfix|release|chore|docs|refactor|test|build|ci|perf|revert)/[a-zA-Z0-9-]+$`
- PR titles must follow [Conventional Commits](https://www.conventionalcommits.org/): `^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-zA-Z0-9_-]+\))?!?: .+$`
- Centralized workflows from [github-workflows](https://github.com/grzegorzfranus/github-workflows) version `v3.0.1` are used to run CI/CD pipelines
- Ensure your code passes all CI checks (`yamllint`, `ansible-lint`, Molecule tests)
- Submit a pull request describing your changes (a template is available under `.github/PULL_REQUEST_TEMPLATE/pull_request_template.md`)

---

## 📝 License

This project is licensed under the Apache-2.0 License - see the [LICENSE](LICENSE) file for details.

---

## 👥 Author Information

This role was created and is maintained by [Grzegorz Franus](https://github.com/grzegorzfranus).
