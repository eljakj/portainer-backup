# portainer-backup

Kleiner Hilfs-Stack, der Portainers eigene Konfiguration (Stacks,
Environment-Variablen, Settings — gespeichert in einer eingebetteten BoltDB)
nachts automatisch sichert.

## Warum

Portainers Daten (`portainer_data`-Volume) liegen bewusst **lokal** auf
VM101, nicht auf TrueNAS/NFS: BoltDB braucht zuverlässiges File-Locking,
das NFS bei einem kurzen Netzwerk-Hänger oder TrueNAS-Neustart verlieren
kann — Risiko von Korruption oder einer hängenden Portainer-Instanz, und
zwar ausgerechnet dann, wenn man Portainer bräuchte, um das Problem zu
beheben.

Statt das Live-Volume auf NFS zu legen, zieht dieser Stack stattdessen
regelmäßig einen **Snapshot** über Portainers eigene Backup-API
(`POST /api/backup`) und legt den auf TrueNAS ab. Die API erzeugt den
Snapshot intern konsistent — kein Risiko, mitten in einem Schreibvorgang
zu kopieren, kein Container-Stopp nötig.

## Funktionsweise

- Container läuft dauerhaft (`restart: unless-stopped`) und macht einmal
  direkt beim Start und danach alle 24h ein Backup
- Ruft `https://192.168.178.17:9443/api/backup` mit einem Portainer-API-Key auf
- Speichert das Ergebnis als `.zip` in einem NFS-Volume auf TrueNAS
  (`/mnt/mainpool/docker/docker-volume/portainer_backups`)
- Behält die letzten 14 Tage, ältere Backups werden automatisch gelöscht

## Setup

1. In Portainer einen API-Token erstellen: Benutzer-Icon → **My account**
   → **Access tokens** → **Add access token**
2. Auf TrueNAS den Ordner `portainer_backups` unter `docker-volume` anlegen
   (falls noch nicht vorhanden)
3. Stack in Portainer anlegen (Web editor, Inhalt aus `docker-compose.yml`)
4. Environment-Variable `PORTAINER_API_KEY` mit dem Token aus Schritt 1
   eintragen
5. Deploy

## Wiederherstellung

Im Portainer-Web-UI unter **Settings** gibt es eine Restore-Funktion, die
genau so ein Backup-Zip wieder einspielt. Einfach die aktuellste Datei aus
`portainer_backups` auf TrueNAS herunterladen und dort hochladen.
