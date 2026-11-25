# Backup- und Restore-Strategie

## 1. Grundprinzip: Clientseitige Verschlüsselung

Alle Backups werden vor dem Verlassen des Servers durch `restic` lokal verschlüsselt und dedupliziert. Die Speicheranbieter (Proton Drive und pCloud) sehen ausschließlich verschlüsselte Blobs und Metadaten ohne Klartextzugriff auf Dateinamen oder Inhalte.

## 2. Repository-Topologie

| Rolle | Ziel | Frequenz | Transport |
| :--- | :--- | :--- | :--- |
| **Primäres Repo** | Proton Drive (`rclone:proton:backups/vps_backup`) | Täglich | `restic` über `rclone` (separate `rclone.conf`) |
| **Sekundäres Repo** | pCloud (`rclone:pcloud:vps-userdata-repo`) | Wöchentlich | `restic` über `rclone` (getrennte `pcloud.conf` zur Vermeidung von Token-Refresh-Konflikten) |

*Hinweis zur Trennung der `rclone`-Konfigurationen:* Da `rclone` OAuth-/Session-Tokens bei der Ausführung zurückschreibt, verhindert eine getrennte Konfigurationsdatei pro Anbieter parallele Schreibkonflikte.

## 3. 3-2-1-Backup-Abbildung

- **3 Kopien:** Produktivdaten auf dem Host + primäres Repo (Proton Drive) + sekundäres Repo (pCloud).
- **2 verschiedene Medien/Systeme:** Lokales `btrfs`/NVMe-Storage auf dem VPS + Cloud-Object/Drive-Storage.
- **1 Offsite-Kopie:** Beide `restic`-Repositories liegen physisch getrennt vom VPS bei zwei unabhängigen europäischen bzw. externen Anbietern.

## 4. Integritätsprüfung

- **Täglicher Check:** Automatisierter Lauf von `restic check` zur Verifikation der Repository-Struktur und Index-Konsistenz.
- **Stichproben-Restore:** Regelmäßige Prüfung kritischer Datenbank-Dumps (z. B. SQLite-Integritätscheck nach Test-Restore), um sicherzustellen, dass Backups nicht nur geschrieben, sondern auch lesbar sind.

## 5. Restore-Pfad (Wiederherstellung)

Ein Backup ist erst dann belastbar, wenn der Restore-Pfad getestet ist.

### Einzelne Dateien oder Verzeichnisse wiederherstellen

```bash
# Snapshots auflisten
restic -r rclone:proton:backups/vps_backup snapshots

# Spezifischen Pfad aus dem letzten Snapshot wiederherstellen
restic -r rclone:proton:backups/vps_backup restore latest \
  --target /tmp/restore-test \
  --include /pfad/zur/datei
```

### Kompletter Host-Neuaufbau (Disaster Recovery)

1. Neuen Host mit openSUSE MicroOS aufsetzen und `restic` + `rclone` installieren.
2. `rclone.conf` (Anbieter-Zugang) und `restic`-Repository-Passwort aus dem Passwort-Safe bereitstellen.
3. Konfigurationsdateien (`/etc/`, Compose-Stacks, Datenbank-Dumps) in ein Staging-Verzeichnis wiederherstellen.
4. Datenbank-Dumps vor Container-Start in die jeweiligen Volumes importieren und Container mit `podman compose up -d` starten.
5. Falls das primäre Repo (Proton Drive) nicht erreichbar ist, identischen Ablauf gegen das sekundäre Repo (`rclone:pcloud:vps-userdata-repo`) ausführen.
