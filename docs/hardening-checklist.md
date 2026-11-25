# Härtungs-Checkliste (OS, Container, Netzwerk)

Jede Maßnahme dient der Reduzierung der Angriffsfläche und der Begrenzung möglicher Schäden bei Kompromittierung einer Einzelkomponente.

## Betriebssystem (Host)

- [ ] **Transaktionales Root-Dateisystem (openSUSE MicroOS):** Verhindert unautorisierte Änderungen an Systembinärdateien zur Laufzeit; Updates erfolgen atomar mit automatischem Rollback.
- [ ] **SELinux im Modus `enforcing`:** Erzwingt Mandatory Access Control (MAC) zwischen Host-Prozessen und Containern, sodass ein Container-Ausbruch nicht automatisch Host-Dateien lesen/schreiben kann.
- [ ] **SSH ausschließlich per Public-Key-Authentifizierung:** Passwort-Logins deaktiviert (`PasswordAuthentication no`), um Brute-Force-Angriffe wirkungslos zu machen.
- [ ] **SSH-Zugriff auf WireGuard-Zone (`wg-management-ip`) beschränkt:** Reduziert die Angriffsfläche des SSH-Daemons auf authentifizierte VPN-Peers.
- [ ] **Minimale Host-Pakete:** Nur zwingend benötigte Pakete auf dem Host installiert; alle Anwendungsabhängigkeiten liegen gekapselt in Containern.

## Container (Podman)

- [ ] **Read-Only Root-Dateisystem (`read_only: true`):** Verhindert, dass ein kompromittierter Prozess zur Laufzeit Binärdateien oder Skripte im Container verändert.
- [ ] **Alle Linux Capabilities entfernt (`cap_drop: [ALL]`):** Entzieht dem Container unnötige Kernel-Privilegien; nur explizit benötigte Capabilities werden gezielt wieder gewährt.
- [ ] **Keine Rechteausweitung (`security_opt: ["no-new-privileges:true"]`):** Blockiert `setuid`/`setgid`-Binaries und verhindert Privilegieneskalation innerhalb des Containers.
- [ ] **Restriktive `tmpfs`-Mounts (`noexec,nosuid,nodev`):** Temporäre Verzeichnisse wie `/tmp` oder `/var/cache/nginx` liegen im RAM und erlauben keine Ausführung von Binaries.
- [ ] **Non-Root-Ausführung (`user:` / unprivilegierte Images):** Container-Prozesse laufen mit unprivilegierten UIDs (z. B. `101` für `nginx-unprivileged`), nicht als `root` (`UID 0`).
- [ ] **Lokale Port-Bindung (`127.0.0.1:<port>:<port>`):** Container-Ports werden niemals direkt an `0.0.0.0` gebunden, sondern nur lokal für den Reverse Proxy exponiert.
- [ ] **Automatisierte CVE-Scans (`trivy`):** Regelmäßige Prüfung aller Images auf bekannte Schwachstellen, gekoppelt an Wazuh-Alerts.

## Netzwerk & Reverse Proxy

- [ ] **Striktes Firewall-Regelwerk (`firewalld` / `nftables`):** Standardmäßig `drop` für eingehenden Traffic; öffentlich offen sind nur `80/TCP` (Redirect/ACME) und `443/TCP` (HTTPS).
- [ ] **WireGuard-Segmentierung:** Management-Dienste (Cockpit, Grafana, Wazuh) sind ausschließlich über das WireGuard-Interface (`wg-management-ip`) erreichbar.
- [ ] **TLS 1.2/1.3 mit Let's Encrypt:** Verschlüsselter Transport inklusive HSTS (`Strict-Transport-Security`).
- [ ] **HTTP-Security-Header:** `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin` und restriktive `Content-Security-Policy` (`default-src 'self'`).
