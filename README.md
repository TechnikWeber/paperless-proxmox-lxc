# Paperless-ngx auf Proxmox mit Claude Haiku

Komplette Anleitung zum Aufsetzen von Paperless-ngx in einem LXC-Container auf Proxmox mit Docker Compose, Tika/Gotenberg für erweiterte Dateiformat-Unterstützung, KI-Anbindung via Claude Haiku, und einer mehrstufigen Backup-Strategie aufs NAS.

## Inhaltsverzeichnis

1. [Überblick und Entscheidungen](#1-überblick-und-entscheidungen)
2. [Alten LXC löschen (falls vorhanden)](#2-alten-lxc-löschen-falls-vorhanden)
3. [Neuen LXC in Proxmox erstellen](#3-neuen-lxc-in-proxmox-erstellen)
4. [Docker installieren](#4-docker-installieren)
5. [Paperless-ngx mit Tika und Gotenberg installieren](#5-paperless-ngx-mit-tika-und-gotenberg-installieren)
6. [Grundeinrichtung in Paperless](#6-grundeinrichtung-in-paperless)
7. [Claude Haiku per paperless-ai anbinden](#7-claude-haiku-per-paperless-ai-anbinden)
8. [Updates](#8-updates)
9. [Backup-Strategie](#9-backup-strategie)
10. [Restore im Notfall](#10-restore-im-notfall)
11. [FAQ und Troubleshooting](#11-faq-und-troubleshooting)

---

## 1. Überblick und Entscheidungen

### Warum LXC statt VM?

- Leichter, weniger RAM-Overhead, schnellerer Start
- Reicht für Paperless-ngx vollkommen aus
- Docker-in-LXC ist stabil, benötigt `nesting=1` und `keyctl=1`

Nachteile von LXC (für diesen Use-Case nicht relevant):
- Docker supportet offiziell nur VMs
- GPU-Passthrough komplizierter (falls später lokales LLM)
- Weniger Isolation vom Host

### Warum Docker Compose ohne Portainer?

- **Weniger Abhängigkeiten**: Paperless hängt nicht von einer zusätzlichen Verwaltungs-Software ab
- **Compose-Files sind versionierbar**: `docker-compose.yml` und `.env` lassen sich in Git legen
- **Einfacher zu debuggen**: Keine Zwischenschicht, direkter Zugriff auf Docker
- **Keine "Limited Stack"-Probleme**: Portainer und extern erstellte Stacks vertragen sich nicht immer

Portainer lässt sich später jederzeit nachinstallieren, falls du eine GUI willst.

### Warum Tika und Gotenberg?

**Paperless-ngx Basis** (Redis, PostgreSQL, Webserver) kann nur:
- PDF, PNG, JPG, TIFF

**Mit Tika + Gotenberg** zusätzlich:
- Word (.docx, .doc), Excel (.xlsx), PowerPoint (.pptx)
- OpenOffice/LibreOffice (.odt, .ods)
- E-Mails (.eml, .msg)
- HTML-Dateien

**Tika** extrahiert Text und Metadaten, **Gotenberg** wandelt Office-Dateien in PDF um. Zusammen ca. 300–500 MB zusätzlicher RAM-Bedarf.

### Warum Claude Haiku statt lokalem Ollama?

Auf schwacher Hardware (z.B. Ryzen 5 3500U mit Vega 8):
- 20 Sekunden bis 3 Minuten pro Dokument
- Häufige Halluzinationen bei kleinen Modellen (3B–8B)
- Mittelmäßige Deutsch-Qualität

Claude Haiku 4.5:
- 2–5 Sekunden pro Dokument
- Sehr zuverlässige strukturierte Ausgaben
- Sehr gute Deutsch-Qualität
- Kosten: ~0,001–0,01 € pro Dokument, bei 500 Dokumenten/Jahr unter 5 €

### Datenschutz-Überlegung

Für typische Privatunterlagen (Rechnungen, Verträge, Kontoauszüge) ist Claude Haiku unkritisch. Für sensible Dokumente (Gesundheit, Anwalt, Mandantendaten) Ausschluss-Tag `no-ai` verwenden oder manuell bearbeiten.

Anthropic speichert API-Inputs standardmäßig nicht zu Trainingszwecken, typischerweise 30 Tage zur Missbrauchserkennung. Server in den USA.

---

## 2. Alten LXC löschen (falls vorhanden)

**Vorher Daten sichern**, falls schon relevante Dokumente drin sind:

```bash
cd /opt/paperless
docker compose exec -T webserver document_exporter ../export
docker compose cp webserver:/usr/src/paperless/export/. /root/paperless-export/
```

Dann `/root/paperless-export/` irgendwohin kopieren.

**Löschen in der Proxmox-GUI:**

1. Container markieren → **Shutdown** (oder **Stop** erzwingen)
2. Rechts oben **More** → **Remove**
3. Bestätigen (auch Festplatte löschen lassen)

---

## 3. Neuen LXC in Proxmox erstellen

### 3.1 Debian 13 Template herunterladen

Am Proxmox-Host:

```bash
pveam update
```

In der GUI:
1. Storage `local` anklicken
2. Reiter **CT Templates**
3. **Templates** → `debian-13-standard` → **Download**
4. Warten bis "TASK OK"

### 3.2 Container erstellen

**Create CT** (rechts oben):

**General:**
- CT ID: z.B. `110`
- Hostname: `paperless`
- Password: sicheres Root-Passwort
- ✓ **Unprivileged container**

**Template:**
- Storage: `local`
- Template: `debian-13-standard_...`

**Disks:**
- Storage: dein Hauptstorage (z.B. `local-lvm`)
- Disk size: **20 GB**

**CPU:**
- Cores: **2** (4 bei viel paralleler OCR)

**Memory:**
- Memory: **3072** MB (mit Tika/Gotenberg)
- Swap: **512** MB

**Network:**
- Bridge: `vmbr0`
- IPv4: **DHCP** oder statisch

**DNS:**
- Leer lassen

**Confirm:**
- ✗ **"Start after created" NICHT anhaken**
- **Finish**

### 3.3 LXC für Docker vorbereiten

1. Container markieren
2. **Options** → **Features** → **Edit**
3. Haken setzen:
   - ✓ **keyctl**
   - ✓ **nesting**
4. **OK**

### 3.4 Container starten

1. **Start**
2. **Console** öffnen
3. Als `root` einloggen

---

## 4. Docker installieren

In der Container-Console:

```bash
apt update && apt upgrade -y
apt install -y ca-certificates curl gnupg nano cron rsync cifs-utils
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian trixie stable" > /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Falls `apt update` einen 404 für `trixie` wirft, `trixie` durch `bookworm` ersetzen.

Test:

```bash
docker run --rm hello-world
```

"Hello from Docker!" = alles gut.

---

## 5. Paperless-ngx mit Tika und Gotenberg installieren

### 5.1 Verzeichnis und Compose-Datei erstellen

```bash
mkdir -p /opt/paperless && cd /opt/paperless
nano docker-compose.yml
```

Folgenden Inhalt einfügen:

```yaml
services:
  broker:
    image: docker.io/library/redis:8
    restart: unless-stopped
    volumes:
      - redisdata:/data

  db:
    image: docker.io/library/postgres:18
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: paperless

  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - db
      - broker
      - gotenberg
      - tika
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-fs", "-S", "--max-time", "2", "http://localhost:8000"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - data:/usr/src/paperless/data
      - media:/usr/src/paperless/media
      - ./export:/usr/src/paperless/export
      - ./consume:/usr/src/paperless/consume
    env_file: docker-compose.env
    environment:
      PAPERLESS_REDIS: redis://broker:6379
      PAPERLESS_DBHOST: db
      PAPERLESS_TIKA_ENABLED: 1
      PAPERLESS_TIKA_GOTENBERG_ENDPOINT: http://gotenberg:3000
      PAPERLESS_TIKA_ENDPOINT: http://tika:9998

  gotenberg:
    image: docker.io/gotenberg/gotenberg:7.10
    restart: unless-stopped
    command:
      - "gotenberg"
      - "--chromium-disable-javascript=true"
      - "--chromium-allow-list=file:///tmp/.*"

  tika:
    image: docker.io/apache/tika:latest
    restart: unless-stopped

volumes:
  data:
  media:
  pgdata:
  redisdata:
```

Speichern: `Strg+O`, `Enter`, `Strg+X`.

**Wichtig:** Die drei Variablen `PAPERLESS_TIKA_ENABLED`, `PAPERLESS_TIKA_GOTENBERG_ENDPOINT` und `PAPERLESS_TIKA_ENDPOINT` **aktivieren** die Integration von Tika und Gotenberg in Paperless. Ohne diese Variablen würden die Container zwar laufen, Paperless würde sie aber ignorieren und Office-Dateien nicht verarbeiten. Die Services werden über die Container-Namen `gotenberg` und `tika` im internen Docker-Netzwerk angesprochen.

### 5.2 Environment-Datei erstellen

```bash
nano docker-compose.env
```

Folgenden Inhalt einfügen:

```
PAPERLESS_OCR_LANGUAGE=deu+eng
PAPERLESS_OCR_LANGUAGES=deu eng
PAPERLESS_TIME_ZONE=Europe/Berlin
PAPERLESS_SECRET_KEY=HIER_LANGEN_ZUFALLSSTRING_EINFUEGEN
PAPERLESS_URL=http://LXC-IP:8000
PAPERLESS_ADMIN_USER=admin
PAPERLESS_ADMIN_PASSWORD=DeinPasswortHier
```

**Wichtige Hinweise zur Syntax:**
- Keine spitzen Klammern `< >` – direkt den Wert hinter `=`
- Keine Anführungszeichen
- Keine Leerzeichen um das `=`
- Beim Passwort die Sonderzeichen `$`, `` ` ``, `\`, `"`, `'` vermeiden
- `LXC-IP` durch die tatsächliche IP ersetzen (z.B. `192.168.1.50`)

**OCR-Sprachen:**
- `PAPERLESS_OCR_LANGUAGE` (Singular, mit `+`): welche Sprachen OCR anwenden
- `PAPERLESS_OCR_LANGUAGES` (Plural, mit Leerzeichen): welche Sprachpakete installieren
- Nur Deutsch: `PAPERLESS_OCR_LANGUAGE=deu` (spart ca. 30–50% OCR-Zeit)

**Secret-Key generieren:**

```bash
openssl rand -base64 50
```

Output kopieren, als Wert für `PAPERLESS_SECRET_KEY` einsetzen.

### 5.3 Starten

```bash
cd /opt/paperless
docker compose up -d
```

Erster Start dauert 5–10 Minuten (Images laden, DB initialisieren, Tika/Gotenberg initialisieren).

Status prüfen:

```bash
docker compose ps
```

Du solltest 5 Container sehen: `broker`, `db`, `webserver`, `gotenberg`, `tika` – alle mit Status "Up". Der Webserver zeigt nach ca. 1 Minute "healthy".

Logs einsehen:

```bash
docker compose logs -f
```

`Strg+C` beendet nur die Anzeige, nicht die Container.

### 5.4 Tika/Gotenberg-Integration prüfen

Nach dem Start:

```bash
docker compose exec webserver cat /usr/src/paperless/paperless.conf 2>/dev/null | grep -i tika
```

Wenn du hier `PAPERLESS_TIKA_ENABLED` mit Wert `1` siehst, ist alles aktiv. Ab jetzt werden automatisch auch .docx, .xlsx, .eml etc. beim Upload verarbeitet – ohne weitere Einstellung in der Paperless-GUI.

**Was passiert intern beim Upload einer .docx:**
1. Paperless erkennt das Format
2. Schickt die Datei an Tika → Textextraktion + Metadaten
3. Schickt die Datei an Gotenberg → Konvertierung in PDF
4. Speichert Original + generiertes PDF + extrahierten Text
5. In der Weboberfläche siehst du die Datei als PDF-Vorschau, die Volltextsuche findet Inhalte

### 5.5 IP herausfinden und aufrufen

```bash
ip a | grep inet
```

Browser: `http://LXC-IP:8000`

Login: `admin` + gesetztes Passwort.

**Falls Admin nicht erstellt:**

```bash
docker compose exec webserver python manage.py createsuperuser
```

---

## 6. Grundeinrichtung in Paperless

### 6.1 Korrespondenten, Dokumenttypen, Tags

Sidebar links:
- **Verwaltung → Korrespondenten**: Absender/Empfänger (Bank, Arbeitgeber, Versicherung, Stadtwerke, Finanzamt, …)
- **Verwaltung → Dokumenttypen**: Kategorien (Rechnung, Vertrag, Mahnung, Bescheid, Kontoauszug, …)
- **Verwaltung → Tags**: Schlagworte (wichtig, steuer-2025, no-ai, …)

Beim Anlegen jeweils **Zuordnungsalgorithmus: Auto** für ML-Training.

**Wichtig:** Tag `no-ai` direkt anlegen – für Dokumente, die nicht an Claude geschickt werden sollen.

### 6.2 Eingebautes ML

Automatisch aktiv, sobald Dinge auf "Auto" stehen. Trainiert sich nachts selbst, sobald 20–30 getaggte Dokumente da sind.

Das ML ist ein **Klassifikator**: Wählt nur aus bestehenden Tags/Korrespondenten. Halluziniert nicht.

Manuell antriggern:

```bash
cd /opt/paperless
docker compose exec webserver document_retagger --use-first --overwrite
```

### 6.3 Dokumente importieren

- **Upload-Button** in der Weboberfläche
- **Drag & Drop** ins Dashboard
- **Consume-Ordner**: alles unter `/opt/paperless/consume/` wird automatisch eingelesen

---

## 7. Claude Haiku per paperless-ai anbinden

### 7.1 API-Key holen

1. **console.anthropic.com** anmelden (separates Konto zu claude.ai!)
2. **Billing** → Guthaben aufladen (5–10 € reichen lange)
3. **API Keys** → **Create Key** → kopieren

### 7.2 paperless-ai installieren

```bash
mkdir -p /opt/paperless-ai && cd /opt/paperless-ai
nano docker-compose.yml
```

Inhalt:

```yaml
services:
  paperless-ai:
    image: clusterzx/paperless-ai:latest
    container_name: paperless-ai
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - data:/app/data
    environment:
      - PUID=1000
      - PGID=1000

volumes:
  data:
```

Starten:

```bash
docker compose up -d
```

### 7.3 Paperless-API-Token erstellen

In Paperless-Weboberfläche:
1. User-Icon oben rechts → **Mein Profil**
2. **API-Token** → **Token erstellen** → kopieren

### 7.4 paperless-ai konfigurieren

Browser: `http://LXC-IP:3000`

Setup-Wizard:
- **Paperless-ngx URL**: `http://LXC-IP:8000`
- **API-Token**: kopierter Paperless-Token
- **AI Provider**: **Anthropic**
- **API-Key**: Anthropic-Key
- **Model**: `claude-haiku-4-5`
- **Felder zum Setzen**: Titel, Tags, Korrespondent, Dokumenttyp
- **Tag-Filter Ausschluss**: `no-ai` eintragen
- **Scan-Modus**: "Only new documents" für den Anfang

---

## 8. Updates

### 8.1 Paperless-ngx aktualisieren

```bash
cd /opt/paperless
docker compose pull
docker compose up -d
```

`pull` lädt neue Images (falls vorhanden), `up -d` startet nur die geänderten Container neu. Daten in Volumes bleiben erhalten.

### 8.2 paperless-ai aktualisieren

```bash
cd /opt/paperless-ai
docker compose pull
docker compose up -d
```

### 8.3 Debian aktualisieren

Monatlich:

```bash
apt update && apt upgrade -y
```

### 8.4 Alte Images aufräumen

Nach mehreren Updates sammeln sich ungenutzte Images:

```bash
docker image prune -af
```

---

## 9. Backup-Strategie

### 9.1 Das Konzept – warum mehrere Backup-Ebenen?

Eine einzige Backup-Methode deckt nicht alle Szenarien ab. Wir bauen drei sich ergänzende Ebenen:

**Ebene A: Proxmox-Backup des gesamten LXC → NAS**
- Sichert den **kompletten Container** inkl. Betriebssystem, Docker-Volumes, Konfiguration
- **Restore mit wenigen Klicks**: alles wieder wie vorher
- Format: Proxmox-proprietär (`.vma.zst` bzw. `.tar.zst`)
- **Antwort auf "wie sichere ich den gesamten LXC?"**: genau das macht diese Ebene

**Ebene B: Rohe Dokumente als Dateien → NAS**
- Sichert nur die Original-PDFs und Scans
- Dateien liegen als **normale PDFs im Dateisystem**, ohne Paperless lesbar
- Für den Worst Case: Paperless kaputt, Proxmox kaputt – du kannst trotzdem alle Dokumente öffnen
- **Antwort auf "ich will Dateien unabhängig von Paperless öffnen können"**: genau diese Ebene

**Ebene C: Paperless-Export (portabel) → NAS**
- Sichert Dokumente **mit sprechenden Dateinamen** (z.B. "2024-03-15 Vodafone Rechnung.pdf") plus JSON-Manifest mit allen Metadaten (Tags, Korrespondent, Notizen)
- Format: Standard-PDFs + offenes JSON
- Ideal für Migration zu anderem DMS oder Langzeitarchivierung

**Warum alle drei?**
- Proxmox-Backup ist am schnellsten zurückgespielt, aber nur auf Proxmox nutzbar
- Rohe Dokumente sind universell lesbar, aber ohne Metadaten
- Paperless-Export ist universell lesbar **inkl.** Metadaten

Du brauchst nicht alle drei täglich – die Kombination aus verschiedenen Rhythmen macht's.

### 9.2 NAS per SMB im LXC mounten

**Auf dem NAS:** Einen Freigabe-Ordner anlegen, z.B. `paperless-backups`. Dazu einen Benutzer mit Schreibrechten darauf (nicht den Admin!).

**Im LXC:**

```bash
mkdir -p /mnt/nas
```

Credentials-Datei anlegen (damit das Passwort nicht im Klartext in `/etc/fstab` steht):

```bash
nano /root/.smbcreds
```

Inhalt:

```
username=backupuser
password=DEIN_NAS_PASSWORT
```

Rechte beschränken:

```bash
chmod 600 /root/.smbcreds
```

In `/etc/fstab` eintragen für automatisches Mounten beim Boot:

```bash
nano /etc/fstab
```

Zeile anhängen (IP und Sharename anpassen):

```
//NAS-IP/paperless-backups /mnt/nas cifs credentials=/root/.smbcreds,uid=0,gid=0,vers=3.0,iocharset=utf8,_netdev 0 0
```

Mount testen:

```bash
mount -a
ls /mnt/nas
```

Wenn du den Inhalt deines NAS-Shares siehst, passt alles.

**Hinweis zu SMB-Version:** `vers=3.0` funktioniert auf den meisten aktuellen NAS. Bei älteren Synology/QNAP eventuell `vers=2.1`. Bei ganz alten Geräten `vers=1.0`, aber **das ist unsicher** – dann lieber NAS-Firmware aktualisieren.

### 9.3 Ebene A: Proxmox-Backup aufs NAS

Das ist deine Haupt-Sicherung. Einrichten auf Proxmox-Ebene, nicht im LXC.

**Schritt 1: NAS als Storage in Proxmox einbinden**

In der Proxmox-GUI:
1. **Datacenter** (links oben) → **Storage** → **Add** → **SMB/CIFS**
2. Einstellungen:
   - **ID**: `nas-backup` (frei wählbar)
   - **Server**: IP deines NAS
   - **Username**: NAS-Benutzer mit Schreibrechten
   - **Password**: dazugehöriges Passwort
   - **Share**: wähle aus dem Dropdown den Share (z.B. `paperless-backups`)
   - **Content**: **VZDump backup file** anhaken
3. **Add**

**Schritt 2: Backup-Job anlegen**

1. **Datacenter** → **Backup** → **Add**
2. Einstellungen:
   - **Node**: dein Host
   - **Storage**: `nas-backup`
   - **Schedule**: z.B. `02:00` täglich
   - **Selection mode**: Include selected VMs
   - **VMs**: Container `110` (paperless) anhaken
   - **Mode**: **Snapshot** (läuft im Hot-Betrieb, kurze IO-Pause von wenigen Sekunden)
   - **Compression**: **zstd** (schneller als gzip, kleinere Dateien)
3. **Retention** (Aufbewahrung): Am besten auf der Storage-Ebene setzen statt im Job:
   - Keep last: 3 (die letzten 3 Backups)
   - Keep daily: 7 (täglich der letzten 7 Tage)
   - Keep weekly: 4 (wöchentlich der letzten 4 Wochen)
   - Keep monthly: 6 (monatlich der letzten 6 Monate)
4. **Create**

**Was das bewirkt:**
Jede Nacht um 02:00 wird der gesamte LXC "eingefroren" (Snapshot), die Dateien werden aufs NAS kopiert, der Snapshot wird wieder freigegeben. Der Container läuft die ganze Zeit weiter. Im NAS-Share liegen dann Dateien wie `vzdump-lxc-110-2024_10_15-02_00_03.tar.zst`.

### 9.4 Ebene B: Rohe Dokumente als Dateien aufs NAS

Script anlegen:

```bash
mkdir -p /opt/scripts
nano /opt/scripts/sync-documents.sh
```

Inhalt:

```bash
#!/bin/bash
set -e

# Quelle: das Volume, in dem Paperless die Dokumente ablegt
SOURCE="/var/lib/docker/volumes/paperless_media/_data/documents"

# Ziel auf dem NAS
TARGET="/mnt/nas/paperless-documents"

LOG="/var/log/documents-sync.log"

mkdir -p "$TARGET"

echo "=== Sync gestartet: $(date) ===" >> "$LOG"

# rsync:
# -a: archive (Berechtigungen, Zeitstempel, rekursiv)
# -v: verbose (für Log)
# --delete: löscht im Ziel, was in der Quelle nicht mehr existiert
rsync -av --delete "$SOURCE/" "$TARGET/" >> "$LOG" 2>&1

echo "=== Sync fertig: $(date) ===" >> "$LOG"
```

Ausführbar machen:

```bash
chmod +x /opt/scripts/sync-documents.sh
```

**Test:** Einmal manuell ausführen:

```bash
/opt/scripts/sync-documents.sh
```

Auf dem NAS sollten nun unter `paperless-documents/` die Ordner `originals`, `archive` und `thumbnails` liegen. Die PDFs in `originals/` sind **exakt die Dateien, die du hochgeladen hast** – unverändert, mit Paperless-internen Dateinamen wie `0000001.pdf`.

**Cron einrichten** (täglich um 04:00, 2h nach dem Proxmox-Backup):

```bash
crontab -e
```

Zeile anhängen:

```
0 4 * * * /opt/scripts/sync-documents.sh
```

Speichern.

**Warum das sinnvoll ist:** Selbst wenn Paperless, Docker und Proxmox komplett weg sind – solange du an die NAS-Dateien kommst, kannst du jedes Dokument einzeln öffnen. Die Dateinamen sind zwar nur Nummern, aber: wenn du die Metadaten brauchst (wofür ist Dokument 0000017?), greifst du auf Ebene C zurück.

### 9.5 Ebene C: Paperless-Export mit sprechenden Dateinamen

Script anlegen:

```bash
nano /opt/scripts/paperless-export.sh
```

Inhalt:

```bash
#!/bin/bash
set -e

BACKUP_DIR="/mnt/nas/paperless-exports"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
EXPORT_TMP="/tmp/paperless-export-$TIMESTAMP"
KEEP_DAYS=90

mkdir -p "$BACKUP_DIR" "$EXPORT_TMP"

# Export im Container triggern
# --use-filename-format: sprechende Dateinamen ("2024-03-15 Vodafone Rechnung.pdf")
# --delete: vorher Export-Ordner leeren
cd /opt/paperless
docker compose exec -T webserver document_exporter ../export --use-filename-format --delete

# Aus Volume herauskopieren
docker compose cp webserver:/usr/src/paperless/export/. "$EXPORT_TMP/"

# Komprimieren und aufs NAS
tar -czf "$BACKUP_DIR/paperless-export-$TIMESTAMP.tar.gz" -C "$EXPORT_TMP" .
rm -rf "$EXPORT_TMP"

# Alte Exporte löschen (älter als 90 Tage)
find "$BACKUP_DIR" -name "paperless-export-*.tar.gz" -mtime +$KEEP_DAYS -delete

echo "Export fertig: paperless-export-$TIMESTAMP.tar.gz"
```

Ausführbar machen:

```bash
chmod +x /opt/scripts/paperless-export.sh
```

**Cron** (sonntags um 03:00, wöchentlich):

```
0 3 * * 0 /opt/scripts/paperless-export.sh >> /var/log/paperless-export.log 2>&1
```

**Was landet im .tar.gz?**
- Alle Dokumente mit sprechenden Dateinamen
- `manifest.json` mit allen Metadaten (Tags, Korrespondent, Titel, Notizen, Datum)
- Eigene Unterordner pro Korrespondent/Dokumenttyp (je nach Einstellung)

Auspacken geht auf jedem Rechner (Linux, Windows, Mac) ohne Paperless – das ist dein **universelles Archiv-Format**.

### 9.6 Optional: Monats-Archive für Langzeit-Aufbewahrung

Die wöchentlichen Exporte werden nach 90 Tagen gelöscht. Für Langzeit willst du pro Monat einen Export dauerhaft behalten:

```bash
nano /opt/scripts/paperless-archive-monthly.sh
```

```bash
#!/bin/bash
set -e

ARCHIVE_DIR="/mnt/nas/paperless-archive"
MONTH=$(date +%Y-%m)
EXPORT_TMP="/tmp/paperless-archive-$MONTH"

mkdir -p "$ARCHIVE_DIR" "$EXPORT_TMP"

cd /opt/paperless
docker compose exec -T webserver document_exporter ../export --use-filename-format --delete
docker compose cp webserver:/usr/src/paperless/export/. "$EXPORT_TMP/"

tar -czf "$ARCHIVE_DIR/paperless-archive-$MONTH.tar.gz" -C "$EXPORT_TMP" .
rm -rf "$EXPORT_TMP"

echo "Monats-Archiv: paperless-archive-$MONTH.tar.gz"
```

```bash
chmod +x /opt/scripts/paperless-archive-monthly.sh
```

**Cron** (1. jedes Monats, 05:00):

```
0 5 1 * * /opt/scripts/paperless-archive-monthly.sh >> /var/log/paperless-archive.log 2>&1
```

Diese Archive werden **nie automatisch gelöscht**. So sammelst du pro Monat ein Archiv – perfekt fürs Dokumentenarchiv für zehn Jahre und mehr.

### 9.7 Zeitplan-Übersicht

| Was | Wann | Aufbewahrung | Wo |
|---|---|---|---|
| Proxmox-LXC-Backup | täglich 02:00 | 7d + 4w + 6m | NAS via Proxmox |
| Rohe Dokumente (rsync) | täglich 04:00 | spiegelt aktuell | NAS (Live-Kopie) |
| Paperless-Export (rotierend) | sonntags 03:00 | 90 Tage | NAS |
| Monats-Archiv (permanent) | 1. d. Monats 05:00 | für immer | NAS |

### 9.8 Offsite-Backup – das fehlende Glied

Ein NAS zuhause schützt vor Festplattendefekt, aber nicht vor Feuer, Blitzschlag, Diebstahl, Ransomware.

Empfohlen: Auf dem NAS selbst einen Sync zu einem Cloud-Dienst einrichten (meistens per Hersteller-App: Synology Hyper Backup, QNAP Hybrid Backup Sync, TrueNAS Cloud Sync). Günstige Ziele:
- **Hetzner Storage Box**: 1 TB für ~4 €/Monat
- **Backblaze B2**: pay-per-use, günstig bei wenig Daten
- **Wasabi, Idrive e2**: Alternativen

**Wichtig:** Vor dem Upload verschlüsseln (Hyper Backup und Co. bieten das). Keinesfalls unverschlüsselte Dokumente in der Cloud ablegen.

### 9.9 Restore-Test – bitte nicht überspringen!

Einmal im Jahr ausprobieren:
1. In Proxmox einen Test-LXC aus einem Backup restoren
2. Prüfen ob Paperless startet und Dokumente sichtbar sind
3. Zusätzlich: ein Monats-Archiv vom NAS entpacken, prüfen ob PDFs lesbar sind

Ein Backup das nicht getestet wurde, ist kein Backup.

---

## 10. Restore im Notfall

Je nach Problem greifst du auf eine andere Ebene zurück. Hier die Schritte.

### 10.1 LXC komplett kaputt (Szenario: Container startet nicht mehr, Daten korrupt)

**Quelle:** Proxmox-Backup (Ebene A).

**In der Proxmox-GUI:**
1. Links auf deinen NAS-Storage (`nas-backup`) klicken
2. Reiter **Backups**
3. Gewünschtes Backup auswählen (z.B. das letzte von heute nacht)
4. Button **Restore** klicken
5. Optionen:
   - **Override**: den bestehenden kaputten Container ersetzen (oder neue CT-ID wählen für parallel)
   - **Storage**: wohin gerestored werden soll (z.B. `local-lvm`)
6. **Restore** bestätigen

Nach 5–15 Minuten (je nach Größe) ist der LXC wieder wie zum Zeitpunkt des Backups. Einfach starten, fertig – Docker-Container, Daten, alles wieder da.

### 10.2 Einzelnes Dokument versehentlich gelöscht

**Quelle:** rohe Dokumente auf NAS (Ebene B) oder Paperless-Export (Ebene C).

Am schnellsten: Auf dem NAS unter `paperless-documents/originals/` die entsprechende PDF suchen (Dateiname ist die Paperless-Doc-ID). Nochmal in Paperless hochladen.

Alternativ: Aus dem letzten Monats-Archiv entpacken (sprechende Dateinamen, einfacher zu finden).

### 10.3 Paperless-Installation komplett weg, Dokumente wichtig

**Szenario:** Du willst auf ein ganz neues System, oder Paperless existiert nicht mehr.

**Quelle:** Paperless-Export (Ebene C).

Alle PDFs liegen mit sprechenden Dateinamen im Archiv. Kannst du direkt in einen Ordner legen und mit jedem PDF-Reader öffnen. Metadaten stecken in `manifest.json`.

### 10.4 Migration zu neuer Paperless-Instanz (z.B. nach Neuinstallation)

**Quelle:** Paperless-Export (Ebene C).

1. Neue Paperless-Installation wie in Schritt 5 aufsetzen
2. Auf dem NAS den letzten Export entpacken
3. Entpackten Inhalt in den Import-Ordner kopieren:

```bash
# Auf dem LXC, nach frischer Paperless-Installation:
mkdir -p /tmp/paperless-import
cd /tmp/paperless-import
tar -xzf /mnt/nas/paperless-archive/paperless-archive-YYYY-MM.tar.gz

# In den Container kopieren
docker compose cp /tmp/paperless-import/. /opt/paperless/webserver:/usr/src/paperless/export/

# Import triggern
docker compose exec webserver document_importer ../export
```

Nach ein paar Minuten sind alle Dokumente **inkl. Tags, Korrespondenten, Dokumenttypen, Titel** wieder da.

### 10.5 Totaler Verlust (Proxmox-Host kaputt, NAS noch intakt)

1. Neuen Proxmox-Host aufsetzen
2. NAS als Storage einbinden (siehe 9.3)
3. Backup aus dem NAS-Storage restoren

Alles wieder da, meist innerhalb einer Stunde.

### 10.6 Totaler Verlust inkl. NAS (schlimmster Fall)

**Quelle:** Offsite-Backup.

Du brauchst eine Offsite-Kopie (siehe 9.8). Ohne die: Daten sind weg. Deshalb **nicht am Offsite sparen**.

---

## 11. FAQ und Troubleshooting

### OCR in mehreren Sprachen

```
PAPERLESS_OCR_LANGUAGE=deu+eng
PAPERLESS_OCR_LANGUAGES=deu eng
```

Singular mit `+` = OCR-Sprachen, Plural mit Leerzeichen = zu installierende Pakete.

### Welche Dateiformate werden unterstützt?

Mit Tika + Gotenberg:
- Bilder: PNG, JPG, TIFF
- PDF
- Office: .docx, .doc, .xlsx, .xls, .pptx, .ppt, .odt, .ods
- E-Mail: .eml, .msg
- HTML

### Tika/Gotenberg funktionieren nicht

Prüfen ob alle Container laufen:

```bash
cd /opt/paperless
docker compose ps
```

Sollten 5 Services sein (broker, db, webserver, gotenberg, tika), alle "Up".

Logs von Tika/Gotenberg prüfen:

```bash
docker compose logs tika
docker compose logs gotenberg
```

Integration prüfen:

```bash
docker compose exec webserver env | grep -i tika
```

Sollte zeigen:
```
PAPERLESS_TIKA_ENABLED=1
PAPERLESS_TIKA_GOTENBERG_ENDPOINT=http://gotenberg:3000
PAPERLESS_TIKA_ENDPOINT=http://tika:9998
```

### Secret-Key vergessen

Neuen setzen, Container neu starten (`docker compose up -d`). User müssen sich neu einloggen, Daten bleiben.

### Admin-Passwort vergessen

```bash
cd /opt/paperless
docker compose exec webserver python manage.py changepassword admin
```

### Admin wurde nicht erstellt

`PAPERLESS_ADMIN_USER` und `PAPERLESS_ADMIN_PASSWORD` greifen nur beim allerersten Start:

```bash
docker compose exec webserver python manage.py createsuperuser
```

### SMB-Mount funktioniert nicht

- SMB-Version anpassen (`vers=3.0` → `2.1` → `1.0`)
- Credentials-Datei-Rechte prüfen (`chmod 600`)
- NAS-Benutzer hat Schreibrechte?
- Firewall am NAS erlaubt SMB von LXC-IP?

Test manuell:

```bash
mount -t cifs //NAS-IP/sharename /mnt/nas -o credentials=/root/.smbcreds,vers=3.0
```

### paperless-ai findet Paperless nicht

- IP statt `localhost` verwenden
- Port 8000 erreichbar? `curl http://LXC-IP:8000` aus dem LXC testen
- API-Token korrekt? Neuen erstellen und testen

### Logs einsehen

```bash
cd /opt/paperless
docker compose logs -f webserver
```

### Eingebautes ML vs. Claude

Das eingebaute ML halluziniert nicht, wählt nur aus bestehenden Tags. Claude kann neue generieren und Titel erfinden. Kombination: eingebautes ML für Klassifikation, Claude für Titel/Zusatz.

### Sensible Dokumente von Claude ausschließen

Beim Upload Tag `no-ai` vergeben. paperless-ai überspringt diese.

### Kostenrahmen Claude Haiku

Bei 500 Dokumenten/Jahr typischerweise unter 5 €. Im Anthropic-Dashboard monatliches Limit setzbar.

### Dokumenten-Struktur auf der Festplatte

Paperless speichert in `/var/lib/docker/volumes/paperless_media/_data/documents/`:
- `originals/` – unveränderte Original-PDFs/Bilder, Dateiname = `<Doc-ID>.pdf`
- `archive/` – OCR-durchsuchbare PDF-Versionen
- `thumbnails/` – Vorschaubilder

Die Originale in `originals/` sind deine Dokumente 1:1 wie hochgeladen – ohne Paperless mit jedem PDF-Reader öffenbar.

---

## URLs zum Merken

- Paperless-ngx: `http://LXC-IP:8000`
- paperless-ai: `http://LXC-IP:3000`
- Anthropic Console: `https://console.anthropic.com`

## Wichtige Pfade im LXC

- Paperless Compose: `/opt/paperless/`
- paperless-ai Compose: `/opt/paperless-ai/`
- Backup-Scripts: `/opt/scripts/`
- NAS-Mount: `/mnt/nas/`
- Docker-Volumes: `/var/lib/docker/volumes/`
- Consume-Ordner (Auto-Import): `/opt/paperless/consume/`

## NAS-Struktur (nach dem Setup)

Auf deinem NAS-Share liegen dann:

```
paperless-backups/
├── dump/                              (Proxmox-LXC-Backups)
│   ├── vzdump-lxc-110-2024_10_14-02_00_03.tar.zst
│   └── ...
├── paperless-documents/               (tägliche 1:1-Spiegelung)
│   ├── originals/
│   ├── archive/
│   └── thumbnails/
├── paperless-exports/                 (wöchentlich, 90 Tage)
│   ├── paperless-export-20241013-030001.tar.gz
│   └── ...
└── paperless-archive/                 (monatlich, dauerhaft)
    ├── paperless-archive-2024-10.tar.gz
    ├── paperless-archive-2024-11.tar.gz
    └── ...
```
