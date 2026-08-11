# Jira Service Management (Data Center) Ansible Automation

Ansible automation to install, configure, validate, and uninstall Atlassian
Jira Service Management Data Center on RHEL 9 hosts, with optional cluster (Data
Center) support and PostgreSQL as the backing database.

## Contents

- [Overview](#overview)
- [Directory Structure](#directory-structure)
- [Requirements](#requirements)
- [Preparing the Installer](#preparing-the-installer)
- [Inventory](#inventory)
- [Configuration Variables](#configuration-variables)
- [Usage](#usage)
  - [1. Install Jira](#1-install-jira)
  - [2. Reconfigure Jira](#2-reconfigure-jira)
  - [3. Validate the Installation](#3-validate-the-installation)
  - [4. Uninstall Jira](#4-uninstall-jira)
- [Role Task Flow](#role-task-flow)
- [Tags Reference](#tags-reference)
- [Cluster (Data Center) Mode](#cluster-data-center-mode)
- [Known Issues / Things to Fix Before Production Use](#known-issues--things-to-fix-before-production-use)
- [Troubleshooting](#troubleshooting)

## Overview

A single `jsm` role drives the entire lifecycle (precheck → prerequisites →
install → configure → systemd → validate), imported with tags from
`roles/jsm/tasks/main.yml`. Two playbooks wrap the role:

- `playbooks/install_jsm.yml` — runs the full role (all tags) to install and
  bring Jira up.
- `playbooks/uninstall_jsm.yml` — imports only the `uninstall.yml` task file
  to tear Jira down.

There is no separate `configure`/`validate`-only playbook shipped yet; those
stages are reached via `--tags` against `install_jsm.yml` (see
[Usage](#usage)).

## Directory Structure

```
jsm-dc-ansible/
├── ansible.cfg
├── files
│   └── atlassian-servicedesk-11.3.10.tar.gz   # JSM installer (you provide this)
├── inventory
│   ├── group_vars
│   │   └── jira.yml
│   └── hosts.yml
├── playbooks
│   ├── install_jsm.yml
│   └── uninstall_jsm.yml
├── roles
│   └── jira
│       ├── defaults
│       │   └── main.yml
│       ├── files
│       ├── handlers
│       │   └── main.yml
│       ├── tasks
│       │   ├── configure.yml
│       │   ├── install.yml
│       │   ├── main.yml
│       │   ├── precheck.yml
│       │   ├── prerequisites.yml
│       │   ├── systemd.yml
│       │   ├── uninstall.yml
│       │   └── validate.yml
│       └── templates
│           ├── cluster.properties.j2
│           ├── dbconfig.xml.j2
│           ├── jira-application.properties.j2
│           ├── jsm.service.j2
│           ├── server.xml.j2
│           └── setenv.sh.j2
├── templates
└── vars
```

## Requirements

- **Control node**: Ansible >= 2.15, Python 3.
- **Managed nodes**: **RHEL 9 only** — `precheck.yml` hard-fails on any other
  distro/version.
- A PostgreSQL database already reachable from the JSM host (see the
  companion `postgresql-ansible` automation), with a database and user
  created for Jira.
- At least 10 GB free space on `/app` (enforced by `precheck.yml`).
- The JSM installer `.tar.gz` placed in `files/` before running (see below).

## Preparing the Installer

The role does **not** download Jira for you. Place the official Data Center
tarball in `files/` on the control node, named to match `jsm_archive` in
`inventory/group_vars/jsm.yml`, e.g.:

```
jsm-dc-ansible/files/atlassian-servicedesk-11.3.10.tar.gz
```

`install.yml` looks for it at `{{ role_path }}/files/{{ jsm_archive }}`
and fails fast if it's missing, then copies it to the configured temporary staging directory on the target,
extracts it under `/app`, and renames the extracted folder
(`jsm_extract_dir`) to `jsm_install_dir`.

## Inventory

`inventory/hosts.yml` should define a `jsm` group (the install playbook
targets `hosts: jsm`):

```yaml
all:
  children:
    jsm:
      hosts:
        jsm01.example.com:
          ansible_host: 10.1.20.20
          ansible_user: ansible
        # jsm02.example.com:            # additional node for cluster mode
        #   ansible_host: 10.1.20.21
```

## Configuration Variables

Variables referenced by the role (define/override in
`inventory/group_vars/jsm.yml` and `roles/jsm/defaults/main.yml`):

**Package & paths**

| Variable             | Used for                                             |
|-----------------------|-------------------------------------------------------|
| `jsm_version`        | Displayed during precheck; should match the archive   |
| `jsm_archive`        | Installer filename under `files/`                     |
| `jsm_extract_dir`    | Directory name produced by extracting the archive     |
| `jsm_install_dir`    | Final installation path (e.g. `/app/jsm-11.3.10`)             |
| `jsm_home_dir`       | JSM home directory                                    |
| `jsm_shared_dir`     | Shared home (cluster mode only)                        |

**User / ownership**

| Variable               | Used for                                    |
|--------------------------|----------------------------------------------|
| `jsm_user` / `jsm_group` | OS account Jira runs as, and file ownership in most tasks |
| `jsm_user_home`         | Home directory for the `jsm_user` account   |
| `jsm_owner` / `jsm_owner_group` | Ownership used specifically for `jsm_home_dir` / `jsm_shared_dir` in `prerequisites.yml` — see [Known Issues](#known-issues--things-to-fix-before-production-use) |
| `jsm_directory_mode`    | Mode for `jsm_home_dir`                     |
| `jsm_shared_dir_mode`   | Mode for `jsm_shared_dir`                   |

**JVM / Tomcat**

| Variable              | Used for                              |
|-------------------------|-----------------------------------------|
| `jsm_jvm_min_heap`     | `JVM_MINIMUM_MEMORY` in `setenv.sh`     |
| `jsm_jvm_max_heap`     | `JVM_MAXIMUM_MEMORY` in `setenv.sh`     |
| `jsm_jvm_code_cache`   | `-XX:ReservedCodeCacheSize` in `setenv.sh` |
| `jsm_http_port`        | Tomcat connector port in `server.xml`   |
| `jsm_context_path`     | Context path used by the HTTP validation check |

**Database**

| Variable            | Used for                                  |
|-----------------------|---------------------------------------------|
| `jsm_db_host`        | PostgreSQL host (precheck + validate connectivity) |
| `jsm_db_port`        | PostgreSQL port                             |
| `jsm_db_name`        | Database name (shown in validation summary) |
| `jsm_db_jdbc_jar`    | Expected JDBC driver filename under `{{ jsm_install_dir }}/lib` |

**Deployment / clustering**

| Variable                | Used for                                     |
|----------------------------|-------------------------------------------------|
| `jsm_deployment_mode`     | `"standalone"` or `"cluster"` — gates shared-home, `cluster.properties`, and node-id checks |
| `jsm_node_id`             | Written to / verified in `cluster.properties` when clustered |

**Service**

| Variable                  | Used for                                    |
|------------------------------|------------------------------------------------|
| `jsm_service_name`         | systemd unit name                              |
| `jsm_service_enabled`      | Whether the unit is enabled at boot            |
| `jsm_service_started`      | Whether the play starts the service            |
| `jsm_start_script` / `jsm_stop_script` | Paths validated in `systemd.yml`  |
| `jsm_pid_file`             | PID file waited on after start, validated later |
| `jsm_start_timeout`        | Seconds to wait for the PID file to appear     |

## Usage

Run from the project root so `ansible.cfg` and relative paths (`files/`,
`inventory/`) resolve correctly.

### 1. Install Jira

```bash
ansible-playbook playbooks/install_jsm.yml
```

This runs, in order: `precheck` → `prerequisites` → `install` → `configure`
→ `systemd` → `validate` (see [Role Task Flow](#role-task-flow)). The
playbook's `pre_tasks` also independently detect `JAVA_HOME` and fail early
if Java isn't already present — in practice the role's own
`prerequisites.yml` installs `java-21-openjdk`, so run the full playbook
rather than skipping straight to later tags on a fresh host.

To install a single node in cluster mode, set `jsm_deployment_mode: cluster`
and a unique `jsm_node_id` for that host (e.g. in `host_vars/`), then run
the same playbook against each node.

### 2. Reconfigure Jira

To re-apply JVM heap, Tomcat port, `dbconfig.xml`, `cluster.properties`, or
`jira-application.properties` changes without repeating install/prerequisite
steps:

```bash
ansible-playbook playbooks/install_jsm.yml --tags configure
```

Configuration changes trigger the `Restart JSM` handler automatically.

### 3. Validate the Installation

```bash
ansible-playbook playbooks/install_jsm.yml --tags validate
```

Checks install/home/shared-home directories, `dbconfig.xml`, JDBC driver
presence, systemd service state, the Java process, the PID file, HTTP
reachability (`200`/`302`), PostgreSQL connectivity, file ownership, the logs
directory, and (in cluster mode) the node ID in `cluster.properties`. See
[Known Issues](#known-issues--things-to-fix-before-production-use) for one
variable this stage needs defined.

### 4. Uninstall Jira

```bash
ansible-playbook playbooks/uninstall_jsm.yml
```

Stops and disables the systemd service, removes the unit file, and removes
(per the hardcoded vars in the playbook) the install directory, JSM home,
shared home (cluster mode), OS user, and OS group. The installer archive
under `files/` is preserved. Review
[Known Issues](#known-issues--things-to-fix-before-production-use) — this
playbook currently targets `hosts: all`, not `hosts: jsm`.

## Role Task Flow

`roles/jsm/tasks/main.yml` imports, in order:

| Tags                    | File                | Purpose |
|--------------------------|---------------------|---------|
| `always`, `precheck`     | `precheck.yml`      | OS check (RHEL 9), archive presence, user/group/dir existence report, Java check, PostgreSQL reachability, disk space (>=10 GB on `/app`) |
| `prerequisites`          | `prerequisites.yml` | Installs Java 21, rsync/unzip/tar/wget/fontconfig, creates `jira` user/group, `/app`, install dir, home dir, shared dir (cluster) |
| `install`                | `install.yml`       | Copies/extracts the installer, cleans up incomplete installs, sets ownership, validates start/stop scripts exist |
| `configure`               | `configure.yml`     | Creates home/shared-home dirs, edits `setenv.sh` (heap, code cache), deploys `jira-application.properties`, `dbconfig.xml`, `cluster.properties` (cluster mode), sets Tomcat HTTP port in `server.xml`, validates the deployed files |
| `systemd`                | `systemd.yml`       | Deploys the systemd unit, enables/starts the service, waits for the PID file, checks active/enabled state |
| `validate`                | `validate.yml`      | Full post-install validation (see above) |
| *(imported directly, not tagged in `main.yml`)* | `uninstall.yml` | Full teardown, called via `import_role: tasks_from: uninstall` in `uninstall_jsm.yml` |

## Tags Reference

```bash
# Only OS/prereq checks
ansible-playbook playbooks/install_jsm.yml --tags precheck

# Only install packages/user/dirs
ansible-playbook playbooks/install_jsm.yml --tags prerequisites

# Only extract/install the app
ansible-playbook playbooks/install_jsm.yml --tags install

# Only push config + restart
ansible-playbook playbooks/install_jsm.yml --tags configure

# Only (re)deploy the systemd unit / start service
ansible-playbook playbooks/install_jsm.yml --tags systemd

# Only run validation
ansible-playbook playbooks/install_jsm.yml --tags validate
```

## Cluster (Data Center) Mode

Set `jsm_deployment_mode: cluster` to enable:

- Creation of `jsm_shared_dir` (`prerequisites.yml`, `configure.yml`).
- Deployment of `cluster.properties` from `cluster.properties.j2`, keyed by
  `jsm_node_id`.
- Shared-home and cluster-properties checks in `validate.yml`.
- Shared-home removal in `uninstall.yml` when `jsm_remove_shared_home: true`.

Each cluster node needs its own unique `jsm_node_id`; set this per-host
(e.g. in `inventory/host_vars/<node>.yml`) rather than in the shared
`group_vars/jsm.yml`.

## Known Issues / Things to Fix Before Production Use

These are things spotted in the current task files worth resolving before
relying on this automation in production:

- **`validate.yml` references an undefined variable.** The final `assert`
  checks `jsm_service_file.stat.exists`, but no task in this role registers
  `jsm_service_file` (the systemd unit's `stat` result isn't captured
  anywhere). Either add a `stat` task for
  `/etc/systemd/system/{{ jsm_service_name }}.service` registered as
  `jsm_service_file` in `validate.yml`, or remove that line from the
  `assert`.
- **`uninstall_jsm.yml` targets `hosts: all`** instead of `hosts: jsm`.
  Running it as-is will attempt the Jira uninstall role against every host
  in inventory, not just Jira nodes. Change it to `hosts: jsm` (or a more
  specific group/limit) before running against a real inventory.
- **Ownership variable mismatch between `prerequisites.yml` and
  `configure.yml`.** `prerequisites.yml` creates `jsm_home_dir` and
  `jsm_shared_dir` owned by `jsm_owner`/`jsm_owner_group`, while
  `configure.yml` creates/expects them owned by `jsm_user`/`jsm_group`, and
  `validate.yml`'s ownership check also uses `jsm_user`/`jsm_group`. Make
  sure `jsm_owner`/`jsm_owner_group` are set identically to
  `jsm_user`/`jsm_group` in `group_vars/jsm.yml`, or standardize on one
  pair of variables.
- **No dedicated `configure_jira.yml` / `validate_jira.yml` playbooks** exist
  yet (unlike the PostgreSQL automation, which has one playbook per stage).
  Today you reach those stages via `--tags` on `install_jsm.yml`. Add thin
  wrapper playbooks if you want parity/consistency with the PostgreSQL repo.
- **`install_jsm.yml`'s `pre_tasks`** will fail the whole play if Java isn't
  already installed, even though `prerequisites.yml` (further into the same
  role run) would have installed it. Since the `precheck` role tag also runs
  first, this is currently redundant/conflicting — decide whether Java
  installation should be a hard pre-condition or something the role handles.


## Validated Reference Configuration

The automation has been validated with the following reference configuration:

| Component | Value |
|---|---|
| Product | Atlassian Jira Service Management Data Center |
| Version | 11.3.10 |
| Installer | `atlassian-servicedesk-11.3.10.tar.gz` |
| Extracted directory | `atlassian-jira-servicedesk-11.3.10-standalone` |
| Install directory | `/app/jsm-11.3.10` |
| Home directory | `/app/jsm-data` |
| Shared home | `/shared/jsm` |
| OS user/group | `jsm:jsm` |
| HTTP port | `8009` |
| PostgreSQL port | `15432` |
| Database / user | `jsm` / `jsm` |
| systemd service | `jsm.service` |
| Cluster node ID | `jsm-node01` |
| Bundled PostgreSQL JDBC driver | `postgresql-42.7.12.jar` |

A successful startup was observed with the cluster node becoming `ACTIVE`,
the plugin system starting, and the log message `Startup is complete. Jira is
ready to serve.` The HTTP connector listens on port `8009`.

On a fresh database, an HTTP `302` redirect to
`/secure/SetupDatabase!default.jspa` can occur while the initial application
setup is not yet complete. Treat that separately from basic Tomcat/systemd
availability.


## Troubleshooting

- **"JSM archive not found"**: confirm the file exists at
  `jsm-dc-ansible/files/{{ jsm_archive }}` and that `jsm_archive` in
  `group_vars/jsm.yml` matches the actual filename exactly.
- **"Only RHEL 9 is supported"**: `precheck.yml` asserts
  `ansible_distribution == "RedHat"` and major version `9`; this playbook
  will not run on Rocky/Alma/CentOS or other RHEL versions without editing
  `precheck.yml`.
- **PostgreSQL not reachable**: `precheck.yml` and `validate.yml` both probe
  `jsm_db_host:jsm_db_port` with `wait_for`. Confirm the database is up,
  the port is open, and firewall rules allow the JSM host to reach it.
- **HTTP validation fails (`JSM HTTP endpoint is unavailable`)**: JSM can
  take several minutes to fully start after `systemd.yml` starts the
  service; re-run with `--tags validate` after confirming
  `systemctl status {{ jsm_service_name }}` shows `active` and the logs
  under `{{ jsm_install_dir }}/logs` show a completed startup.
- **Verbose output**: add `-vvv` to any `ansible-playbook` command for
  detailed task-level debugging.
