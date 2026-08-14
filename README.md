# Jira Service Management Data Center Ansible Automation

Production-oriented Ansible automation for installing, configuring,
validating, and uninstalling Atlassian Jira Service Management Data
Center on RHEL 9 with PostgreSQL.

The role supports **two installation methods**:

-   **Archive installation** using the Atlassian `.tar.gz` distribution.
-   **Linux binary installer** using the Atlassian `-x64.bin` installer
    in unattended mode.

Both methods converge on the same Ansible-managed configuration, systemd
service, validation, and uninstall lifecycle.

> Current tested JSM version: **11.3.10**

## Contents

-   [Overview](#overview)
-   [Key Features](#key-features)
-   [Directory Structure](#directory-structure)
-   [Requirements](#requirements)
-   [Installation Methods](#installation-methods)
    -   [Archive Method](#archive-method)
    -   [Linux Binary Installer Method](#linux-binary-installer-method)
-   [Preparing the Installers](#preparing-the-installers)
-   [Inventory](#inventory)
-   [Configuration Variables](#configuration-variables)
-   [Usage](#usage)
    -   [Syntax Check](#syntax-check)
    -   [Precheck](#precheck)
    -   [Install JSM](#install-jsm)
    -   [Reconfigure JSM](#reconfigure-jsm)
    -   [Validate JSM](#validate-jsm)
    -   [Uninstall JSM](#uninstall-jsm)
-   [Role Task Flow](#role-task-flow)
-   [Binary Installer Response File](#binary-installer-response-file)
-   [Database Configuration Behavior](#database-configuration-behavior)
-   [Systemd Ownership Model](#systemd-ownership-model)
-   [Idempotency](#idempotency)
-   [Uninstall Behavior](#uninstall-behavior)
-   [Cluster Data Center Mode](#cluster-data-center-mode)
-   [Tags Reference](#tags-reference)
-   [Validation](#validation)
-   [Troubleshooting](#troubleshooting)
-   [Security Recommendations](#security-recommendations)
-   [Tested Lifecycle](#tested-lifecycle)

## Overview

The `jsm` role manages the complete Jira Service Management Data Center
lifecycle:

``` text
precheck
   |
   v
prerequisites
   |
   v
installation method dispatcher
   |
   +-- archive   -> install_archive.yml
   |
   +-- installer -> install_installer.yml
   |
   v
configure
   |
   v
systemd
   |
   v
validate
```

The installation method is selected with:

``` yaml
jsm_install_method: "archive"
```

or:

``` yaml
jsm_install_method: "installer"
```

The current automation contains both archive and Linux binary installer
workflows. The Linux binary installer path has been exercised
successfully with Jira Service Management 11.3.10.

The binary installer implementation supports:

-   unattended Install4j execution;
-   a generated response file;
-   custom installation and JSM Home directories;
-   custom HTTP and control ports;
-   prevention of Atlassian-managed service creation;
-   prevention of automatic JSM startup;
-   normalization of runtime ownership to the Ansible-managed `jsm`
    account;
-   handoff to the same Ansible-managed systemd service used by the
    archive installation.

## Key Features

-   RHEL 9 prechecks.
-   Java 21 support.
-   PostgreSQL connectivity validation.
-   Installer SHA256 verification.
-   Dual installation methods: `.tar.gz` and `.bin`.
-   Custom installation directory under `/app`.
-   JSM Home management.
-   Jira Service Management Data Center shared-home support.
-   Cluster node configuration.
-   JVM heap and code-cache configuration.
-   Tomcat HTTP connector configuration.
-   Ansible-managed systemd service.
-   HTTP, process, PID, and database validation.
-   Configurable startup timeout for slow first startup.
-   Common uninstall workflow for archive and binary installations.
-   Database preserved by default during uninstall.

## Directory Structure

``` text
jsm-dc-ansible/
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   └── jsm.yml
│   └── hosts.yml
├── playbooks
│   ├── install_jsm.yml
│   └── uninstall_jsm.yml
└── roles
    └── jsm
        ├── defaults
        │   └── main.yml
        ├── files
        │   ├── atlassian-servicedesk-11.3.10-x64.bin
        │   └── atlassian-servicedesk-11.3.10.tar.gz
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── tasks
        │   ├── configure.yml
        │   ├── install.yml
        │   ├── install_archive.yml
        │   ├── install_installer.yml
        │   ├── main.yml
        │   ├── precheck.yml
        │   ├── prerequisites.yml
        │   ├── systemd.yml
        │   ├── uninstall.yml
        │   └── validate.yml
        └── templates
            ├── cluster.properties.j2
            ├── dbconfig.xml.j2
            ├── jira-application.properties.j2
            ├── jira.service.j2
            ├── response.varfile.j2
            ├── server.xml.j2
            └── setenv.sh.j2
```

## Requirements

### Control node

-   Ansible 2.15 or later recommended.
-   Python 3.
-   JSM installation media stored under `roles/jsm/files/`.

### Managed node

-   RHEL 9.
-   Sufficient free space under `/app` for the installation, local home,
    logs, and temporary installer activity.
-   Network access to the PostgreSQL server.
-   Privilege escalation/root access for package, user, directory, and
    systemd management.
-   Java 21.

### Database

The PostgreSQL database and database account must already exist and be
reachable.

Tested configuration:

``` yaml
jsm_db_host: "localhost"
jsm_db_port: 15432
jsm_db_name: "jsm"
jsm_db_username: "jsm"
jsm_db_password: "jsm"
```

For production, store the database password in **Ansible Vault** rather
than plaintext inventory.

## Installation Methods

### Archive Method

Set:

``` yaml
jsm_install_method: "archive"
```

The role uses:

``` yaml
jsm_archive: "atlassian-servicedesk-{{ jsm_version }}.tar.gz"
jsm_archive_sha256: "98db3b60f37ee94abbc5cb28e2d5cb27bca2871547d7ead909e7468e53ae9509"
jsm_extract_dir: "atlassian-jira-servicedesk-{{ jsm_version }}-standalone"
```

The archive workflow is implemented in:

``` text
roles/jsm/tasks/install_archive.yml
```

The role is structured to:

1.  verify the archive and checksum;
2.  stage the distribution;
3.  extract the archive;
4.  place it under the configured final installation directory;
5.  normalize ownership;
6.  validate the Jira/JSM start and stop scripts;
7.  remove temporary staged content as implemented by the role.

### Linux Binary Installer Method

Set:

``` yaml
jsm_install_method: "installer"
```

The role uses:

``` yaml
jsm_installer: "atlassian-servicedesk-{{ jsm_version }}-x64.bin"
jsm_installer_sha256: "4811cc5bdb7b059c058950749cc3ef19202ff27a1c2dda88ace0db7ae1cd0384"
```

For JSM 11.3.10 the tested Linux installer is:

``` text
atlassian-servicedesk-11.3.10-x64.bin
```

with SHA256:

``` text
4811cc5bdb7b059c058950749cc3ef19202ff27a1c2dda88ace0db7ae1cd0384
```

The binary workflow is implemented in:

``` text
roles/jsm/tasks/install_installer.yml
```

The installer response-file format was derived from a successful
interactive JSM 11.3.10 installation and is used to automate the same
choices without allowing the Atlassian installer to own the systemd
lifecycle.

## Preparing the Installers

Place the installation media under:

``` text
roles/jsm/files/
```

For JSM 11.3.10:

``` text
roles/jsm/files/atlassian-servicedesk-11.3.10.tar.gz
roles/jsm/files/atlassian-servicedesk-11.3.10-x64.bin
```

Verify checksums:

``` bash
sha256sum roles/jsm/files/atlassian-servicedesk-11.3.10.tar.gz
sha256sum roles/jsm/files/atlassian-servicedesk-11.3.10-x64.bin
```

Tested values:

``` text
Archive:
98db3b60f37ee94abbc5cb28e2d5cb27bca2871547d7ead909e7468e53ae9509

Linux installer:
4811cc5bdb7b059c058950749cc3ef19202ff27a1c2dda88ace0db7ae1cd0384
```

The precheck should validate only the media required by the selected
`jsm_install_method`.

## Inventory

Example `inventory/hosts.yml`:

``` yaml
all:
  children:
    jsm:
      hosts:
        localhost:
          ansible_connection: local
```

For remote nodes:

``` yaml
all:
  children:
    jsm:
      hosts:
        jsm01.example.com:
          ansible_host: 10.1.20.30
          ansible_user: ansible
```

For multiple Data Center nodes, use unique node IDs per host.

## Configuration Variables

### Product and installation

  Variable                 Purpose
  ------------------------ ------------------------------------------
  `jsm_version`            Jira Service Management version
  `jsm_install_method`     `archive` or `installer`
  `jsm_archive`            `.tar.gz` filename
  `jsm_archive_sha256`     SHA256 for archive
  `jsm_installer`          Linux `.bin` filename
  `jsm_installer_sha256`   SHA256 for binary installer
  `jsm_control_port`       Tomcat control/RMI port
  `jsm_extract_dir`        Directory produced by archive extraction
  `jsm_base_dir`           Base installation filesystem
  `jsm_install_dir`        Final JSM installation directory
  `jsm_home_dir`           JSM/Jira local home
  `jsm_shared_dir`         Data Center shared home
  `jsm_temp_dir`           Installer staging directory

Example:

``` yaml
jsm_version: "11.3.10"
jsm_install_method: "installer"

jsm_archive: "atlassian-servicedesk-{{ jsm_version }}.tar.gz"
jsm_archive_sha256: "98db3b60f37ee94abbc5cb28e2d5cb27bca2871547d7ead909e7468e53ae9509"

jsm_installer: "atlassian-servicedesk-{{ jsm_version }}-x64.bin"
jsm_installer_sha256: "4811cc5bdb7b059c058950749cc3ef19202ff27a1c2dda88ace0db7ae1cd0384"

jsm_control_port: 8005

jsm_base_dir: "/app"
jsm_install_dir: "{{ jsm_base_dir }}/jsm-{{ jsm_version }}"
jsm_home_dir: "{{ jsm_base_dir }}/jsm-data"
jsm_shared_dir: "/shared/jsm"
jsm_temp_dir: "/var/tmp"
```

### Runtime user

``` yaml
jsm_user: "jsm"
jsm_group: "jsm"
jsm_user_home: "/home/jsm"
```

The Atlassian installer internally uses Jira platform scripts such as
`start-jira.sh`, `stop-jira.sh`, and `bin/user.sh`. The Ansible-managed
deployment ultimately runs under the configured `jsm` account.

### JVM

Tested Java:

``` text
OpenJDK 21.0.12 LTS
```

Example configuration:

``` yaml
java_home: "/usr/lib/jvm/java-21-openjdk-21.0.12.0.8-1.2.el9.x86_64"

jsm_jvm_min_heap: "2g"
jsm_jvm_max_heap: "4g"
jsm_jvm_code_cache: "512m"

jsm_gc: "G1GC"
```

The validated runtime used:

``` text
-Xms2g
-Xmx4g
-XX:ReservedCodeCacheSize=512m
-Djira.home=/app/jsm-data
```

### Network

``` yaml
jsm_http_port: 8009
jsm_https_port: 8444
jsm_control_port: 8005
jsm_context_path: ""
```

### Reverse proxy

``` yaml
jsm_proxy_enabled: false
jsm_proxy_name: "jsm.company.com"
jsm_proxy_port: 443
jsm_proxy_scheme: "https"
```

### Database

``` yaml
jsm_database_type: "postgres72"

jsm_db_host: "localhost"
jsm_db_port: 15432
jsm_db_name: "jsm"
jsm_db_username: "jsm"
jsm_db_password: "jsm"

jsm_jdbc_url: "jdbc:postgresql://{{ jsm_db_host }}:{{ jsm_db_port }}/{{ jsm_db_name }}"

jsm_db_driver: "org.postgresql.Driver"
jsm_db_jdbc_version: "42.7.12"
jsm_db_jdbc_jar: "postgresql-{{ jsm_db_jdbc_version }}.jar"
```

### Data Center

``` yaml
jsm_deployment_mode: "cluster"

jsm_cluster_name: "jsm-dc"
jsm_node_id: "jsm-node01"

jsm_ehcache_listener_port: 40002
jsm_ehcache_object_port: 40012
```

### Service

``` yaml
jsm_service_name: "jsm"

jsm_start_script: "{{ jsm_install_dir }}/bin/start-jira.sh"
jsm_stop_script: "{{ jsm_install_dir }}/bin/stop-jira.sh"

jsm_pid_file: "{{ jsm_install_dir }}/work/catalina.pid"
jsm_log_dir: "{{ jsm_install_dir }}/logs"

jsm_service_enabled: true
jsm_service_started: true

jsm_start_delay: 10
jsm_start_timeout: 300
jsm_stop_timeout: 300
```

### Validation

``` yaml
jsm_healthcheck_url: "http://localhost:{{ jsm_http_port }}{{ jsm_context_path }}/status"
```

### Uninstall

``` yaml
jsm_remove_install_dir: true
jsm_remove_home: true
jsm_remove_shared_home: true
jsm_remove_user: true
jsm_remove_group: true
```

The tested uninstall leaves the PostgreSQL `jsm` database intact.

## Usage

Run commands from the repository root.

### Syntax Check

``` bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --syntax-check
```

For uninstall:

``` bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_jsm.yml \
  --syntax-check
```

### Precheck

``` bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags precheck
```

Precheck should validate:

-   operating system;
-   selected installation method;
-   required installation media;
-   SHA256 checksum;
-   Java;
-   JSM user/group state;
-   filesystem capacity;
-   PostgreSQL reachability.

### Install JSM

Select the desired method in:

``` text
inventory/group_vars/jsm.yml
```

For archive:

``` yaml
jsm_install_method: "archive"
```

For binary installer:

``` yaml
jsm_install_method: "installer"
```

Then run:

``` bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml
```

The tested installation completed with JSM running as a systemd service
and listening on port `8009`.

### Reconfigure JSM

``` bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags configure
```

### Validate JSM

``` bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags validate
```

The successful tested validation reported:

``` text
========================================
JSM validation completed
Version           : 11.3.10
Install Directory : /app/jsm-11.3.10
Home Directory    : /app/jsm-data
Shared Directory  : /shared/jsm
HTTP Port         : 8009
Database          : jsm
Service           : jsm
Deployment Mode   : cluster
========================================
Next: Complete setup at http://10.148.0.2:8009
Complete the JSM Setup Wizard and apply the Data Center license.
```

### Uninstall JSM

``` bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_jsm.yml
```

The tested uninstall removed the service, installation, home, shared
home, user, and group while preserving PostgreSQL.

## Role Task Flow

`roles/jsm/tasks/main.yml` imports:

  -----------------------------------------------------------------------
  Tag                     Task file               Purpose
  ----------------------- ----------------------- -----------------------
  `always`, `precheck`    `precheck.yml`          Platform, installer,
                                                  Java, PostgreSQL and
                                                  disk checks

  `prerequisites`         `prerequisites.yml`     Packages, Java,
                                                  user/group and
                                                  directories

  `install`               `install.yml`           Select installation
                                                  method

  `configure`             `configure.yml`         JVM, database, cluster
                                                  and Tomcat
                                                  configuration

  `systemd`               `systemd.yml`           Ansible-managed JSM
                                                  systemd service

  `validate`              `validate.yml`          Full post-install
                                                  validation
  -----------------------------------------------------------------------

`install.yml` dispatches to:

``` text
jsm_install_method=archive
        |
        +--> install_archive.yml

jsm_install_method=installer
        |
        +--> install_installer.yml
```

`uninstall.yml` is called directly by `playbooks/uninstall_jsm.yml`.

## Binary Installer Response File

The template is:

``` text
roles/jsm/templates/response.varfile.j2
```

The response-file keys were confirmed from a successful interactive Jira
Service Desk 11.3.10 installation.

The generated Install4j response file contained:

``` text
# install4j response file for Jira Service Desk 11.3.10
app.install.service$Boolean=false
app.jiraHome=/var/tmp/jsm-bin-test-home
existingInstallationDir=/opt/Jira Service Desk
httpPort$Long=8080
launch.application$Boolean=false
portChoice=custom
rmiPort$Long=8005
sys.adminRights$Boolean=true
sys.adminRightsUiRootUnix$Boolean=false
sys.confirmedUpdateInstallationString=false
sys.installationDir=/var/tmp/jsm-bin-test
sys.languageId=en
```

The Ansible template substitutes the configured production values,
particularly:

``` text
app.jiraHome={{ jsm_home_dir }}
httpPort$Long={{ jsm_http_port }}
rmiPort$Long={{ jsm_control_port }}
sys.installationDir={{ jsm_install_dir }}
```

Two settings are especially important:

``` text
app.install.service$Boolean=false
launch.application$Boolean=false
```

These prevent the installer from creating its own service and from
starting JSM before Ansible configuration is complete.

The desired lifecycle is:

``` text
Atlassian JSM .bin
       |
       v
Install application files
       |
       v
Normalize ownership/runtime configuration
       |
       v
configure.yml
       |
       v
jira.service.j2
       |
       v
systemd.yml
       |
       v
Start JSM
       |
       v
validate.yml
```

## Database Configuration Behavior

Jira Service Management runs on the Jira platform and uses Jira's
database configuration mechanisms.

The automation contains:

``` text
roles/jsm/templates/dbconfig.xml.j2
```

and the tested deployment connected successfully to:

``` text
PostgreSQL host : localhost
PostgreSQL port : 15432
Database        : jsm
Database user   : jsm
```

After startup, PostgreSQL showed active JSM connections, for example:

``` text
postgres: jsm jsm 127.0.0.1(...) idle
```

The database itself was verified as:

``` text
jsm | jsm | UTF8
```

Because Jira-family applications can manage or secure runtime database
configuration after startup, production automation should avoid blindly
overwriting a runtime-modified `dbconfig.xml`. Treat database bootstrap
and deliberate database migration as controlled operations.

## Systemd Ownership Model

Regardless of installation method, JSM is managed through the
Ansible-created systemd service.

The role uses Jira platform scripts:

``` text
/app/jsm-11.3.10/bin/start-jira.sh
/app/jsm-11.3.10/bin/stop-jira.sh
```

while the service itself is:

``` text
jsm.service
```

The tested runtime state was:

``` text
Loaded: loaded (/etc/systemd/system/jsm.service; enabled)
Active: active (running)
Main PID: Java
```

and the Java process ran as:

``` text
jsm
```

This gives a consistent lifecycle:

``` text
.tar.gz -> Ansible systemd -> jsm.service
.bin    -> Ansible systemd -> jsm.service
```

rather than allowing the Atlassian installer and Ansible to manage
competing services.

## Idempotency

The role is designed for repeatable Ansible execution.

Important steady-state behaviors include:

-   detecting an existing installation;
-   avoiding unnecessary reinstall;
-   preserving correct ownership;
-   keeping the configured systemd service;
-   avoiding unnecessary restarts when templates do not change;
-   validating the existing process and PID;
-   allowing validation to be rerun independently.

The JSM validation was successfully rerun after the initial startup
issue was resolved and completed with:

``` text
ok=48
changed=0
unreachable=0
failed=0
skipped=5
```

This confirms a clean validation-only steady state.

## Uninstall Behavior

The same Ansible uninstall workflow supports the JSM deployment
regardless of installation media.

The tested uninstall task list includes:

``` text
Check if JSM service exists
Stop JSM service
Disable JSM service
Remove JSM systemd service
Reload systemd
Remove JSM installation directory
Remove JSM Home
Remove JSM Shared Home
Remove JSM user
Remove JSM group
Remove temporary installer
Verify JSM installation directory removed
Verify Jira home removed
Verify systemd service removed
Fail if uninstall incomplete
Uninstall completed
Display JSM uninstall summary
```

The successful run reported:

``` text
JSM uninstall completed
Install Dir : REMOVED
Home Dir    : REMOVED
Shared Home : REMOVED
Service     : REMOVED
OS User     : REMOVED
OS Group    : REMOVED
Database    : NOT MODIFIED
```

Post-uninstall verification confirmed:

``` text
jsm.service       -> not found
/app/jsm-11.3.10  -> removed
/app/jsm-data     -> removed
/shared/jsm       -> removed
jsm user          -> removed
jsm group         -> removed
JSM process       -> none
port 8009         -> not listening
PostgreSQL jsm DB -> preserved
```

Be careful with:

``` yaml
jsm_remove_home: true
jsm_remove_shared_home: true
```

These settings delete application data from the filesystem.

## Cluster Data Center Mode

The tested inventory uses:

``` yaml
jsm_deployment_mode: "cluster"
jsm_cluster_name: "jsm-dc"
jsm_node_id: "jsm-node01"
jsm_shared_dir: "/shared/jsm"

jsm_ehcache_listener_port: 40002
jsm_ehcache_object_port: 40012
```

Cluster mode uses the template:

``` text
roles/jsm/templates/cluster.properties.j2
```

Each Data Center node must have a unique `jsm_node_id`.

For multi-node deployments, define node-specific values in host
variables or directly on inventory hosts.

The shared directory must also be implemented as storage suitable for
all participating Data Center nodes; a local directory is appropriate
only for single-node testing or where the underlying path is actually
shared storage.

## Tags Reference

``` bash
# Precheck
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags precheck

# Prerequisites
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags prerequisites

# Installation
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags install

# Configuration
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags configure

# systemd
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags systemd

# Validation
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  --tags validate
```

For a clean host, prefer the complete playbook rather than jumping
directly to later-stage tags.

## Validation

The role validates the installed JSM environment through checks covering
the application, runtime, service, and network state.

The tested deployment confirmed:

-   `/app/jsm-11.3.10` existed and was owned by `jsm:jsm`;
-   `/app/jsm-data` existed and was owned by `jsm:jsm`;
-   `/shared/jsm` existed and was owned by `jsm:jsm`;
-   `jsm.service` was enabled and active;
-   the Java process ran as `jsm`;
-   JSM used Java 21;
-   JVM heap was `-Xms2g -Xmx4g`;
-   JSM used `/app/jsm-data` as Jira Home;
-   PostgreSQL connections were active;
-   HTTP port `8009` was reachable after startup;
-   HTTP returned `302`;
-   the response contained `X-ANODEID: jsm-node01`.

Example HTTP response:

``` text
HTTP/1.1 302
X-ANODEID: jsm-node01
Location: /secure/errors.jsp
```

A redirect during pre-license/setup state does not by itself mean the
Tomcat/JSM process failed. Validation should distinguish process/port
availability from completion of the web setup wizard.

## Troubleshooting

### `jsm_version` is undefined

If precheck fails with:

``` text
'jsm_version' is undefined
```

verify that `inventory/group_vars/jsm.yml` contains:

``` yaml
jsm_version: "11.3.10"
```

The JSM inventory previously referenced `{{ jsm_version }}` in filenames
and paths without defining the variable. Adding the version variable is
required before those expressions can resolve.

### Binary installer checksum failure

Verify:

``` bash
sha256sum roles/jsm/files/atlassian-servicedesk-11.3.10-x64.bin
```

Expected tested checksum:

``` text
4811cc5bdb7b059c058950749cc3ef19202ff27a1c2dda88ace0db7ae1cd0384
```

### Archive checksum failure

Verify:

``` bash
sha256sum roles/jsm/files/atlassian-servicedesk-11.3.10.tar.gz
```

Expected configured checksum:

``` text
98db3b60f37ee94abbc5cb28e2d5cb27bca2871547d7ead909e7468e53ae9509
```

### JSM HTTP port validation times out

The first tested startup reached:

``` text
TASK [jsm : Wait for JSM HTTP port]
fatal: Timeout when waiting for 127.0.0.1:8009
elapsed: 300
```

However, JSM subsequently became available and:

``` bash
curl -sS -I --max-time 10 http://localhost:8009/
```

returned:

``` text
HTTP/1.1 302
X-ANODEID: jsm-node01
Location: /secure/errors.jsp
```

This demonstrates that first startup can exceed the configured `300`
second wait period.

When this happens, check in another terminal:

``` bash
systemctl status jsm --no-pager -l
ps -ef | grep -E '[j]sm|[j]ira'
ss -lntp | grep ':8009'
tail -100 /app/jsm-11.3.10/logs/catalina.out
```

If the Java process remains healthy and startup logs continue
progressing, increase `jsm_start_timeout` for the environment rather
than assuming immediate application failure.

### JSM returns HTTP 302

A tested pre-setup response was:

``` text
HTTP/1.1 302
Location: /secure/errors.jsp
X-ANODEID: jsm-node01
```

The service, Java process, PostgreSQL connection, and HTTP listener were
nevertheless healthy.

Complete the JSM web setup and apply the Data Center license before
expecting the normal application UI.

### Service does not start

Check:

``` bash
systemctl status jsm.service --no-pager -l
journalctl -u jsm.service -n 100 --no-pager
```

Then:

``` bash
tail -100 /app/jsm-11.3.10/logs/catalina.out
```

### PostgreSQL unreachable

Verify:

``` bash
nc -vz localhost 15432
```

or:

``` bash
psql -h localhost -p 15432 -U jsm -d jsm
```

Also verify PostgreSQL listener and `pg_hba.conf` configuration.

### Check runtime Java

``` bash
ps -ef | grep '/app/jsm-' | grep -v grep
```

The tested process used:

``` text
/usr/lib/jvm/java-21-openjdk-21.0.12.0.8-1.2.el9.x86_64/bin/java
```

### Check installer method

``` bash
grep -E \
  'jsm_version|jsm_install_method|jsm_archive:|jsm_installer:' \
  inventory/group_vars/jsm.yml
```

Expected installer-based configuration includes:

``` text
jsm_version: "11.3.10"
jsm_install_method: "installer"
jsm_archive: "atlassian-servicedesk-{{ jsm_version }}.tar.gz"
jsm_installer: "atlassian-servicedesk-{{ jsm_version }}-x64.bin"
```

### Verbose Ansible troubleshooting

``` bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_jsm.yml \
  -vvv
```

## Security Recommendations

-   Store `jsm_db_password` in Ansible Vault.
-   Do not commit production database credentials.
-   Verify Atlassian installation-media SHA256 values before deployment.
-   Restrict access to `inventory/group_vars/` when it contains secrets.
-   Back up JSM Home, shared home, and PostgreSQL before destructive
    uninstall or upgrade operations.
-   Review `jsm_remove_home` and `jsm_remove_shared_home` before running
    uninstall.
-   Keep database deletion outside the normal uninstall workflow unless
    it is explicitly designed, separately controlled, and intentionally
    invoked.
-   Use a unique `jsm_node_id` for every Data Center node.
-   Use appropriate shared storage for multi-node Data Center
    deployments.
-   Validate firewall access to PostgreSQL and JSM ports.
-   Test installer and version changes outside production before
    rollout.

## Tested Lifecycle

The JSM 11.3.10 Linux binary-installer implementation has been exercised
through:

``` text
Installer media validation
       |
       v
Interactive installer test
       |
       +--> response.varfile captured
       +--> service creation disabled
       +--> automatic startup disabled
       |
       v
Clean Ansible deployment
       |
       v
Precheck
       |
       v
Unattended .bin installation
       |
       v
Ansible configuration
       |
       v
Ansible-managed jsm.service
       |
       v
JSM startup
       |
       +--> first HTTP wait exceeded 300 seconds
       |
       v
Application continued starting
       |
       +--> port 8009 available
       +--> HTTP 302 response
       +--> X-ANODEID: jsm-node01
       +--> PostgreSQL connections active
       |
       v
Validation rerun
       |
       +--> ok=48
       +--> changed=0
       +--> failed=0
       |
       v
Ansible uninstall
       |
       +--> service removed
       +--> /app/jsm-11.3.10 removed
       +--> /app/jsm-data removed
       +--> /shared/jsm removed
       +--> jsm user/group removed
       +--> process absent
       +--> port 8009 closed
       +--> PostgreSQL jsm database preserved
```

The archive installation remains available by changing:

``` yaml
jsm_install_method: "archive"
```

This allows the same JSM role to maintain both installation approaches
while preserving a common configuration, systemd, validation, and
uninstall model.
