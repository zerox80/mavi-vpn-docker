# Mavi VPN · Docker

[![Publish VPN image](https://github.com/zerox80/mavi-vpn-docker/actions/workflows/publish.yml/badge.svg)](https://github.com/zerox80/mavi-vpn-docker/actions/workflows/publish.yml)

A prebuilt [Mavi VPN server](https://github.com/zerox80/mavi-vpn) for Linux VPS hosts.
GitHub Actions compiles the Rust code; your server only downloads the image from GHCR.
All you need to run it is `compose.yaml` and your own `.env` file.

- Image: `ghcr.io/zerox80/mavi-vpn:latest`
- Architectures: Linux AMD64 (x86_64) and ARM64 (aarch64)
- Runs only the VPN server; you can connect it to an existing Keycloak server.
- Certificates and ECH keys are persisted in `./data`.
- No Rust, Cargo, or local Docker build is required on the VPS.

## Requirements

Linux with [Docker Engine and the Compose plugin](https://docs.docker.com/engine/install/),
permission to use Docker, and an available `/dev/net/tun` device.
The container uses host networking and requires `NET_ADMIN`, `NET_RAW`,
and `NET_BIND_SERVICE` for TUN, routing, firewall rules, and privileged ports.
Only one instance using these firewall chains may run on a host.

Allow `10443/UDP` in both the host and provider firewalls (or the port chosen
in `VPN_BIND_ADDR`). For the optional HTTP/2 listener, also allow its TCP port.
There are no Compose port mappings because the container uses `network_mode: host`.

## Installation

```bash
git clone https://github.com/zerox80/mavi-vpn-docker.git
cd mavi-vpn-docker
cp .env.example .env
chmod 600 .env
openssl rand -hex 32
nano .env
```

Enter the generated token in `.env` as `VPN_AUTH_TOKEN=...`.
Your actual `.env` contains credentials and stays local; it is excluded by
`.gitignore`. Additional options are documented in the comments in `.env.example`.

Enable IPv4 forwarding on the host once and persist the setting:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-mavi-vpn-ipv4.conf
sudo sysctl -p /etc/sysctl.d/99-mavi-vpn-ipv4.conf
```

Then start the server:

```bash
docker compose pull
docker compose up -d
docker compose logs --tail=100 -f vpn-server
```

After startup, retrieve the certificate pin:

```bash
sudo cat data/cert_pin.txt
```

Enter the server address, port, token, and certificate pin in the client.
Standard QUIC without censorship resistance (CR) is used by default. To enable
CR, set `VPN_CENSORSHIP_RESISTANT=true` and select the matching mode in the client.

## Updates and rollbacks

```bash
git pull --ff-only
docker compose pull
docker compose up -d
```

Git does not replace `.env` or `data/` during this process. Copy new options from
`.env.example` as needed. Active VPN connections are interrupted when the
container is replaced and must reconnect.

`latest` contains the most recent successfully published build for both
architectures. The daily build does not install anything automatically on your
server; updates only take effect when you run the commands above.

Each published build also has its own tag,
`build-<run-id>-<attempt>`, listed in the run summary under
[Actions](https://github.com/zerox80/mavi-vpn-docker/actions/workflows/publish.yml).
To update or roll back to a specific build, set `MAVI_IMAGE` in `.env` to the
image with that tag, then run `docker compose pull && docker compose up -d` again.
You can also pin an immutable image using
`MAVI_IMAGE=ghcr.io/zerox80/mavi-vpn@sha256:<digest>`.

Keep and back up `data/`: losing the existing certificates changes the
certificate pin for all clients.

## Migrating an existing installation

1. Clone this repository into a new directory and copy `.env.example` to `.env`.
   Copy over your existing `VPN_*` values and use the same token. In particular,
   keep the CR mode, ports, IP networks, and IPv6 setting unchanged.
   Do not copy `COMPOSE_FILE` or `COMPOSE_PROFILES` from the old full stack.
2. Run `docker compose stop vpn-server` in the old `mavi-vpn/backend` directory.
   This releases the port, TUN device, and firewall chains used by the old server.
3. Copy the entire existing `backend/data` directory, including certificates,
   private keys, and ECH files, into this new repository as `data/`.
   Preserve ownership and file permissions, for example with `sudo cp -a`.
4. Remove the stopped VPN container by running `docker compose rm -f vpn-server`
   in its old directory to free up the `mavi-vpn` container name.
   Keep the old data directory as a backup.
5. Run `docker compose pull` and `docker compose up -d` in the new directory.

An existing Keycloak/Postgres stack can continue running separately. Enter its
reachable Keycloak URL and authentication options in the new `.env`.

## Memory on small VPS hosts

The memory-intensive Rust release build runs entirely on GitHub.
The running VPN's memory usage depends on connections and traffic;
no fixed minimum RAM requirement has been measured for this setup.

Compose limits the VPN container to `512m` by default through `MAVI_MEMORY_LIMIT`.
The combined RAM and swap limit has the same value, so the container cannot use
swap. If it exceeds the limit, the kernel may terminate the container; Docker
will then restart it. This does not prevent other processes from exhausting
the host's RAM. Increase the limit for heavy VPN traffic while leaving enough
memory for the operating system.

Check memory usage and the latest out-of-memory (OOM) status:

```bash
docker stats --no-stream mavi-vpn
docker inspect mavi-vpn --format '{{.State.OOMKilled}}'
```

Logs are limited to three files of 10 MB each. Keycloak and PostgreSQL are not
part of this Compose configuration; account for their resource usage separately
when planning capacity.

## Enabling IPv6

`VPN_DISABLE_IPV6=true` is the default to simplify initial setup.
If the host has public IPv6 connectivity, configure forwarding first. When
using router advertisements (RAs), the WAN interface must continue accepting them.
Identify the interface and save the settings in this order:

```bash
MAVI_WAN=$(ip -4 route get 1.1.1.1 | awk '{for (i=1; i<=NF; i++) if ($i=="dev") {print $(i+1); exit}}')
sudo tee /etc/sysctl.d/99-mavi-vpn-ipv6.conf >/dev/null <<CONF
net.ipv6.conf.${MAVI_WAN}.accept_ra = 2
net.ipv6.conf.default.accept_ra = 2
net.ipv6.conf.all.forwarding = 1
CONF
sudo sysctl -p /etc/sysctl.d/99-mavi-vpn-ipv6.conf
```

Then set `VPN_DISABLE_IPV6=false` in `.env` and run `docker compose up -d`.
`VPN_NETWORK_V6=fd00::/64` is the internal VPN network; the server uses NAT66.
Do not set it to the VPS's public IPv6 address.

See the [server installation guide](https://github.com/zerox80/mavi-vpn/blob/main/docs/INSTALLATION.md)
for more details.

## Optional: HTTP/2 and Keycloak

To enable HTTP/2, set `VPN_HTTP2_BIND_ADDR=0.0.0.0:10443` and allow `10443/TCP`
through the firewalls. Select HTTP/2 in the client without CR, HTTP/3 framing,
or ECH. The UDP listener remains available alongside HTTP/2.

To use an existing Keycloak server, configure `VPN_KEYCLOAK_ENABLED=true`,
`VPN_KEYCLOAK_URL`, the realm, and the client ID in `.env`. Roles and scopes are
optional. A static `VPN_AUTH_TOKEN` is not required when Keycloak is enabled.
This repository does not include a Keycloak or Traefik stack; the full version
is available in the [source project](https://github.com/zerox80/mavi-vpn/tree/main/backend).

## Publishing the image (maintainers)

The `Publish VPN image` workflow runs when the image definition changes on
`main`, is scheduled daily at 06:17 UTC, and can be started manually through
**Actions → Run workflow**. Changes in the source project are picked up by the
next daily run; start the workflow manually to publish them sooner.

The workflow:

1. Records the current `main` commit from `zerox80/mavi-vpn`.
2. Updates all seven `zerox80` fork packages to their latest branch commits
   and verifies their sources using the existing fork verification script.
3. Builds AMD64 and ARM64 natively on separate GitHub runners using the same
   source revision and updated `Cargo.lock`. No tests are run.
4. Publishes `latest` and the build tag only after both builds succeed.

The source code stays in the main repository. This pipeline does not modify
`Cargo.toml` or `Cargo.lock` there. The lockfile used for the build is saved as
an Actions artifact and included in the image at `/usr/share/mavi-vpn/Cargo.lock`.
The server revision is recorded in the OCI label `org.opencontainers.image.revision`;
the deployment revision is in `io.mavi-vpn.deployment.revision`.

GHCR authentication automatically uses `GITHUB_TOKEN` with `packages: write`.
No additional registry secrets are required.
**After the first publication, set the GHCR package visibility to Public:**
GitHub profile → Packages → `mavi-vpn` → Package settings → Change visibility → Public.
A public Git repository does not automatically make a new container package
public. This step allows `docker compose pull` to work without logging in to GitHub.
See the [GitHub Container Registry documentation](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

## License

[MIT](LICENSE). The included VPN server comes from
[zerox80/mavi-vpn](https://github.com/zerox80/mavi-vpn).
