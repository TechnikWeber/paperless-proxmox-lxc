# Paperless-ngx auf Proxmox mit Docker und Claude Haiku

Komplette Anleitung zum Aufsetzen von Paperless-ngx in einem LXC-Container auf Proxmox, inklusive KI-Anbindung über Claude Haiku via paperless-ai.

## Inhaltsverzeichnis

1. [Überblick und Entscheidungen](#1-überblick-und-entscheidungen)
2. [LXC in Proxmox erstellen](#2-lxc-in-proxmox-erstellen)
3. [Docker installieren](#3-docker-installieren)
4. [Paperless-ngx installieren](#4-paperless-ngx-installieren)
5. [Grundeinrichtung in Paperless](#5-grundeinrichtung-in-paperless)
6. [Claude Haiku per paperless-ai anbinden](#6-claude-haiku-per-paperless-ai-anbinden)
7. [Updates](#7-updates)
8. [Backup und Automatisierung](#8-backup-und-automatisierung)
9. [Restore im Notfall](#9-restore-im-notfall)
10. [FAQ und Troubleshooting](#10-faq-und-troubleshooting)

---

## 1. Überblick und Entscheidungen

### Warum LXC statt VM?

- Leichter, weniger RAM-Overhead, schnellerer Start
- Reicht für Paperless-ngx vollkommen aus
- Docker-in-LXC ist stabil, benötigt aber `nesting=1` und `keyctl=1`

Nachteile von LXC (für diesen Use-Case nicht relevant):
- Docker supportet offiziell nur VMs
- GPU-Passthrough komplizierter (falls später lokales LLM geplant)
- Weniger Isolation vom Host
- Snapshots nur dateisystem-basiert, nicht Live inkl. RAM

### Warum Docker statt nativer Installation?

- Offiziell empfohlener Weg, beste Dokumentation
- Updates trivial: `docker compose pull && docker compose up -d`
- Abhängigkeiten (Redis, PostgreSQL, Tika, Gotenberg) sauber isoliert
- Einfache Backups über Volumes
- Community-Support

### Warum Claude Haiku statt lokalem Ollama?

Lokales Ollama auf schwacher Hardware (z.B. Ryzen 5 3500U mit Vega 8):
- 20 Sekunden bis 3 Minuten pro Dokument
- Häufige Halluzinationen bei kleinen Modellen (3B-8B)
- Mittelmäßige Deutsch-Qualität
- Inkonsistente strukturierte Ausgaben

Claude Haiku 4.5:
- 2-5 Sekunden pro Dokument
- Sehr zuverlässige strukturierte Ausgaben
- Sehr gute Deutsch-Qualität
- Kosten: ~0,001-0,01 € pro Dokument, bei 500 Dokumenten/Jahr unter 5 €

### Datenschutz-Überlegung

Für typische Privatunterlagen (Rechnungen, Verträge, Kontoauszüge) ist Claude Haiku unkritisch. Für sensible Dokumente (Gesundheit, Anwalt, Mandantendaten) empfiehlt sich ein Ausschluss-Tag (`no-ai`) oder manuelle Bearbeitung.

Anthropic speichert API-Inputs standardmäßig nicht zu Trainingszwecken, verwahrt sie aber typischerweise 30 Tage zur Missbrauchserkennung. Server stehen in den USA.

---

## 2. LXC in Proxmox erstellen

### 2.1 Debian 13 Template herunterladen

Am Proxmox-Host (SSH oder Shell):

```bash
pveam update
```

In der GUI:
1. Storage `local` anklicken
2. Reiter **CT Templates** öffnen
3. Button **Templates** klicken
4. `debian-13-standard` suchen → **Download**
5. Warten bis "TASK OK" erscheint

### 2.2 Container erstellen

Rechts oben **Create CT** klicken und durch die Tabs:

**General:**
- Node: dein Proxmox-Host
- CT ID: z.B. `110`
- Hostname: `paperless`
- Password: sicheres Root-Passwort
- ✓ **Unprivileged container**

**Template:**
- Storage: `local`
- Template: `debian-13-standard_...`

**Disks:**
- Storage: dein Hauptstorage (z.B. `local-lvm`)
- Disk size: **20 GB** (bei Bedarf später vergrößerbar)

**CPU:**
- Cores: **2** (4 bei viel paralleler OCR)

**Memory:**
- Memory: **2048** MB
- Swap: **512** MB

**Network:**
- Bridge: `vmbr0`
- IPv4: **DHCP** oder statisch

**DNS:**
- Leer lassen (übernimmt Host-Einstellungen)

**Confirm:**
- ✗ **"Start after created" NICHT anhaken**
- **Finish**

### 2.3 LXC für Docker vorbereiten

In der GUI:
1. Container `110` markieren
2. **Options** → **Features** → **Edit**
3. Haken setzen bei:
   - ✓ **keyctl**
   - ✓ **nesting**
4. **OK**

### 2.4 Container starten

1. Container markieren → **Start**
2. **Console** öffnen
3. Als `root` einloggen

---

## 3. Docker installieren

In der Container-Console:

```bash
apt update && apt upgrade -y
apt install -y ca-certificates curl gnupg nano cron
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian trixie stable" > /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Falls `apt update` einen 404 für `trixie` wirft (Docker hat Debian 13 noch nicht im Repo), `trixie` durch `bookworm` ersetzen – läuft auf Debian 13 problemlos, da binärkompatibel.

Test:

```bash
docker run --rm hello-world
```

"Hello from Docker!" = alles gut.

---

## 4. Paperless-ngx installieren

### 4.1 Compose-Dateien holen

```bash
mkdir -p /opt/paperless && cd /opt/paperless
curl -L https://raw.githubusercontent.com/paperless-ngx/paperless-ngx/main/docker/compose/docker-compose.postgres.yml -o docker-compose.yml
curl -L https://raw.githubusercontent.com/paperless-ngx/paperless-ngx/main/docker/compose/docker-compose.env -o docker-compose.env
```

### 4.2 Umgebungsvariablen setzen

```bash
nano docker-compose.env
```

Folgende Zeilen setzen bzw. anhängen (Raute entfernen, wo vorhanden):

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
- Keine spitzen Klammern `< >` verwenden – direkt den Wert hinter `=` schreiben
- Keine Anführungszeichen
- Keine Leerzeichen um das `=` herum
- Beim Passwort die Sonderzeichen `$`, `` ` ``, `\`, `"`, `'` vermeiden (können in .env-Dateien Probleme machen)
- Einträge die noch nicht in der Datei stehen, einfach unten anhängen

**OCR-Sprachen erklärt:**
- `PAPERLESS_OCR_LANGUAGE` (Singular, mit `+`): welche Sprachen bei der OCR anwenden
- `PAPERLESS_OCR_LANGUAGES` (Plural, mit Leerzeichen): welche Sprachpakete installieren

Wenn nur Deutsch benötigt: `PAPERLESS_OCR_LANGUAGE=deu`. Zwei Sprachen machen OCR ca. 30-50% langsamer.

**Secret-Key generieren:**

```bash
openssl rand -base64 50
```

Der Output (ca. 68 Zeichen) ist der Secret-Key.

Speichern: `Strg+O`, `Enter`, `Strg+X`.

### 4.3 Starten

```bash
docker compose up -d
```

Erster Start dauert einige Minuten (Images laden, DB initialisieren).

Logs ansehen:
```bash
docker compose logs -f
```
`Strg+C` beendet nur die Anzeige, nicht die Container.

### 4.4 IP herausfinden und aufrufen

```bash
ip a | grep inet
```

Browser: `http://LXC-IP:8000`

Login: `admin` + gesetztes Passwort.

**Falls der Admin-User nicht erstellt wurde** (wenn Paperless vor dem Setzen der Variablen schon mal gestartet wurde):

```bash
cd /opt/paperless
docker compose exec webserver python manage.py createsuperuser
```

Oder bestehendes Passwort ändern:
```bash
docker compose exec webserver python manage.py changepassword admin
```

---

## 5. Grundeinrichtung in Paperless

### 5.1 Korrespondenten, Dokumenttypen, Tags anlegen

Sidebar links:
- **Verwaltung → Korrespondenten**: Absender/Empfänger (Bank, Arbeitgeber, Versicherung, Stadtwerke, Vodafone, Finanzamt, …)
- **Verwaltung → Dokumenttypen**: Kategorien (Rechnung, Vertrag, Mahnung, Bescheid, Kontoauszug, …)
- **Verwaltung → Tags**: Schlagworte (wichtig, steuer-2025, no-ai, …)

Beim Anlegen jeweils **Zuordnungsalgorithmus: Auto** wählen, damit das ML später lernt.

**Wichtig:** Tag `no-ai` gleich anlegen – für Dokumente, die nicht an Claude geschickt werden sollen.

### 5.2 Eingebautes ML

Ist automatisch aktiv, sobald Dinge auf "Auto" stehen. Es trainiert sich nachts selbst, sobald genug getaggte Dokumente vorhanden sind (20-30 reichen für erste brauchbare Vorschläge).

Das ML ist ein **Klassifikator**: Es kann nur Kategorien/Korrespondenten/Tags zuweisen, die bereits existieren. Es halluziniert nicht.

Manuell antriggern nach großem Tagging-Schub:

```bash
cd /opt/paperless
docker compose exec webserver document_retagger --use-first --overwrite
```

### 5.3 Dokumente importieren

Drei Wege:
- **Upload-Button** in der Weboberfläche
- **Drag & Drop** ins Dashboard
- **Consume-Ordner**: alles unter `/opt/paperless/consume/` wird automatisch eingelesen

---

## 6. Claude Haiku per paperless-ai anbinden

### 6.1 API-Key holen

1. **console.anthropic.com** anmelden (separates Konto zu claude.ai!)
2. **Billing** → Guthaben aufladen (5-10 € reichen lange)
3. **API Keys** → **Create Key** → kopieren

### 6.2 paperless-ai installieren

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
      - ./data:/app/data
    environment:
      - PUID=1000
      - PGID=1000
```

Starten:

```bash
docker compose up -d
```

### 6.3 Paperless-API-Token erstellen

In Paperless-Weboberfläche:
1. User-Icon oben rechts → **Mein Profil**
2. **API-Token** → **Token erstellen** → kopieren

### 6.4 paperless-ai konfigurieren

Browser: `http://LXC-IP:3000`

Setup-Wizard:
- **Paperless-ngx URL**: `http://LXC-IP:8000`
- **API-Token**: gerade kopierter Paperless-Token
- **AI Provider**: **Anthropic**
- **API-Key**: Anthropic-Key
- **Model**: `claude-haiku-4-5`
- **Felder zum Setzen**: Titel, Tags, Korrespondent, Dokumenttyp
- **Tag-Filter Ausschluss**: `no-ai` eintragen
- **Scan-Modus**: "Only new documents" für den Anfang

---

## 7. Updates

### 7.1 Manuelle Updates

**Paperless-ngx:**
```bash
cd /opt/paperless
docker compose pull
docker compose up -d
```

**paperless-ai:**
```bash
cd /opt/paperless-ai
docker compose pull
docker compose up -d
```

**Debian + Docker selbst:**
```bash
apt update && apt upgrade -y
```

### 7.2 Aufräumen alter Images

```bash
docker image prune -af
```

---

## 8. Backup und Automatisierung

### 8.1 Strategie

Drei Ebenen:
1. **Proxmox-Backup des ganzen LXC** (täglich) – Hauptschutz
2. **Paperless-Export** (wöchentlich) – portabel, migrationsfähig
3. **Auto-Updates mit vorherigem Snapshot** (monatlich, optional)

### 8.2 Proxmox-Backup einrichten (GUI)

1. **Datacenter** → **Backup** → **Add**
2. Einstellungen:
   - **Node**: dein Host
   - **Storage**: Backup-Storage (PBS, NFS, zweite Platte)
   - **Schedule**: `02:00` täglich
   - **Selection mode**: Include selected VMs
   - **VMs**: Container `110` anhaken
   - **Mode**: **Snapshot**
   - **Compression**: zstd
   - **Retention**: z.B. 7 daily, 4 weekly, 6 monthly
3. **Create**

### 8.3 Paperless-Export per Cron

Im LXC:

```bash
mkdir -p /opt/paperless-backups /opt/scripts
nano /opt/scripts/paperless-backup.sh
```

Inhalt:

```bash
#!/bin/bash
set -e

BACKUP_DIR="/opt/paperless-backups"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
EXPORT_DIR="$BACKUP_DIR/export-$TIMESTAMP"
KEEP_DAYS=30

mkdir -p "$EXPORT_DIR"

cd /opt/paperless
docker compose exec -T webserver document_exporter ../export
docker compose cp webserver:/usr/src/paperless/export/. "$EXPORT_DIR/"

tar -czf "$BACKUP_DIR/paperless-$TIMESTAMP.tar.gz" -C "$EXPORT_DIR" .
rm -rf "$EXPORT_DIR"

find "$BACKUP_DIR" -name "paperless-*.tar.gz" -mtime +$KEEP_DAYS -delete

echo "Backup fertig: paperless-$TIMESTAMP.tar.gz"
```

Ausführbar machen und Cron einrichten:

```bash
chmod +x /opt/scripts/paperless-backup.sh
crontab -e
```

Zeile (sonntags 03:00):

```
0 3 * * 0 /opt/scripts/paperless-backup.sh >> /var/log/paperless-backup.log 2>&1
```

### 8.4 Auto-Update mit Snapshot (optional)

**Am Proxmox-Host** – `/usr/local/bin/snapshot-paperless.sh`:

```bash
#!/bin/bash
CTID=110
SNAPNAME="autoupdate-$(date +%Y%m%d)"

# Alte Auto-Snapshots löschen (älter als 14 Tage)
pct listsnapshot $CTID | awk '/^autoupdate-/ {print $1}' | while read snap; do
    snapdate=$(echo "$snap" | sed 's/autoupdate-//')
    if [ $(( ($(date +%s) - $(date -d "$snapdate" +%s)) / 86400 )) -gt 14 ]; then
        pct delsnapshot $CTID "$snap"
    fi
done

pct snapshot $CTID "$SNAPNAME" --description "Auto-Snapshot vor Update"
```

```bash
chmod +x /usr/local/bin/snapshot-paperless.sh
crontab -e
```

Zeile (1. des Monats, 04:00):

```
0 4 1 * * /usr/local/bin/snapshot-paperless.sh
```

**Im Container** – `/opt/scripts/auto-update.sh`:

```bash
#!/bin/bash
set -e

LOG="/var/log/auto-update.log"
echo "=== Update gestartet: $(date) ===" >> "$LOG"

apt update >> "$LOG" 2>&1
apt upgrade -y >> "$LOG" 2>&1

cd /opt/paperless
docker compose pull >> "$LOG" 2>&1
docker compose up -d >> "$LOG" 2>&1

cd /opt/paperless-ai
docker compose pull >> "$LOG" 2>&1
docker compose up -d >> "$LOG" 2>&1

docker image prune -af >> "$LOG" 2>&1

echo "=== Update fertig: $(date) ===" >> "$LOG"
```

```bash
chmod +x /opt/scripts/auto-update.sh
crontab -e
```

Zeile (1. des Monats, 04:30):

```
30 4 1 * * /opt/scripts/auto-update.sh
```

### 8.5 Zeitplan-Übersicht

| Was | Wann | Wo |
|---|---|---|
| Proxmox-Backup LXC | täglich 02:00 | Proxmox-GUI |
| Paperless-Export | sonntags 03:00 | Cron im LXC |
| Snapshot vor Update | 1. d. Monats 04:00 | Cron am Host |
| Auto-Update | 1. d. Monats 04:30 | Cron im LXC |

---

## 9. Restore im Notfall

- **Komplett kaputt**: Proxmox-GUI → Container → **Backup** → Backup wählen → **Restore**
- **Update kaputt**: Proxmox-GUI → Container → **Snapshots** → Auto-Snapshot → **Rollback**
- **Migration / einzelne Dokumente**: tar aus `/opt/paperless-backups/` entpacken, Inhalt in den `import`-Ordner einer neuen Paperless-Installation legen, dann `document_importer` im Container ausführen

---

## 10. FAQ und Troubleshooting

### OCR in mehreren Sprachen

Für Deutsch und Englisch beide Variablen setzen:

```
PAPERLESS_OCR_LANGUAGE=deu+eng
PAPERLESS_OCR_LANGUAGES=deu eng
```

Der Plural-Eintrag sorgt dafür, dass fehlende Sprachpakete automatisch nachinstalliert werden.

### Secret-Key vergessen

Einfach einen neuen setzen und Container neu starten. Alle User müssen sich neu einloggen, Daten bleiben erhalten.

### Admin-Passwort vergessen

```bash
cd /opt/paperless
docker compose exec webserver python manage.py changepassword admin
```

### Admin-User wurde nicht erstellt

`PAPERLESS_ADMIN_USER` und `PAPERLESS_ADMIN_PASSWORD` greifen nur beim allerersten Start. Wenn schon mal gestartet:

```bash
docker compose exec webserver python manage.py createsuperuser
```

### paperless-ai findet Paperless nicht

- `LXC-IP` statt `localhost` verwenden
- Port 8000 von außen erreichbar? `curl http://LXC-IP:8000` aus dem LXC heraus testen
- API-Token korrekt? Neuen erstellen und testen

### Logs einsehen

```bash
cd /opt/paperless
docker compose logs -f webserver
```

### Eingebautes ML vs. Claude

Das eingebaute ML halluziniert nicht, wählt nur aus bestehenden Tags/Korrespondenten. Claude kann neue generieren und Titel erfinden. Sinnvolle Kombination: eingebautes ML für Klassifikation, Claude für Titel und als Ergänzung.

### Sensible Dokumente von Claude ausschließen

Beim Upload Tag `no-ai` vergeben. paperless-ai ist so konfiguriert, dass diese Dokumente übersprungen werden.

### Kostenrahmen Claude Haiku

Bei 500 Dokumenten/Jahr liegt man typischerweise unter 5 €. Im Anthropic-Dashboard lässt sich ein monatliches Limit setzen.

---

## URLs zum Merken

- Paperless-ngx: `http://LXC-IP:8000`
- paperless-ai: `http://LXC-IP:3000`
- Anthropic Console: `https://console.anthropic.com`

## Wichtige Pfade im LXC

- Paperless Compose: `/opt/paperless/`
- Paperless-ai Compose: `/opt/paperless-ai/`
- Backup-Scripts: `/opt/scripts/`
- Backup-Ablage: `/opt/paperless-backups/`
- Consume-Ordner (Auto-Import): `/opt/paperless/consume/`
