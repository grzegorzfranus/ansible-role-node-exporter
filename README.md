# Ansible Role: Node Exporter

| Source | Version | CI | License |
| :--- | :--- | :--- | :--- |
| [![GitHub](https://img.shields.io/badge/github-grzegorzfranus/ansible--role--node--exporter-blue.svg?logo=github)](https://github.com/grzegorzfranus/ansible-role-node-exporter) | [![Release](https://img.shields.io/github/v/release/grzegorzfranus/ansible-role-node-exporter?color=blue&logo=github)](https://github.com/grzegorzfranus/ansible-role-node-exporter/releases) | [![CI](https://github.com/grzegorzfranus/ansible-role-node-exporter/actions/workflows/ci.yml/badge.svg)](https://github.com/grzegorzfranus/ansible-role-node-exporter/actions/workflows/ci.yml) | [![License](https://img.shields.io/badge/license-Apache--2.0-green.svg)](LICENSE) |

Enterprise-grade Ansible role that installs, configures, and manages Prometheus `node_exporter` as a native binary systemd service on Linux hosts (Ubuntu, Debian, RHEL/Rocky Linux).

---

## ✨ Features

- **Native Binary Deployment**: Deploys static Go binary under systemd with full host `/proc` and `/sys` visibility.
- **Versioned Directory Symlinking**: Installs into versioned `/opt/node_exporter/node_exporter-<version>.linux-<arch>/` with atomic `/usr/local/bin/node_exporter` symlink swaps for zero-downtime upgrades.
- **Upstream SHA256 Verification**: Verifies release archives against official `sha256sums.txt` automatically, with support for explicit checksum pinning.
- **Systemd Security Sandboxing**: Enforces `ProtectSystem=strict`, `ProtectHome=read-only`, `NoNewPrivileges=true`, empty `CapabilityBoundingSet`, and system call filtering.
- **Dual-Layer Validation**: Declarative `meta/argument_specs.yml` paired with runtime `tasks/assert.yml` assertions.
- **Textfile Collector Integration**: Automated directory creation and systemd `ReadWritePaths` configuration for custom metrics.
- **Automated Version Cleanup**: Retains configurable number of past release directories (`node_exporter_retain_versions: 2`) for instant offline rollbacks.

---

## 🎯 Architecture

Upstream Prometheus explicitly discourages running `node_exporter` inside containers because mounted container filesystems mask host `/proc`, `/sys`, and PID namespaces. This role deploys `node_exporter` as a single static Go binary managed directly by systemd. Host isolation and security boundaries are enforced using systemd service sandboxing directives instead of container namespaces.

---

## 📋 Requirements

### Supported operating systems

- **Ubuntu**: 22.04 LTS (Jammy), 24.04 LTS (Noble), 26.04 (Resolute)
- **Debian**: 11 (Bullseye), 12 (Bookworm), 13 (Trixie)
- **Enterprise Linux / RHEL / Rocky Linux**: 9.x

### Ansible version

Ansible core version `>= 2.15` is required.

### Python version

Python 3.9+ on managed target nodes.

### Setup module

Gathering facts (`gather_facts: true`) is required to populate `ansible_facts['architecture']`, `ansible_facts['os_family']`, and `ansible_facts['distribution']`.

### Root access

Root or `become: true` privileges are required for system package installation, service user/group management, directory creation in `/opt`, and systemd unit placement.

---

## 🚀 Quick Start

### 1. Basic Setup

Add the role to your playbook requirements or role path and apply default configuration:

```yaml
- hosts: all
  become: true
  roles:
    - role: grzegorzfranus.node_exporter
```

### 2. Custom Collectors and Textfile Directory

Configure custom enabled collectors and textfile directory for custom metrics:

```yaml
- hosts: monitoring_targets
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

Default configuration binds `node_exporter` to port 9100 on all network interfaces (`:9100`) exposing default collectors:

```yaml
node_exporter_version: "1.9.1"
node_exporter_port: 9100
node_exporter_listen_address: ""
node_exporter_telemetry_path: "/metrics"
```

### Advanced Configuration

To restrict listening to a specific internal management interface (e.g., `192.0.2.10` per RFC 5737):

```yaml
node_exporter_listen_address: "192.0.2.10"
node_exporter_port: 9100
node_exporter_disabled_collectors:
  - "bcache"
  - "infiniband"
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
| `node_exporter_dashboard_urls` | `list[str]` | `[]` | List of Grafana dashboard download URLs. Recommended: Dashboard ID `1860` ("Node Exporter Full"). Pin a specific revision URL verified on grafana.com. |

---

## 📌 Role Properties

- **Idempotency**: All tasks use native stateful Ansible modules or idempotency guards (`stat`, `creates`).
- **Sanitization**: Binary ownership is fixed to `root:root 0755` so the service user cannot modify its own executable.
- **Service Ordering**: `After=network-online.target` prevents listener binding failures during boot.

---

## 📤 Role Output

- Systemd service unit `/etc/systemd/system/node_exporter.service`.
- Binary release directory `/opt/node_exporter/node_exporter-<version>.linux-<arch>/`.
- Active binary symlink `/usr/local/bin/node_exporter`.
- Running HTTP metric endpoint on port 9100.

---

## 🔍 Verification

### Check Service Status

```bash
systemctl status node_exporter
```

### Verify Metrics Endpoint

```bash
curl -s http://127.0.0.1:9100/metrics | grep node_cpu_seconds_total
```

### Check Logs

```bash
journalctl -u node_exporter -f
```

---

## 🔄 Upgrade Procedure

### Upgrade Steps

1. Update `node_exporter_version` in your host or group variables (e.g. `node_exporter_version: "1.9.2"`).
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
ln -sfn /opt/node_exporter/node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/node_exporter
systemctl restart node_exporter
```

---

## 🛡️ Security Features

### Enhanced Security Configuration

Systemd service unit contains enterprise sandboxing controls:
- `ProtectSystem=strict`
- `ProtectHome=read-only` (**Note**: `ProtectHome=read-only` is a deliberate design requirement. `ProtectHome=true` breaks host filesystem collector metrics).
- `PrivateTmp=true`
- `PrivateDevices=true`
- `NoNewPrivileges=true`
- `ProtectKernelTunables=true`
- `ProtectKernelModules=true`
- `ProtectControlGroups=true`
- `CapabilityBoundingSet=` (empty)

### Uninstall

Set `node_exporter_state: "absent"` and run the playbook to purge systemd units, symlinks, binary directories, user accounts, and groups:

```yaml
vars:
  node_exporter_state: "absent"
```

### Roll-back Capabilities

Because releases are stored in versioned subdirectories, rolling back requires no internet connection or archive re-download.

---

## 🔒 Security considerations

- Binary ownership is strictly `root:root` with permissions `0755`.
- Service runs under dedicated system user `node_exporter` with `/sbin/nologin` shell and `create_home: false`.

---

## 🧪 Check mode behavior

Running Ansible with `--check` simulates task execution. Health check assertions (`test.yml`) are skipped in check mode.

---

## 🏷️ Tags usage

- `node_exporter_setup` - System user, group, and base directory setup.
- `node_exporter_install` - Binary download, extraction, symlink, and version cleanup.
- `node_exporter_configure` - Flag assembly and systemd unit rendering.
- `node_exporter_service` - Systemd unit deployment and service management.
- `node_exporter_remove` - Purge role resources (`state: absent`).
- `node_exporter_test` - Metrics endpoint verification test.

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
systemctl status node_exporter.service
journalctl -u node_exporter.service --no-pager -n 50
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
├── .ansible-lint
├── .gitignore
├── .release-please-manifest.json
├── .yamllint
├── CHANGELOG.md
├── LICENSE
├── README.md
├── release-please-config.json
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── PULL_REQUEST_TEMPLATE/
│   ├── dependabot.yml
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── meta/
│   ├── argument_specs.yml
│   └── main.yml
├── molecule/
│   ├── default/
│   ├── textfile/
│   └── uninstall/
├── tasks/
│   ├── assert.yml
│   ├── cleanup.yml
│   ├── configure.yml
│   ├── install.yml
│   ├── main.yml
│   ├── prerequisites.yml
│   ├── remove.yml
│   ├── service.yml
│   ├── test.yml
│   └── user.yml
├── templates/
│   └── systemd/
│       └── node_exporter.service.j2
└── vars/
    ├── debian.yml
    ├── main.yml
    └── redhat.yml
```

---

## 🏷️ Tags

Summary of all available Ansible tags: `node_exporter_setup`, `node_exporter_install`, `node_exporter_configure`, `node_exporter_service`, `node_exporter_remove`, `node_exporter_test`.

---

## CI/CD Pipeline

### CI Pipeline (`ansible-ci.yml@v3.0.1`)

Runs on pull requests to validate branch name (`branch-name-lint`), PR title (`pr-title-lint`), `yamllint`, `ansible-lint`, `actionlint`, and Molecule matrix testing across 7 OS distributions.

### Release & Publish Pipeline (`ansible-publish.yml@v3.0.1`)

Automates version bumping, changelog generation, GitHub release creation via release-please, and publication to Ansible Galaxy upon merging to `main`.

---

## Example Playbooks

```yaml
- hosts: all
  become: true
  roles:
    - role: grzegorzfranus.node_exporter
      vars:
        node_exporter_version: "1.9.1"
        node_exporter_port: 9100
        node_exporter_enabled_collectors:
          - "systemd"
          - "processes"
```

---

## 🤝 Contributing

Contributions are welcome!

- Branch naming must match: `^(feature|bugfix|fix|hotfix|release|chore|docs|refactor|test|build|ci|perf|revert)/[a-zA-Z0-9-]+$`
- PR titles must follow Conventional Commits: `^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-zA-Z0-9_-]+\))?!?: .+$`

---

## 📝 License

This project is licensed under the Apache-2.0 License - see the [LICENSE](LICENSE) file for details.

---

## 👥 Author Information

Created and maintained by Grzegorz Franus.
