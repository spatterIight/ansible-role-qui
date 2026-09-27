<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up qui

This is an [Ansible](https://www.ansible.com/) role which installs [qui](https://getqui.com/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

qui is a fast, modern web interface for [qBittorrent](https://www.qbittorrent.org/), which can manage multiple qBittorrent instances from a single place. On top of managing torrents, it offers cross-seeding, rule-based automations, backups, orphan file scanning, and a reverse proxy which lets other applications (Sonarr, Radarr, autobrr, etc.) use qBittorrent without knowing its credentials.

See the project's [documentation](https://getqui.com/docs/intro/) to learn what qui does and why it might be useful to you.

## Prerequisites

qui does not include a BitTorrent client of its own. It needs at least one qBittorrent instance (version 4.3.9 or newer) to manage, which it reaches over qBittorrent's Web API.

If you are looking for an Ansible role for qBittorrent, you can check out [ansible-role-qbittorrent](https://github.com/mother-of-all-self-hosting/ansible-role-qbittorrent). See [below](#connecting-qui-to-qbittorrent) for how to connect the two.

## Adjusting the playbook configuration

To enable qui with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# qui                                                                  #
#                                                                      #
########################################################################

qui_enabled: true

########################################################################
#                                                                      #
# /qui                                                                 #
#                                                                      #
########################################################################
```

### Set the hostname

To enable qui you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
qui_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

>[!NOTE]
> The `qui_path_prefix` variable can be adjusted to host under a subpath (e.g. `qui_path_prefix: /qui`). qui itself is then configured to serve everything (the web UI, its API and its qBittorrent proxy) under that path.

### Connecting qui to qBittorrent

If qBittorrent is installed on the same server with [ansible-role-qbittorrent](https://github.com/mother-of-all-self-hosting/ansible-role-qbittorrent), you can let qui talk to it directly over the container network, instead of going through qBittorrent's public hostname.

To do so, add the following configuration to your `vars.yml` file:

```yaml
# Lets qui reach qBittorrent by its container name (see "Adding qBittorrent to qui" below)
qui_container_additional_networks_custom:
  - "{{ qbittorrent_container_network }}"

# Starts qui after qBittorrent
qui_systemd_wanted_services_list_custom:
  - "{{ qbittorrent_identifier }}.service"
```

#### Enabling Local Filesystem Access (optional)

Some features of qui (orphan file scanning, hardlink/reflink based cross-seeding, content file downloads, MediaInfo, some automation conditions, etc.) read qBittorrent's files directly, rather than through qBittorrent's Web API. For these to work, qui needs to see qBittorrent's download directory at exactly the same path as qBittorrent sees it. See [this page](https://getqui.com/docs/features/instance-settings/#local-filesystem-access) on the project's documentation for details.

To mount the download directory of the qBittorrent instance into the qui container at that path, add the following configuration to your `vars.yml` file:

```yaml
qui_container_additional_volumes_custom:
  - type: bind
    src: "{{ qbittorrent_download_path }}"
    dst: "{{ qbittorrent_download_bind_path }}"
```

qui runs as the user specified with `qui_uid` / `qui_gid`, and it needs to be able to write to that directory for features which create or delete files (e.g. hardlink mode, orphan file deletion). If you use the MASH playbook, both qui and qBittorrent run as the same user by default, so no adjustment is needed.

If you do not need write access, you can add `options: readonly` to the entry above.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `qui_environment_variables_additional_variables` variable

See [this page](https://getqui.com/docs/configuration/environment/) for a complete list of qui's config options that you could put in `qui_environment_variables_additional_variables`.

>[!NOTE]
> By default qui's session cookie is marked as `Secure` when qui is served over HTTPS via Traefik, so that it is never sent over plain HTTP. If you also access qui over plain HTTP (e.g. via `qui_container_http_host_bind_port`), you need to set `qui_environment_variables_qui_session_cookie_secure: false` to be able to log in there.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, qui becomes available at the specified hostname like `https://example.com`.

To get started, open the URL with a web browser, and create the account.

### Create the first account promptly

qui has no default credentials. Instead, a freshly installed instance accepts an unauthenticated setup call from anybody who can reach it, and whoever makes that call first creates the only account and takes control of it (and of every qBittorrent instance added to it later).

Because this role publishes qui on a public hostname, the window between installing it and creating your account is a window in which a stranger can claim the instance. To prevent it, it is recommended to create your account immediately after installing.

If you would rather not race anybody, you can create the account on the server before the service is ever reachable. To do so, adjust the command below as your playbook sets them, and run it on the server after running `ansible-playbook -i inventory/hosts setup.yml --tags=setup-qui` but before starting the service:

```sh
docker run --rm -it \
  --user=UID_HERE:GID_HERE \
  --mount type=bind,src=QUI_DATA_PATH_HERE,dst=/config \
  ghcr.io/autobrr/qui:QUI_VERSION_HERE \
  create-user --username YOUR_USERNAME_HERE
```

The program prompts for the password, which must be at least 8 characters long.

### Adding qBittorrent to qui

To manage a qBittorrent instance with qui, open `Settings > Instances` on qui and click `Add Instance`.

If you have connected qui to qBittorrent [as described above](#connecting-qui-to-qbittorrent), set the `URL` field to `http://QBITTORRENT_IDENTIFIER_HERE:8080`, replacing `QBITTORRENT_IDENTIFIER_HERE` with the name of qBittorrent's container (the value of `qbittorrent_identifier`, e.g. `mash-qbittorrent` if you use the MASH playbook), and `8080` with the value of `qbittorrent_container_http_port` if you have changed it. Otherwise, set it to the public URL of your qBittorrent instance.

Under `qBittorrent Authentication`, select either of these:

- **API Key** (recommended): qBittorrent 5.2.0 and newer can authenticate API clients by a key, which keeps working even if you change the password of the Web UI. You can generate the key on qBittorrent under `Tools > Options > WebUI` in the `Authentication` section.
- **Username and Password**: the credentials of qBittorrent's Web UI.

>[!WARNING]
> qBittorrent generates a *temporary* password on every start until a permanent one is set, so make sure to set a password (or generate the API key) on qBittorrent before adding it to qui. Otherwise qui will lose the connection as soon as qBittorrent restarts.
>
> Please also do not bypass qBittorrent's authentication with its "Bypass authentication for clients in whitelisted IP subnets" option for the sake of qui. Requests which come in through the reverse-proxy also originate from a container on the same kind of network, and they would bypass the authentication as well.

If you have enabled [Local Filesystem Access](#enabling-local-filesystem-access-optional), turn on the `Local Filesystem Access` toggle on the instance as well.

After adding the instance, you can manage qBittorrent from qui. As qui is capable of changing qBittorrent's preferences too, you might consider stopping to expose qBittorrent's own Web UI publicly by setting `qbittorrent_container_labels_traefik_enabled: false`.

### Integration with Sonarr/Radarr, autobrr, etc. (optional)

qui includes a reverse proxy for qBittorrent's Web API. Other applications can be pointed at it instead of at qBittorrent, so that they do not need to know qBittorrent's credentials. It also answers some of their frequent requests from qui's own cache, which reduces the load on qBittorrent.

To use it, open `Settings > Client Proxy` on qui and create a client API key for each application. See [this page](https://getqui.com/docs/features/reverse-proxy/) on the project's documentation for details about how to set up each application with the generated proxy URL.

For example, to add it to your [Sonarr](https://sonarr.tv/) or [Radarr](https://radarr.video/) instance, navigate to the form at `Settings > Download Clients > Add > qBittorrent` and do the following:

- Set the `Host` field to your qui hostname (without the protocol), and `Port` as 443. Make sure to click `Use SSL`.
- Set the `URL Base` field to the path of the proxy URL (e.g. `/proxy/abc123...`). If you have set `qui_path_prefix`, prepend it to the path as well (e.g. `/qui/proxy/abc123...`).
- Leave the `Username` and `Password` fields blank.

>[!NOTE]
> If you are looking for an Ansible role for Sonarr and Radarr, you can check out [ansible-role-sonarr](https://github.com/mother-of-all-self-hosting/ansible-role-sonarr) and [ansible-role-radarr](https://github.com/mother-of-all-self-hosting/ansible-role-radarr).

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu qui` (or how you/your playbook named the service, e.g. `mash-qui`).

### Reset the password

If you have forgotten the password, you can set a new one by running the command below on the server. Replace `YOUR_USERNAME_HERE` with your username, and `qui` with the name of the container (e.g. `mash-qui`).

```sh
docker exec -it qui qui change-password --username YOUR_USERNAME_HERE
```
