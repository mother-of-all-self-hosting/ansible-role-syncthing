<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Syncthing

This is an [Ansible](https://www.ansible.com/) role which installs [Syncthing](https://syncthing.net/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Syncthing is a continuous file synchronization program which synchronizes files between two or more computers in real time, safely protected from prying eyes.

See the project's [documentation](https://docs.syncthing.net/) to learn what Syncthing does and why it might be useful to you.

## Adjusting the playbook configuration

To enable Syncthing with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# syncthing                                                            #
#                                                                      #
########################################################################

syncthing_enabled: true

# By default, the data directory is created at (`/mash/syncthing/data`), as defined below.
# If you'd like to put it elsewhere on the host, uncomment and edit the line below.
#
# Regardless of the location of the data directory on the host,
# it will be mounted into the Syncthing container at `/data`.
# syncthing_data_path: "{{ syncthing_base_path }}/data"

# To mount additional data directories, use `syncthing_container_additional_volumes`.

# Secure with HTTP Basic Auth (at the Traefik level)
syncthing_basicauth_enabled: true

# Generate each entry with `htpasswd -nb USERNAME PASSWORD`
# and paste the whole `USERNAME:HASH` line below.
# Example:
# htpasswd -nb someone 'secret-password'
# htpasswd -nb another 'more-secret-password'
syncthing_basicauth_htpasswds:
  - 'someone:$apr1$Dz1QzvR9$TQj8rP2QfLz7dYkP6Y0K4/'
  - 'another:$apr1$QfJ1mU7a$gR0d9D0dKfIDm0w3lN4hY0'

########################################################################
#                                                                      #
# /syncthing                                                           #
#                                                                      #
########################################################################
```

### Set the hostname

To enable Syncthing you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
syncthing_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

### Configuring HTTP Basic authentication

For Syncthing, it is configured to enable the HTTP Basic authentication on Traefik by default. Refer to [this page](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/basicauth/) on the Traefik's documentation for details.

You can use `htpasswd` to generate the user and password pair, which needs to be set to `syncthing_basicauth_htpasswds`.

You can log in with **any** of the Basic Auth credentials defined in `syncthing_basicauth_htpasswds`. Syncthing is **not a multi-user system**, so whichever user you authenticate with, you'd ultimately end up looking at the same shared system.

Upon logging in, Syncthing will show you warnings about no GUI password being set. You can safely ignore them, and suppress them on the advanced settings on Syncthing's UI.

If the Syncthing's own authentication system is preferred, another authentication service than Traefik's is used, or authentication is not required at all, you can disable it by adding the following configuration to your `vars.yml` file:

```yaml
syncthing_basicauth_enabled: false
```

>[!NOTE]
> The legacy `syncthing_basicauth_credentials` convenience variable is discouraged.

### Networking

By default, the following ports will be exposed by the container on **all network interfaces**:

- `22000` over **TCP**, controlled by `syncthing_container_sync_tcp_bind_port` and `syncthing_container_sync_tcp_port` — used for TCP based sync protocol traffic
- `22000` over **UDP**, controlled by `syncthing_container_sync_udp_bind_port` and `syncthing_container_sync_udp_port` — used for QUIC based sync protocol traffic
- `21027` over **UDP**, controlled by `syncthing_container_local_discovery_udp_bind_port` — used for discovery broadcasts on IPv4 and multicasts on IPv6

Docker automatically opens these ports in the server's firewall, so you **likely don't need to do anything**. If you use another firewall in front of the server, you may need to adjust it.

If you have multiple devices on the same LAN, you may wish to assign a unique port to each one as recommended in the [Local network setup section on ArchWiki](https://wiki.archlinux.org/title/Syncthing#Local_network_setup).

As the upstream [Firewall documentation](https://docs.syncthing.net/users/firewall.html) says:

> The external forwarded ports and the internal destination ports have to be the same (e.g. 22000/TCP).

Because of this, the Syncthing Ansible role makes the actually exposed ports (`syncthing_container_sync_*_bind_port` variables) the same as the ports that the Syncthing program in the container actually listens on (`syncthing_container_sync_tcp_port` or `syncthing_container_sync_udp_port`). That is to say, **the `_bind_port` variables are automatically adjusted** based on the values of `syncthing_container_sync_tcp_port` and `syncthing_container_sync_udp_port`.

However, changing `syncthing_container_sync_tcp_port` or `syncthing_container_sync_udp_port` in Ansible does not change the Syncthing configuration and the port Syncthing decides to listen, but merely tells the Ansible role which ports you'd like to use, so it can wire things correctly.

**To effectively change the Syncthing ports** being used:

1. Adjust `syncthing_container_sync_tcp_port` and `syncthing_container_sync_udp_port` in your `vars.yml`
2. Re-install the Syncthing service by re-running the Ansible playbook
3. Log in to the Syncthing Web UI (refer to [Usage](#usage))
4. Go to **Settings** -> **Connections** and put something like this in the **Sync Protocol Listen Addresses** configuration (inspired by the [Listen Addresses documentation](https://docs.syncthing.net/v1.27.0/users/config#listen-addresses)): `tcp://0.0.0.0:TCP_PORT_HERE, quic://0.0.0.0:UDP_PORT_HERE, dynamic+https://relays.syncthing.net/endpoint` (adjust `TCP_PORT_HERE` and `UDP_PORT_HERE` with the port numbers you've chosen for `syncthing_container_sync_tcp_port` and `syncthing_container_sync_udp_port`)

### Configuration & Data

The Syncthing configuration (stored in `syncthing_config_path` on the host) is mounted to the `/var/syncthing` directory in the container. By default, Syncthing will create a default `Sync` directory underneath. We advise that you **don't use this** `Sync` directory and use the data directory (discussed below).

As mentioned above, the **data directory** (stored in `syncthing_data_path` on the host) is mounted to the `/data` directory in the container. We advise that you put data files underneath `/data` when you start using Syncthing.

If you'd like to **mount additional directories** into the container, look into the `syncthing_container_additional_volumes` variable part of the [`ansible-role-syncthing` role](https://github.com/mother-of-all-self-hosting/ansible-role-syncthing)'s [`defaults/main.yml` file](https://github.com/mother-of-all-self-hosting/ansible-role-syncthing/blob/main/defaults/main.yml).

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `syncthing_environment_variables_additional_variables` variable

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Syncthing becomes available at the specified hostname like `https://example.com`.

As mentioned in [Configuration & Data](#configuration--data) above, consider to:

- get rid of the `Default Folder` directory that has been automatically created in `/var/syncthing/Sync`
- change the default data directory, by going to **Actions** -> **Settings** -> **General** tab -> **Edit Folder Defaults** and changing **Folder Path** to `/data`

Also, as mentioned in [Authentication](#configuring-http-basic-authentication), you'd wish to disable the "no GUI password set" security warnings, if either the HTTP Basic authentication on Traefik or another authentication service is used.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu syncthing` (or how you/your playbook named the service, e.g. `mash-syncthing`).
