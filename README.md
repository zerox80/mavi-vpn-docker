# Mavi VPN · Docker

[![Publish VPN image](https://github.com/zerox80/mavi-vpn-docker/actions/workflows/publish.yml/badge.svg)](https://github.com/zerox80/mavi-vpn-docker/actions/workflows/publish.yml)

Fertiger [Mavi-VPN-Server](https://github.com/zerox80/mavi-vpn) für Linux-vServer.
GitHub Actions kompiliert Rust; dein Server lädt nur das Image aus GHCR.
Für den Betrieb reichen `compose.yaml` und eine eigene `.env`.

- Image: `ghcr.io/zerox80/mavi-vpn:latest`
- Architekturen: Linux AMD64 (x86_64) und ARM64 (aarch64)
- Startet nur den VPN-Server; ein vorhandener Keycloak-Server kann angebunden werden.
- Zertifikate und ECH-Schlüssel bleiben in `./data` erhalten.
- Kein Rust, Cargo oder lokaler Docker-Build auf dem vServer erforderlich.

## Voraussetzungen

Linux mit [Docker Engine und Compose-Plugin](https://docs.docker.com/engine/install/),
Zugriff auf Docker sowie ein verfügbares `/dev/net/tun`.
Der Container verwendet das Host-Netzwerk und benötigt `NET_ADMIN`, `NET_RAW`
und `NET_BIND_SERVICE` für TUN, Routing, Firewall und privilegierte Ports.
Auf einem Host darf nur eine Instanz mit diesen Firewall-Ketten laufen.

In der Host- und Provider-Firewall `10443/UDP` freigeben (oder den gewählten
Port aus `VPN_BIND_ADDR`). Für den optionalen HTTP/2-Listener zusätzlich dessen
TCP-Port freigeben. Wegen `network_mode: host` gibt es keine Compose-Portzuordnung.

## Installation

```bash
git clone https://github.com/zerox80/mavi-vpn-docker.git
cd mavi-vpn-docker
cp .env.example .env
chmod 600 .env
openssl rand -hex 32
nano .env
```

Den erzeugten Token in `.env` als `VPN_AUTH_TOKEN=...` eintragen.
Die echte `.env` enthält Zugangsdaten und bleibt lokal; sie ist in `.gitignore`
ausgeschlossen. Weitere Optionen stehen kommentiert in `.env.example`.

IPv4-Forwarding einmalig auf dem Host aktivieren und dauerhaft speichern:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-mavi-vpn-ipv4.conf
sudo sysctl -p /etc/sysctl.d/99-mavi-vpn-ipv4.conf
```

Danach starten:

```bash
docker compose pull
docker compose up -d
docker compose logs --tail=100 -f vpn-server
```

Nach dem Start den Zertifikat-PIN auslesen:

```bash
sudo cat data/cert_pin.txt
```

Im Client Serveradresse, Port, Token und Zertifikat-PIN eintragen. Standardmäßig
wird normales QUIC ohne CR-Modus verwendet. Für CR `VPN_CENSORSHIP_RESISTANT=true`
setzen und den entsprechenden Modus im Client wählen.

## Updates und Rückkehr zu einer älteren Version

```bash
git pull --ff-only
docker compose pull
docker compose up -d
```

Die `.env` und `data/` werden dabei nicht durch Git ersetzt. Neue Optionen bei
Bedarf aus `.env.example` übernehmen. Laufende VPN-Verbindungen werden beim
Containerwechsel unterbrochen und müssen sich neu verbinden.

`latest` enthält den letzten erfolgreich veröffentlichten Build beider
Architekturen. Der tägliche Build installiert nichts automatisch auf deinem
Server; ein Update erfolgt erst durch die obigen Befehle.

Jeder veröffentlichte Build hat außerdem einen eigenen Tag
`build-<run-id>-<versuch>`. Er steht in der Zusammenfassung unter
[Actions](https://github.com/zerox80/mavi-vpn-docker/actions/workflows/publish.yml).
Für ein gezieltes Update oder Rollback `MAVI_IMAGE` in `.env` auf diesen Tag
setzen und erneut `docker compose pull && docker compose up -d` ausführen.
Für eine unveränderliche Auswahl ist auch
`MAVI_IMAGE=ghcr.io/zerox80/mavi-vpn@sha256:<digest>` möglich.

`data/` aufbewahren und sichern: Ohne die bisherigen Zertifikate ändert sich
der Zertifikat-PIN für alle Clients.

## Bestehende Installation übernehmen

1. Dieses Repository in einen neuen Ordner klonen und `.env.example` als `.env`
   kopieren. Die bisherigen `VPN_*`-Werte und denselben Token übernehmen.
   Insbesondere CR-Modus, Ports, IP-Netze und IPv6-Einstellung beibehalten.
   `COMPOSE_FILE` und `COMPOSE_PROFILES` des alten Komplett-Stacks nicht übernehmen.
2. Im alten `mavi-vpn/backend`-Ordner `docker compose stop vpn-server` ausführen.
   Dadurch gibt der bisherige Server Port, TUN-Gerät und Firewall-Ketten frei.
3. Den kompletten bisherigen `backend/data`-Ordner inklusive Zertifikaten,
   privaten Schlüsseln und ECH-Dateien als `data/` in dieses neue Repository
   kopieren. Eigentümer und Dateirechte erhalten, beispielsweise mit `sudo cp -a`.
4. Den gestoppten alten VPN-Container mit `docker compose rm -f vpn-server`
   aus dessen altem Ordner entfernen, damit der Name `mavi-vpn` frei ist.
   Den alten Datenordner als Sicherung behalten.
5. Im neuen Ordner `docker compose pull` und `docker compose up -d` ausführen.

Ein vorhandener Keycloak-/Postgres-Stack kann separat weiterlaufen. Seine
erreichbare URL und die Authentifizierungsoptionen in der neuen `.env` eintragen.

## RAM auf kleinen vServern

Der speicherintensive Rust-Release-Build findet vollständig auf GitHub statt.
Der Speicherbedarf des laufenden VPN hängt von Verbindungen und Datenverkehr ab;
eine feste Mindest-RAM-Angabe wurde hier nicht gemessen.

Compose begrenzt den VPN-Container zunächst auf `512m` über `MAVI_MEMORY_LIMIT`.
Das kombinierte RAM-/Swap-Limit hat denselben Wert, sodass der Container keinen
Swap verwenden kann. Bei überschrittenem Limit kann der Kernel den Container
beenden; Docker startet ihn dann erneut. Das verhindert keinen RAM-Mangel durch
andere Prozesse auf dem Host. Bei hoher VPN-Last das Limit passend erhöhen und
dem Betriebssystem ausreichend RAM lassen.

Speichernutzung und letzten OOM-Status ansehen:

```bash
docker stats --no-stream mavi-vpn
docker inspect mavi-vpn --format '{{.State.OOMKilled}}'
```

Logs sind auf drei Dateien mit jeweils 10 MB begrenzt. Keycloak und PostgreSQL
laufen nicht in dieser Compose-Konfiguration und sind bei der Kapazitätsplanung
separat zu berücksichtigen.

## IPv6 aktivieren

Für einen einfachen Erststart ist `VPN_DISABLE_IPV6=true` voreingestellt.
Wenn der Host öffentliches IPv6 hat, zuerst Forwarding einrichten. Bei
Router-Advertisements muss das WAN-Interface weiterhin RAs akzeptieren.
Interface ermitteln und Einstellungen in dieser Reihenfolge speichern:

```bash
MAVI_WAN=$(ip -4 route get 1.1.1.1 | awk '{for (i=1; i<=NF; i++) if ($i=="dev") {print $(i+1); exit}}')
sudo tee /etc/sysctl.d/99-mavi-vpn-ipv6.conf >/dev/null <<CONF
net.ipv6.conf.${MAVI_WAN}.accept_ra = 2
net.ipv6.conf.default.accept_ra = 2
net.ipv6.conf.all.forwarding = 1
CONF
sudo sysctl -p /etc/sysctl.d/99-mavi-vpn-ipv6.conf
```

Dann `VPN_DISABLE_IPV6=false` in `.env` setzen und `docker compose up -d`
ausführen. `VPN_NETWORK_V6=fd00::/64` ist das interne VPN-Netz; der Server nutzt
NAT66. Dort nicht die öffentliche IPv6-Adresse des vServers eintragen.

Weitere Details stehen in der
[Server-Anleitung](https://github.com/zerox80/mavi-vpn/blob/main/docs/INSTALLATION.md).

## Optional: HTTP/2 und Keycloak

Für HTTP/2 `VPN_HTTP2_BIND_ADDR=0.0.0.0:10443` aktivieren und `10443/TCP`
freigeben. Im Client HTTP/2 ohne CR, HTTP/3-Framing oder ECH wählen.
Der UDP-Listener bleibt zusätzlich verfügbar.

Für vorhandenes Keycloak `VPN_KEYCLOAK_ENABLED=true`, `VPN_KEYCLOAK_URL`,
Realm und Client-ID in `.env` konfigurieren. Rollen und Scopes sind optional.
Bei aktivem Keycloak ist kein statischer `VPN_AUTH_TOKEN` erforderlich.
Dieses Repository enthält keinen Keycloak- oder Traefik-Stack; die vollständige
Variante liegt im [Quellprojekt](https://github.com/zerox80/mavi-vpn/tree/main/backend).

## Image veröffentlichen (Maintainer)

Der Workflow `Publish VPN image` läuft bei Änderungen an der Image-Definition
auf `main`, täglich um 06:17 UTC und manuell über **Actions → Run workflow**.
Änderungen am Quellprojekt werden spätestens beim nächsten täglichen Lauf
übernommen; für sofortige Veröffentlichung den Workflow manuell starten.

Der Ablauf:

1. Aktuellen `main`-Commit aus `zerox80/mavi-vpn` festhalten.
2. Alle sieben `zerox80`-Fork-Pakete auf ihre aktuellen Branch-Stände aktualisieren
   und die Quellen mit dem bestehenden Fork-Prüfskript kontrollieren.
3. AMD64 und ARM64 nativ auf getrennten GitHub-Runnern aus derselben Quellrevision
   und demselben aktualisierten `Cargo.lock` bauen. Es werden keine Tests ausgeführt.
4. Erst nach beiden erfolgreichen Builds `latest` und den Build-Tag veröffentlichen.

Der Quellcode bleibt im Hauptrepository. Diese Pipeline verändert dort weder
`Cargo.toml` noch `Cargo.lock`. Das verwendete Lockfile liegt als Actions-Artefakt
und dauerhaft im Image unter `/usr/share/mavi-vpn/Cargo.lock`.
Die Serverrevision steht im OCI-Label `org.opencontainers.image.revision`;
die Deployment-Revision im Label `io.mavi-vpn.deployment.revision`.

Die Anmeldung bei GHCR verwendet automatisch `GITHUB_TOKEN` mit `packages: write`.
Es werden keine zusätzlichen Registry-Secrets benötigt.
**Nach der ersten Veröffentlichung das GHCR-Paket einmalig auf Public stellen:**
GitHub-Profil → Packages → `mavi-vpn` → Package settings → Change visibility → Public.
Ein öffentliches Git-Repository macht ein neues Container-Paket nicht automatisch
öffentlich. Erst danach funktioniert `docker compose pull` ohne GitHub-Anmeldung.
Siehe [GitHub-Dokumentation zur Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

## Lizenz

[MIT](LICENSE). Der enthaltene VPN-Server stammt aus
[zerox80/mavi-vpn](https://github.com/zerox80/mavi-vpn).
