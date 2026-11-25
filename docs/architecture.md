# Systemarchitektur und Design-Entscheidungen

Dieses Dokument beschreibt die Komponenten der Referenzarchitektur auf einem Einzel-Host-VPS sowie die technischen Gründe für die jeweiligen Entscheidungen.

## 1. Betriebssystem: openSUSE MicroOS

- **Eigenschaften:** Transaktionales, weitgehend unveränderliches (immutable) Root-Dateisystem (`btrfs` mit Snapshots), atomare Updates über `transactional-update`, automatisches Rollback bei fehlgeschlagenem Boot-Healthcheck.
- **Warum ein transaktionales OS statt einer klassischen Distribution?**
  Auf einem produktiven Einzel-Host reduziert ein unveränderliches Root-Dateisystem das Risiko von Konfigurationsdrift und unbemerkten Systemmanipulationen. Paketänderungen werden erst beim nächsten definierten Reboot aktiv und lassen sich bei Problemen atomar zurückrollen.

## 2. Container-Laufzeitumgebung: Podman & Compose

- **Eigenschaften:** Daemonless-Architektur, kein zentraler `root`-Daemon erforderlich, native Systemd-Integration, SELinux-Labeling (`:Z` / `:z`), kompatibel mit `compose.yaml`.
- **Warum Podman statt Docker?**
  Docker setzt standardmäßig auf einen zentralen, mit `root`-Rechten laufenden Daemon (`dockerd`), der einen breiten Angriffspunkt darstellt. Podman führt Container ohne zentralen Daemon aus, unterstützt Rootless-Betrieb sowie strikte SELinux-Trennung nativ und minimiert dadurch die Angriffsfläche auf dem Host.
- **Container-Härtung:** Jeder Dienst wird nach dem Prinzip der minimalen Rechte konfiguriert (`read_only: true`, `cap_drop: [ALL]`, `no-new-privileges:true`, `tmpfs` mit `noexec,nosuid`, dedizierte Non-Root-UIDs).
- **Wartung:** Container-Updates erfolgen automatisiert in einem definierten Wartungsfenster inklusive anschließender Healthcheck-Prüfung.

## 3. Netzwerksegmentierung: WireGuard & Reverse Proxy

- **Öffentliche Edge:** Ausschließlich Ports `80/TCP` (ACME-Challenge und HTTPS-Redirect) und `443/TCP` (TLS-Terminierung) sind öffentlich erreichbar.
- **Management-Zone (`wg-management-ip`):** Administrative Dienste (SSH, Cockpit, Grafana, Wazuh-Dashboard) sind nicht öffentlich gebunden, sondern ausschließlich über das WireGuard-Interface erreichbar.
- **Begründung:** Selbst bei einer Schwachstelle in einem Management-Interface ist kein direkter Zugriff aus dem öffentlichen Internet möglich.

## 4. Monitoring & Schwachstellenmanagement: Wazuh, Trivy & Grafana

- **Wazuh (SIEM/HIDS):** Sammelt Host-Logs, Firewall-Events und Container-Sicherheitsereignisse zentral ein.
- **Trivy-Integration:** Scannt regelmäßig alle laufenden Container-Images (insbesondere selbst gebaute Multi-Stage-Images) auf bekannte CVEs und meldet Befunde an Wazuh.
- **Grafana:** Visualisiert Systemmetriken, Sicherheitsereignisse und Backup-Status innerhalb der WireGuard-Management-Zone.

## 5. Produktive Dienste

- **Mail & Groupware:** Stalwart Mailserver (IMAP/SMTP/JMAP sowie CalDAV/CardDAV-Sync).
- **Automatisierung & Assistenz:** `n8n` für deterministische Workflows sowie ein eigener KI-Assistent-Dienst (E-Mail-Triage, Kalender-Integration, Benachrichtigungen über lokale Signal-API).

## 6. Backup-Architektur: `restic`

- **Warum `restic` statt Borg oder reinem `rclone`?**
  - Gegenüber reinem `rclone` bietet `restic` native, clientseitige Ende-zu-Ende-Verschlüsselung, Content-basierte Deduplizierung und atomare Snapshots.
  - Gegenüber `Borg` unterstützt `restic` nativ `rclone`-Backends als Ziel-Transport, wodurch sich Cloud-Speicher wie Proton Drive (primär) und pCloud (sekundär) ohne eigene SSH/Borg-Server-Infrastruktur auf Anbieterseite anbinden lassen.
