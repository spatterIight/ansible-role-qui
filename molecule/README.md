<!--
SPDX-FileCopyrightText: 2018-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer, although [20.04 may be problematic](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/ansible.md#supported-ansible-versions) if you run the Ansible playbook on it)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default, but some don't (e.g. Ubuntu 18.04) and require manual installation (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

There are three testing scenarios available.

### `default`

Tests a standard qui installation, and is where the substance of the suite lives.

It first runs the same container image with none of the role's configuration, to establish what a qui that nobody configured chooses for itself — it starts happily, migrates a database and serves the same web UI, so nothing else in the scenario is allowed to rest on qui merely answering. The scenario then checks that:

- qui reports itself healthy, and Docker considers the container healthy through the health-check command the role hands it — rather than through the image's own, which probes a port qui is deliberately not listening on here;
- the container runs as the configured user, read-only and with every capability dropped;
- unauthenticated calls and forged sessions are refused (403), and forged API keys and qBittorrent proxy keys are refused (401);
- qui serves itself under the configured base URL, and refuses the root at which an unconfigured qui serves its web UI;
- a freshly installed qui can still be claimed by an unauthenticated setup call, and refuses further ones once it has been;
- the running process reports the version `qui_version` pins, matching the image tag and the image's OCI version label;
- the port, base URL, session cookie attribute, log level, update-check setting and timezone qui is running with are the ones the role rendered into its env file — all of them deliberately different from what qui picks for itself;
- an API key created over the API reads itself back, and exists as a row in the SQLite file at the host path the role bind-mounts;
- the service does not restart during a window longer than the unit's `RestartSec`.

### `qbittorrent`

Installs qBittorrent with [ansible-role-qbittorrent](https://github.com/mother-of-all-self-hosting/ansible-role-qbittorrent) alongside qui, wired together the way [the documentation](../docs/configuring-qui.md#connecting-qui-to-qbittorrent) describes, and creates the qui account on the command line before qui is ever started, as the documentation also describes. It checks that:

- qui was never claimable, and runs with the account created on the command line;
- qui connects to qBittorrent by its container name — both with a password and with an API key — and reports the very version, Web API version and libtorrent 2.x build that qBittorrent reports first-hand, while qBittorrent turns away anonymous callers;
- qui reads qBittorrent's preferences, and a category created through qui exists in qBittorrent itself;
- qui's qBittorrent proxy hands out that same qBittorrent to a client presenting nothing but a client key;
- with Local Filesystem Access, qui's orphan scan finds a file placed in qBittorrent's download directory at the very path qBittorrent sees it at (and deletes nothing), while an instance without it is refused the scan;
- neither service restarts during a window longer than the units' `RestartSec`.

### `uninstall`

Installs qui, lets it write its database, and then runs the role again with `qui_enabled: false` — the uninstallation path that the other scenarios never touch. It checks that the systemd unit, its service file, the rendered `env` and `labels` files, the container and the container network are all gone, and that the operator's data is not.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
molecule test --scenario-name qbittorrent
molecule test --scenario-name uninstall
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```
