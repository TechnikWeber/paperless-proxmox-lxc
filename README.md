# Paperless-ngx auf Proxmox mit Claude Haiku

Komplette, reproduzierbare Anleitung zum Aufsetzen von Paperless-ngx in einem **privilegierten** LXC-Container auf Proxmox mit Docker Compose, Tika/Gotenberg für erweiterte Dateiformat-Unterstützung, KI-Anbindung via Claude Haiku, und einer mehrstufigen Backup-Strategie auf ein Synology-NAS via SMB.

## Mein konkretes Setup

Diese Anleitung basiert auf folgender Umgebung – IPs und Namen entsprechend anpassen, falls sich etwas ändert.

| Komponente | Wert |
|---|---|
| Proxmox-Host | MiniPC |
| NAS | Synology, IP `192.168.178.3` |
| SMB-Share auf NAS | `Proxmox` (**großes P**, case-sensitive!) |
| NAS-User für Backups | `proxmox` |
| LXC-ID | `201` |
| LXC-IP | `192.168.178.7` |
| LXC-Hostname | `paperless` |
| Gateway | `192.168.178.1` |
| LXC-Typ | **privileged** (nicht unprivileged!) |
| LXC-Template | `debian-13-standard` |

## Versionsübersicht

| Komponente | Version |
|---|---|
| Docker | neuestes stabiles Release aus dem offiziellen Docker-Repo |
| PostgreSQL | **17** (nicht 18!) |
| Redis | 8 |
| Paperless-ngx | `ghcr.io/paperless-ngx/paperless-ngx:latest` |
| Gotenberg | 7.10 |
| Apache Tika | latest |
| paperless-ai | `clusterzx/paperless-ai:latest` |
| Claude-Modell | `claude-haiku-4-5` |

## Inhaltsverzeichnis

1. [Überblick und Entscheidungen](#1-überblick-und-entscheidungen)
2. [LXC in Proxmox erstellen (privileged!)](#2-lxc-in-proxmox-erstellen-privileged)
3. [Docker installieren](#3-docker-installieren)
4. [NAS per SMB mounten (mit Retry-Logik)](#4-nas-per-smb-mounten-mit-retry-logik)
5. [Paperless-ngx mit Tika und Gotenberg installieren](#5-paperless-ngx-mit-tika-und-gotenberg-installieren)
6. [Grundeinrichtung in Paperless](#6-grundeinrichtung-in-paperless)
7. [Scanner per SMB auf den Consume-Ordner einrichten](#7-scanner-per-smb-auf-den-consume-ordner-einrichten)
8. [Claude Haiku per paperless-ai anbinden](#8-claude-haiku-per-paperless-ai-anbinden)
9. [Updates](#9-updates)
10. [Backup-Strategie](#10-backup-strategie)
11. [Restore im Notfall](#11-restore-im-notfall)
12. [FAQ und Troubleshooting](#12-faq-und-troubleshooting)

---

## 1. Überblick und Entscheidungen

### Warum privileged LXC statt unprivileged?

**Kurze Antwort:** Weil unprivileged LXCs beim SMB-Mount mit "Operation not permitted" scheitern, selbst mit aufgebohrten apparmor-Capabilities. Der privilegierte LXC erlaubt Mount-Operationen direkt.

**Trade-off:** Weniger Isolation vom Host. Für ein Homelab mit eigenen Diensten ist das gängige Praxis und akzeptabel. Nicht geeignet für "fremden" Code oder Multi-Tenant-Szenarien.

**Alternative wäre gewesen:** Mount auf dem Proxmox-Host mit Bind-Mount in den LXC. Funktioniert zuverlässig, war aber hier aus Risikogründen nicht gewünscht.

### Warum Docker Compose ohne Portainer?

- **Weniger Abhängigkeiten**: Paperless hängt nicht von einer zusätzlichen Verwaltungs-Software ab
- **Compose-Files sind versionierbar**: `docker-compose.yml` und `.env` lassen sich in Git legen
- **Einfacher zu debuggen**: Keine Zwischenschicht, direkter Zugriff auf Docker
- **Keine "Limited Stack"-Probleme** wie mit Portainer bei extern erstellten Stacks

### Warum PostgreSQL 17 statt 18?

PostgreSQL 18 hat einen Breaking Change bei der Verzeichnisstruktur (Mount muss auf `/var/lib/postgresql` statt `/var/lib/postgresql/data`). Version 17 funktioniert mit dem Standard-Layout out-of-the-box und ist bis Ende 2029 supportet.

### Warum Tika und Gotenberg?

**Paperless-ngx Basis** kann nur PDF, PNG, JPG, TIFF.

**Mit Tika + Gotenberg** zusätzlich:
- Word (.docx, .doc), Excel (.xlsx), PowerPoint (.pptx)
- OpenOffice/LibreOffice (.odt, .ods)
- E-Mails (.eml, .msg)
- HTML-Dateien

**Tika** extrahiert Text und Metadaten, **Gotenberg** wandelt Office-Dateien in PDF um. Zusammen ca. 300–500 MB zusätzlicher RAM-Bedarf.

### Warum systemd-Service statt fstab für SMB-Mount?

fstab-Mounts scheitern in LXCs häufig beim Boot mit "Network is unreachable", weil der Mount versucht wird, bevor das Netzwerk vollständig steht. Ein systemd-Service mit Retry-Logik wartet bis zu 5 Minuten auf das NAS und probiert mehrfach – robust gegen Netzwerk-Timing und langsam hochfahrende NAS.

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

## 2. LXC in Proxmox erstellen (privileged!)

### 2.1 Debian 13 Template herunterladen

Am Proxmox-Host:

```bash
pveam update
```

In der GUI:
1. Storage `local` anklicken
2. Reiter **CT Templates**
3. **Templates** → `debian-13-standard` → **Download**
4. Warten bis "TASK OK"

### 2.2 Container erstellen

**Create CT** (rechts oben):

**General:**
- CT ID: `201`
- Hostname: `paperless`
- Password: sicheres Root-Passwort
- ✗ **Unprivileged container NICHT anhaken** (= privileged)

**Template:**
- Storage: `local`
- Template: `debian-13-standard_...`

**Disks:**
- Storage: `local-lvm`
- Disk size: **32 GB**

**CPU:**
- Cores: **2**

**Memory:**
- Memory: **3072** MB
- Swap: **512** MB

**Network:**
- Bridge: `vmbr0`
- IPv4: **statisch** → `192.168.178.7/24`
- Gateway: `192.168.178.1`
- Firewall: ✓

**DNS:**
- Leer lassen (übernimmt Host-Einstellungen)

**Confirm:**
- ✗ **"Start after created" NICHT anhaken**
- **Finish**

### 2.3 Features ergänzen

Auf dem Proxmox-Host:

```bash
nano /etc/pve/lxc/201.conf
```

Unter der `hostname`-Zeile einfügen:

```
features: keyctl=1,nesting=1
```

Die Config sollte dann so aussehen:

```
arch: amd64
cores: 2
features: keyctl=1,nesting=1
hostname: paperless
memory: 3072
net0: name=eth0,bridge=vmbr0,firewall=1,gw=192.168.178.1,hwaddr=...,ip=192.168.178.7/24,type=veth
ostype: debian
rootfs: local-lvm:vm-201-disk-0,size=32G
swap: 512
```

**Wichtig:** Keine Zeile `unprivileged: 1` – fehlt diese Zeile, ist der Container privileged.

Speichern.

### 2.4 Container starten

```bash
pct start 201
pct enter 201
```

---

## 3. Docker installieren

Im Container:

```bash
apt update && apt upgrade -y
apt install -y ca-certificates curl gnupg nano cron rsync cifs-utils smbclient
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

### DNS-Probleme beim Image-Download

Falls beim Image-Pull (später in Schritt 5) Fehler wie `i/o timeout` beim DNS-Lookup kommen:

```bash
nano /etc/resolv.conf
```

Inhalt:

```
nameserver 1.1.1.1
nameserver 9.9.9.9
```

Speichern. Cloudflare und Quad9 sind deutlich zuverlässiger als der Router.

---

## 4. NAS per SMB mounten (mit Retry-Logik)

### 4.1 Share-Name auf dem NAS prüfen

**Wichtig:** SMB-Share-Namen sind case-sensitive. Zuerst checken, wie dein Share genau heißt:

```bash
smbclient -L //192.168.178.3 -U proxmox
```

Passwort eingeben. Du siehst eine Liste aller Shares. Der Name (hier: `Proxmox` mit großem P) muss **exakt** so in der Mount-Config verwendet werden.

### 4.2 Mount-Script mit Retry-Logik (Credentials inline)

fstab funktioniert in LXCs nicht zuverlässig beim Boot ("Network is unreachable"), weil das Netzwerk zum Mount-Zeitpunkt noch nicht vollständig steht. Stattdessen ein systemd-Service mit Retry-Script.

**Warum Credentials inline und nicht in separater `.smbcreds`-Datei:** Auf Debian 13 mit aktuellen `cifs-utils` gibt es einen Regressionsbug – Credentials-Dateien werden nicht korrekt eingelesen, selbst wenn sie syntaktisch einwandfrei sind (sauberes UTF-8, Unix-Zeilenumbrüche, richtige Rechte). Das Resultat ist `STATUS_LOGON_FAILURE` bzw. "cannot mount read-only". Der Mount mit Credentials direkt in den Mount-Optionen funktioniert dagegen zuverlässig. Daher schreiben wir Username und Passwort direkt ins Script und schützen das Script mit `chmod 700`.

```bash
mkdir -p /mnt/nas /opt/scripts
nano /opt/scripts/mount-nas.sh
```

Inhalt:

```bash
#!/bin/bash
# Wartet auf Netzwerk/NAS und mountet NAS mit Retry
# Gibt dem NAS bis zu 5 Minuten Zeit zum Hochfahren
# Credentials inline wegen cifs-utils-Bug mit separaten Credentials-Dateien

MOUNT_POINT="/mnt/nas"
MAX_TRIES=60        # 60 Versuche
SLEEP=5             # alle 5 Sekunden = max. 5 Minuten

mkdir -p "$MOUNT_POINT"

for i in $(seq 1 $MAX_TRIES); do
    if mountpoint -q "$MOUNT_POINT"; then
        echo "Bereits gemountet."
        exit 0
    fi
    
    if mount -t cifs //192.168.178.3/Proxmox "$MOUNT_POINT" \
        -o username=proxmox,password=DEIN_NAS_PASSWORT,uid=0,gid=0,vers=3.0,iocharset=utf8 2>/dev/null; then
        echo "Mount erfolgreich nach $i Versuchen ($(($i * $SLEEP)) Sekunden)."
        exit 0
    fi
    
    echo "Versuch $i/$MAX_TRIES fehlgeschlagen, warte ${SLEEP}s..."
    sleep $SLEEP
done

echo "Mount nach $(($MAX_TRIES * $SLEEP)) Sekunden fehlgeschlagen."
exit 1
```

Speichern. **`DEIN_NAS_PASSWORT` durch das echte NAS-Passwort ersetzen.**

**Wichtig zum Passwort:** Verwende **nur Buchstaben und Zahlen** (keine Sonderzeichen wie `!`, `$`, `#`, `\`, `` ` ``, Leerzeichen). Das erspart dir an mehreren Stellen Probleme – sowohl hier als auch bei der Proxmox-GUI-Einbindung des NAS-Storage.

### 4.3 Script-Rechte korrekt setzen

**Wichtig:** Weil das Passwort im Klartext im Script steht, muss es vor fremdem Zugriff geschützt sein. Gleichzeitig muss systemd es ausführen können. Richtige Rechte: **700** (Lesen+Schreiben+Ausführen nur für root):

```bash
chmod 700 /opt/scripts/mount-nas.sh
```

**Falle:** Wenn du versehentlich `chmod 600` setzt (nur Lesen+Schreiben), kann systemd das Script nicht mehr ausführen – du bekommst dann den Fehler `Unable to locate executable` bzw. `Failed at step EXEC spawning`. In dem Fall einfach `chmod 700` setzen, dann läuft's wieder.

### 4.4 systemd-Service anlegen

```bash
nano /etc/systemd/system/nas-mount.service
```

Inhalt:

```ini
[Unit]
Description=NAS SMB-Share mounten mit Retry
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/opt/scripts/mount-nas.sh
TimeoutStartSec=400
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Speichern.

**`TimeoutStartSec=400`** gibt systemd 6:40 Min zum Durchlaufen, das Script selbst bricht nach 5 Min ab.

### 4.5 Aktivieren und testen

```bash
systemctl daemon-reload
systemctl enable --now nas-mount.service
systemctl status nas-mount.service
ls /mnt/nas
```

Status sollte `active (exited)` zeigen, `ls` den NAS-Inhalt.

### 4.6 Reboot-Test (optional)

```bash
reboot
```

Nach dem Boot einloggen:

```bash
ls /mnt/nas
systemctl status nas-mount.service
```

Logs einsehen falls nötig:

```bash
journalctl -u nas-mount.service -f
```

Falls der Mount doch mal fehlschlagen sollte, manuell nachstarten:

```bash
systemctl restart nas-mount.service
```

---

## 5. Paperless-ngx mit Tika und Gotenberg installieren

### 5.1 Verzeichnis und Compose-Datei erstellen

```bash
mkdir -p /opt/paperless && cd /opt/paperless
nano docker-compose.yml
```

Folgenden Inhalt komplett einfügen:

```yaml
services:
  broker:
    image: docker.io/library/redis:8
    restart: unless-stopped
    volumes:
      - redisdata:/data

  db:
    image: docker.io/library/postgres:17
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
      PAPERLESS_DBNAME: paperless
      PAPERLESS_DBUSER: paperless
      PAPERLESS_DBPASS: paperless
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

**Wichtige Erklärungen:**

- **PostgreSQL 17**: Version 18 hat eine geänderte Verzeichnisstruktur, die zum Crash führt. Version 17 funktioniert out-of-the-box.
- **DB-Credentials**: Der `db`-Service erstellt Datenbank+User, der `webserver` kennt sie über `PAPERLESS_DBNAME`/`DBUSER`/`DBPASS`. Beide Blöcke müssen zusammenpassen.
- **Tika/Gotenberg-Aktivierung**: Die drei `PAPERLESS_TIKA_*`-Variablen schalten die Integration in Paperless ein. Ohne diese würden die Container zwar laufen, Paperless würde sie aber ignorieren.

### 5.2 Environment-Datei erstellen

```bash
nano docker-compose.env
```

Inhalt:

```
PAPERLESS_OCR_LANGUAGE=deu+eng
PAPERLESS_OCR_LANGUAGES=deu eng
PAPERLESS_TIME_ZONE=Europe/Berlin
PAPERLESS_SECRET_KEY=HIER_LANGEN_ZUFALLSSTRING_EINFUEGEN
PAPERLESS_URL=http://192.168.178.7:8000
PAPERLESS_ADMIN_USER=admin
PAPERLESS_ADMIN_PASSWORD=DeinPasswortHier
```

**Wichtige Hinweise zur Syntax:**
- Keine spitzen Klammern `< >` – direkt den Wert hinter `=`
- Keine Anführungszeichen
- Keine Leerzeichen um das `=`
- Beim Passwort die Sonderzeichen `$`, `` ` ``, `\`, `"`, `'` vermeiden

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

Erster Start dauert 5–10 Minuten (Images laden, DB initialisieren). Bei DNS-Timeouts einfach `docker compose up -d` nochmal ausführen – bereits geladene Layer werden nicht neu gezogen.

Status prüfen:

```bash
docker compose ps
```

Du solltest 5 Container sehen, alle mit Status "Up":
- `paperless-broker-1`
- `paperless-db-1`
- `paperless-gotenberg-1`
- `paperless-tika-1`
- `paperless-webserver-1` (nach ca. 1 Minute "healthy")

### 5.4 Tika/Gotenberg-Integration prüfen

```bash
docker compose exec webserver env | grep -i tika
```

Sollte zeigen:
```
PAPERLESS_TIKA_ENABLED=1
PAPERLESS_TIKA_GOTENBERG_ENDPOINT=http://gotenberg:3000
PAPERLESS_TIKA_ENDPOINT=http://tika:9998
```

Ab jetzt werden automatisch auch .docx, .xlsx, .eml etc. beim Upload verarbeitet.

### 5.5 Paperless aufrufen

Browser: `http://192.168.178.7:8000`

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
- **Scan direkt vom Scanner** → siehe Abschnitt 7

---

## 7. Scanner per SMB auf den Consume-Ordner einrichten

Damit ein Netzwerk-Scanner (oder das Handy, ein anderer PC etc.) direkt in den Consume-Ordner speichern kann, richten wir den Ordner als SMB-Share im LXC ein. Alles was dort landet, zieht Paperless automatisch ein, verarbeitet es und löscht die Datei aus dem Consume-Ordner.

### 7.1 Samba-Server im LXC installieren

```bash
apt install -y samba
```

### 7.2 Samba-User für den Scanner anlegen

```bash
# Systembenutzer ohne Login-Shell
useradd -M -s /usr/sbin/nologin scanner

# Samba-Passwort setzen (wird fürs Netzwerk verwendet)
smbpasswd -a scanner
```

Passwort zweimal eingeben. **Nur Buchstaben und Zahlen verwenden** – Sonderzeichen machen an Scannern oft Probleme. Passwort merken, brauchst du später beim Scanner-Setup.

### 7.3 Berechtigungen am Consume-Ordner

```bash
mkdir -p /opt/paperless/consume
chown scanner:scanner /opt/paperless/consume
chmod 770 /opt/paperless/consume
```

### 7.4 Samba-Share konfigurieren

```bash
nano /etc/samba/smb.conf
```

Ganz ans Ende der Datei anhängen:

```ini
[paperless-consume]
   comment = Paperless Consume-Ordner
   path = /opt/paperless/consume
   browseable = yes
   writable = yes
   valid users = scanner
   create mask = 0660
   directory mask = 0770
   force user = scanner
   force group = scanner
```

Speichern.

### 7.5 Samba aktivieren

```bash
systemctl restart smbd nmbd
systemctl enable smbd nmbd
```

### 7.6 Test vom PC aus

Bevor du den Scanner anfasst, teste den Share von einem Rechner im Netzwerk:

**Windows:** Im Explorer in die Adresszeile:
```
\\192.168.178.7\paperless-consume
```
Anmelden mit `scanner` + Passwort.

**Mac:** Finder → "Gehe zu" → "Mit Server verbinden" → `smb://192.168.178.7/paperless-consume`

**Linux:** Dateimanager → `smb://192.168.178.7/paperless-consume`

Zieh ein Test-PDF rein und schau in der Paperless-Weboberfläche, ob es nach 10–60 Sekunden auftaucht.

### 7.7 Scanner konfigurieren

Im Scanner-Menü nach **"Scan to SMB"**, **"Scan to Network Folder"**, **"Netzwerkordner"** o.ä. suchen.

| Feld | Wert |
|---|---|
| Name / Label | `Paperless` |
| Host / Server | `192.168.178.7` |
| Share / Ordnerpfad | `paperless-consume` |
| Pfad auf Server | leer oder `/` |
| Benutzername | `scanner` |
| Passwort | dein gesetztes SMB-Passwort |
| Domain | leer oder `WORKGROUP` |
| Port | `445` |
| SMB-Version | **2.1 oder 3.0** |

**Wichtig zur SMB-Version:** Viele ältere Scanner nutzen standardmäßig SMB 1.0, das wir aus Sicherheitsgründen nicht konfiguriert haben. Am Scanner-Webinterface unter **Netzwerk** → **SMB** / **Protokolle** auf `2.1` oder `3.0` stellen. Bei Scannern der letzten 5 Jahre ist das meist Default.

**Falls dein Scanner SMB 2/3 nicht kann** (Notlösung, nur im internen Netz):

```bash
nano /etc/samba/smb.conf
```

Im `[global]`-Bereich ergänzen:
```ini
server min protocol = NT1
client min protocol = NT1
```

Dann:
```bash
systemctl restart smbd
```

### 7.8 Scan-Einstellungen am Scanner

Für beste OCR-Ergebnisse in Paperless:

| Einstellung | Empfehlung | Warum |
|---|---|---|
| **Dateiformat** | **PDF** (oder PDF/A) | Multi-Page, OCR-freundlich |
| **Auflösung** | **300 DPI** | Optimum für OCR und Dateigröße |
| **Farbmodus** | **Graustufen** für Text, Farbe für Fotos | Graustufen spart enorm Platz |
| **Kompression** | Standard / Mittel | |
| **OCR am Scanner** | **AUS** | Paperless macht's selbst besser |
| **Dateiname** | automatisch mit Datum/Zeit | Paperless benennt neu |

**Wichtig:** OCR am Scanner ausschalten ("durchsuchbares PDF" etc.). Paperless' Tesseract-OCR ist konsistenter und indexiert sauber.

### 7.9 Test-Scan

1. Testdokument in den Scanner
2. Scanner → Paperless-Profil → Start
3. Im LXC beobachten:

```bash
ls /opt/paperless/consume/
```

Die Datei taucht kurz auf und verschwindet dann (Paperless hat sie eingezogen).

4. In der Paperless-Weboberfläche unter **Dokumente** das neue Dokument suchen. Import dauert 10–60 Sekunden je nach Seitenanzahl.

### 7.10 Hersteller-Hinweise

- **Brother**: "Scan to Network" im Webinterface. Brother-Scanner lieben SMB 1.0 – eventuell Notlösung oben nötig.
- **HP**: "ScanToFolder" oder "Scan Jobs". Nutzen meist SMB 2.1.
- **Canon imageRUNNER**: Adressbuch → Neues Ziel → SMB.
- **Epson WorkForce**: Scan-Einstellungen → Netzwerkordner/FTP. Mögen SMB 3.0.

---

## 8. Claude Haiku per paperless-ai anbinden

### 8.1 API-Key holen

1. **console.anthropic.com** anmelden (separates Konto zu claude.ai!)
2. **Billing** → Guthaben aufladen (5–10 € reichen lange)
3. **API Keys** → **Create Key** → kopieren

### 8.2 paperless-ai installieren

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

### 8.3 Paperless-API-Token erstellen

In Paperless-Weboberfläche:
1. User-Icon oben rechts → **Mein Profil**
2. **API-Token** → **Token erstellen** → kopieren

### 8.4 paperless-ai konfigurieren

Browser: `http://192.168.178.7:3000`

Setup-Wizard:
- **Paperless-ngx URL**: `http://192.168.178.7:8000`
- **API-Token**: kopierter Paperless-Token
- **AI Provider**: **Anthropic**
- **API-Key**: Anthropic-Key
- **Model**: `claude-haiku-4-5`
- **Felder zum Setzen**: Titel, Tags, Korrespondent, Dokumenttyp
- **Tag-Filter Ausschluss**: `no-ai` eintragen
- **Scan-Modus**: "Only new documents" für den Anfang

---

## 9. Updates

### 9.1 Paperless-ngx aktualisieren

```bash
cd /opt/paperless
docker compose pull
docker compose up -d
```

### 9.2 paperless-ai aktualisieren

```bash
cd /opt/paperless-ai
docker compose pull
docker compose up -d
```

### 9.3 Debian aktualisieren

Monatlich:

```bash
apt update && apt upgrade -y
```

### 9.4 Alte Images aufräumen

```bash
docker image prune -af
```

### 9.5 PostgreSQL-Major-Upgrade (später mal)

PostgreSQL 17 auf 18 (oder höher) ist ein **Major-Upgrade** und erfordert `pg_upgrade`. Vorher vollständiges Proxmox-Backup + Paperless-Export anlegen. Solange PostgreSQL 17 supportet ist (Ende 2029), besteht kein Upgrade-Druck.

---

## 10. Backup-Strategie

### 10.1 Voraussetzung: Zeitzone prüfen

Damit die Cron-Jobs zur richtigen Uhrzeit laufen, muss die Zeitzone im LXC stimmen:

```bash
timedatectl
```

Ausgabe sollte enthalten:
```
Time zone: Europe/Berlin (CEST, +0200)
System clock synchronized: yes
```

Falls die Zeitzone nicht `Europe/Berlin` ist:

```bash
timedatectl set-timezone Europe/Berlin
```

Cron nutzt automatisch die System-Zeitzone. Check ob der Cron-Dienst läuft:

```bash
systemctl status cron
```

Sollte `active (running)` zeigen. Falls nicht:

```bash
systemctl enable --now cron
```

### 10.2 Das Konzept – drei Ebenen

**Ebene A: Proxmox-Backup des gesamten LXC → NAS**
- Kompletter Container inkl. OS, Docker-Volumes, Konfiguration
- **Restore mit wenigen Klicks**
- Format: Proxmox-proprietär (`.tar.zst`)

**Ebene B: Rohe Dokumente als Dateien → NAS**
- Original-PDFs als normale Dateien, ohne Paperless lesbar
- Für den Worst Case: alles kaputt, PDFs bleiben zugänglich

**Ebene C: Paperless-Export (portabel) → NAS**
- PDFs mit sprechenden Dateinamen ("2024-03-15 Vodafone Rechnung.pdf") + JSON-Manifest mit allen Metadaten
- Universell lesbar, migrationsfähig

### 10.3 Ebene A: Proxmox-Backup aufs NAS

**Schritt 1: NAS als Storage in Proxmox einbinden**

In der Proxmox-GUI:
1. **Datacenter** → **Storage** → **Add** → **SMB/CIFS**
2. Einstellungen:
   - **ID**: `nas-backup`
   - **Server**: `192.168.178.3`
   - **Username**: `proxmox`
   - **Password**: dein NAS-Passwort
   - **Share**: aus Dropdown `Proxmox` auswählen
   - **Content**: **VZDump backup file** anhaken
3. **Add**

**Schritt 2: Backup-Job anlegen**

1. **Datacenter** → **Backup** → **Add**
2. Einstellungen:
   - **Node**: dein Host
   - **Storage**: `nas-backup`
   - **Schedule**: `02:00` täglich
   - **Selection mode**: Include selected VMs
   - **VMs**: Container `201`
   - **Mode**: **Snapshot**
   - **Compression**: **zstd**
3. **Retention**:
   - Keep last: 3
   - Keep daily: 7
   - Keep weekly: 4
   - Keep monthly: 6
4. **Create**

### 10.4 Ebene B: Rohe Dokumente als Dateien aufs NAS

```bash
nano /opt/scripts/sync-documents.sh
```

Inhalt:

```bash
#!/bin/bash
set -e

# Prüfen ob NAS-Mount aktiv ist
if ! mountpoint -q /mnt/nas; then
    echo "FEHLER: /mnt/nas ist nicht gemountet. Abbruch."
    exit 1
fi

SOURCE="/var/lib/docker/volumes/paperless_media/_data/documents"
TARGET="/mnt/nas/paperless-documents"
LOG="/var/log/documents-sync.log"

mkdir -p "$TARGET"

echo "=== Sync gestartet: $(date) ===" >> "$LOG"
rsync -av --delete "$SOURCE/" "$TARGET/" >> "$LOG" 2>&1
echo "=== Sync fertig: $(date) ===" >> "$LOG"
```

```bash
chmod +x /opt/scripts/sync-documents.sh
```

Manuell testen:

```bash
/opt/scripts/sync-documents.sh
ls /mnt/nas/paperless-documents/
```

Du solltest `originals`, `archive`, `thumbnails` sehen. PDFs in `originals/` sind unveränderte Originale.

Cron einrichten (täglich 04:00):

```bash
crontab -e
```

Zeile:

```
0 4 * * * /opt/scripts/sync-documents.sh
```

### 10.5 Ebene C: Paperless-Export mit sprechenden Dateinamen

```bash
nano /opt/scripts/paperless-export.sh
```

Inhalt:

```bash
#!/bin/bash
set -e

# Prüfen ob NAS-Mount aktiv ist
if ! mountpoint -q /mnt/nas; then
    echo "FEHLER: /mnt/nas ist nicht gemountet. Abbruch."
    exit 1
fi

BACKUP_DIR="/mnt/nas/paperless-exports"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
EXPORT_TMP="/tmp/paperless-export-$TIMESTAMP"
KEEP_DAYS=90

mkdir -p "$BACKUP_DIR" "$EXPORT_TMP"

cd /opt/paperless
docker compose exec -T webserver document_exporter ../export --use-filename-format --delete
docker compose cp webserver:/usr/src/paperless/export/. "$EXPORT_TMP/"

tar -czf "$BACKUP_DIR/paperless-export-$TIMESTAMP.tar.gz" -C "$EXPORT_TMP" .
rm -rf "$EXPORT_TMP"

find "$BACKUP_DIR" -name "paperless-export-*.tar.gz" -mtime +$KEEP_DAYS -delete

echo "Export fertig: paperless-export-$TIMESTAMP.tar.gz"
```

```bash
chmod +x /opt/scripts/paperless-export.sh
```

Cron (sonntags 03:00):

```
0 3 * * 0 /opt/scripts/paperless-export.sh >> /var/log/paperless-export.log 2>&1
```

### 10.6 Monats-Archive für Langzeit-Aufbewahrung

```bash
nano /opt/scripts/paperless-archive-monthly.sh
```

```bash
#!/bin/bash
set -e

if ! mountpoint -q /mnt/nas; then
    echo "FEHLER: /mnt/nas ist nicht gemountet. Abbruch."
    exit 1
fi

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

Cron (1. jedes Monats, 05:00):

```
0 5 1 * * /opt/scripts/paperless-archive-monthly.sh >> /var/log/paperless-archive.log 2>&1
```

Diese Archive werden **nie automatisch gelöscht**.

### 10.7 Zeitplan-Übersicht

| Was | Wann | Aufbewahrung | Wo |
|---|---|---|---|
| Proxmox-LXC-Backup | täglich 02:00 | 7d + 4w + 6m | NAS via Proxmox |
| Rohe Dokumente (rsync) | täglich 04:00 | spiegelt aktuell | NAS (Live-Kopie) |
| Paperless-Export (rotierend) | sonntags 03:00 | 90 Tage | NAS |
| Monats-Archiv (permanent) | 1. d. Monats 05:00 | für immer | NAS |

### 10.8 Backups manuell ausführen

Alle Backup-Arten lassen sich jederzeit auch **manuell antriggern** – z.B. vor einem Update, nach einer großen Import-Aktion, oder zum Testen der Scripts.

**Ebene A (Proxmox-LXC-Backup):**

In der Proxmox-GUI:
1. Container `201` markieren
2. Links im Container-Menü → **Backup**
3. Oben **Backup now** klicken
4. Storage: `nas-backup` auswählen
5. Mode: **Snapshot**
6. **Backup** klicken

Der Job läuft sofort, Fortschritt im Log-Fenster sichtbar.

**Ebene B (rohe Dokumente rsync):**

```bash
/opt/scripts/sync-documents.sh
```

Ausgabe zeigt kopierte Dateien. Danach prüfen:

```bash
ls /mnt/nas/paperless-documents/
```

**Ebene C (Paperless-Export rotierend):**

```bash
/opt/scripts/paperless-export.sh
```

Ausgabe endet mit `Export fertig: paperless-export-YYYYMMDD-HHMMSS.tar.gz`. Prüfen:

```bash
ls -lh /mnt/nas/paperless-exports/
```

**Monats-Archiv (permanent):**

```bash
/opt/scripts/paperless-archive-monthly.sh
```

Ausgabe endet mit `Monats-Archiv: paperless-archive-YYYY-MM.tar.gz`. Prüfen:

```bash
ls -lh /mnt/nas/paperless-archive/
```

**Alle Scripts prüfen den NAS-Mount vorher** – wenn `/mnt/nas` nicht gemountet ist, brechen sie sauber mit Fehlermeldung ab statt Daten ins Leere zu schreiben.

### 10.9 Offsite-Backup

Ein NAS zuhause schützt vor Festplattendefekt, aber nicht vor Feuer, Blitzschlag, Diebstahl, Ransomware.

Empfohlen: Auf dem NAS selbst einen Sync zu einem Cloud-Dienst einrichten (Synology Hyper Backup). Günstige Ziele:
- **Hetzner Storage Box**: 1 TB für ~4 €/Monat
- **Backblaze B2**: pay-per-use

**Wichtig:** Vor dem Upload verschlüsseln.

### 10.10 Restore-Test – Pflicht!

Einmal im Jahr:
1. In Proxmox Test-LXC aus Backup restoren
2. Prüfen ob Paperless startet, Dokumente sichtbar sind
3. Ein Monats-Archiv vom NAS entpacken, prüfen ob PDFs lesbar sind

Backups die nicht getestet wurden, sind keine Backups.

---

## 11. Restore im Notfall

### 11.1 LXC komplett kaputt

Proxmox-GUI → NAS-Storage (`nas-backup`) → Reiter **Backups** → Backup auswählen → **Restore**. Nach 5–15 Minuten ist der LXC wieder da.

### 11.2 Einzelnes Dokument versehentlich gelöscht

NAS → `paperless-documents/originals/` → PDF suchen → in Paperless neu hochladen.

Alternativ: Monats-Archiv entpacken (sprechende Dateinamen).

### 11.3 Paperless komplett weg, Dokumente wichtig

Paperless-Exports aus dem NAS – PDFs haben sprechende Namen, mit jedem Reader öffenbar. Metadaten in `manifest.json`.

### 11.4 Migration zu neuer Paperless-Instanz

1. Neue Paperless-Installation wie in Abschnitt 5 aufsetzen
2. Letzten Export entpacken:

```bash
mkdir -p /tmp/paperless-import
cd /tmp/paperless-import
tar -xzf /mnt/nas/paperless-archive/paperless-archive-YYYY-MM.tar.gz

cd /opt/paperless
docker compose cp /tmp/paperless-import/. webserver:/usr/src/paperless/export/
docker compose exec webserver document_importer ../export
```

Alle Dokumente **inkl. Tags, Korrespondenten, Titel** sind wieder da.

### 11.5 Totaler Verlust (Proxmox kaputt, NAS intakt)

1. Neuen Proxmox-Host aufsetzen
2. NAS als Storage einbinden
3. Backup restoren

### 11.6 Totaler Verlust inkl. NAS

Offsite-Backup ist die einzige Rettung. **Nicht am Offsite sparen.**

---

## 12. FAQ und Troubleshooting

### NAS-Mount schlägt fehl

**Manuell testen:**

```bash
systemctl status nas-mount.service
systemctl restart nas-mount.service
journalctl -u nas-mount.service -n 50
```

**Sharename-Check:**

```bash
smbclient -L //192.168.178.3 -U proxmox
```

SMB-Shares sind case-sensitive. `Proxmox` ≠ `proxmox`.

**Falls Mount grundsätzlich "Operation not permitted":**

Container ist versehentlich unprivileged. Prüfen:

```bash
# Auf dem Proxmox-Host:
grep unprivileged /etc/pve/lxc/201.conf
```

### Service-Fehler "Unable to locate executable" / "Failed at step EXEC"

Das Script hat keine Ausführungsrechte. Prüfen:

```bash
ls -la /opt/scripts/mount-nas.sh
```

Sollte ausführbar sein (`-rwx`). Fix:

```bash
chmod +x /opt/scripts/mount-nas.sh
systemctl restart nas-mount.service
```

### Mount scheitert mit "STATUS_LOGON_FAILURE" / "cannot mount read-only"

In `dmesg | tail -20` sichtbar als:
```
CIFS: Status code returned 0xc000006d STATUS_LOGON_FAILURE
```

Häufige Ursachen:

1. **Regressionsbug in cifs-utils auf Debian 13**: Credentials-Dateien (`/root/.smbcreds`) funktionieren nicht zuverlässig, selbst bei syntaktisch einwandfreiem Inhalt. **Lösung:** Credentials direkt im Script verwenden (siehe Abschnitt 4.2). Das ist bereits der Standardweg dieser Anleitung.
2. **Sonderzeichen im Passwort** (`!`, `$`, `` ` ``, `\`, `#`, Leerzeichen): Führen an vielen Stellen zu Problemen (Bash-Interpretation, cifs-Parser, Proxmox-GUI). **Lösung:** NAS-Passwort auf reine Buchstaben/Zahlen setzen.
3. **Falsches Passwort**: Mit `smbclient -L //192.168.178.3 -U proxmox` testen. Wenn das klappt, ist das Passwort korrekt.
4. **Schreibrechte fehlen am NAS**: Am NAS dem User `proxmox` Lese- und Schreibrechte auf den Share geben.
5. **IP-Beschränkung am NAS**: Falls User nur von bestimmter IP (z.B. Proxmox-Host) erlaubt ist, muss die LXC-IP (192.168.178.7) auch freigegeben werden.

**Diagnose-Befehle:**

```bash
# Passwort gegen NAS verifizieren
smbclient -L //192.168.178.3 -U proxmox

# Manueller Mount mit Credentials inline (ohne Datei)
mkdir -p /tmp/testmount
mount -t cifs //192.168.178.3/Proxmox /tmp/testmount \
  -o username=proxmox,password=DEIN_PASSWORT,uid=0,gid=0,vers=3.0,iocharset=utf8 -v

# Aktuelle Kernel-Logs
dmesg | tail -10
```

Wenn `smbclient` funktioniert und der manuelle Inline-Mount auch, aber der Service nicht: Script-Inhalt prüfen, Rechte müssen `700` sein.

Falls `unprivileged: 1` → Zeile entfernen, Container stoppen und starten.

### DB-Container restartet ständig (PostgreSQL 18-Fehler)

Ursache: PostgreSQL 18 hat die Verzeichnisstruktur geändert. Diese Anleitung nutzt bewusst PostgreSQL 17. Falls doch auf 18 gewechselt:

```bash
cd /opt/paperless
docker compose down
docker volume rm paperless_pgdata paperless_data paperless_media paperless_redisdata
nano docker-compose.yml
```

In der `db`-Section `postgres:18` zu `postgres:17` ändern, speichern, dann:

```bash
docker compose up -d
```

### Image-Pull scheitert mit DNS-Timeout

```bash
nano /etc/resolv.conf
```

```
nameserver 1.1.1.1
nameserver 9.9.9.9
```

Dann `docker compose up -d` nochmal.

### Cron-Jobs laufen nicht zur erwarteten Zeit

Häufige Ursache: falsche Zeitzone. Prüfen:

```bash
timedatectl
```

Sollte `Time zone: Europe/Berlin` zeigen. Falls nicht:

```bash
timedatectl set-timezone Europe/Berlin
```

Cron-Dienst läuft?

```bash
systemctl status cron
```

Aktive Cron-Einträge anzeigen:

```bash
crontab -l
```

Cron-Log einsehen (zeigt wann was ausgeführt wurde):

```bash
grep CRON /var/log/syslog
```

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

```bash
cd /opt/paperless
docker compose ps
docker compose logs tika
docker compose logs gotenberg
docker compose exec webserver env | grep -i tika
```

### Secret-Key vergessen

Neuen setzen, Container neu starten. User müssen sich neu einloggen, Daten bleiben.

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

### paperless-ai findet Paperless nicht

- `192.168.178.7` statt `localhost` verwenden
- `curl http://192.168.178.7:8000` aus dem LXC testen
- API-Token korrekt? Neuen erstellen

### Logs einsehen

```bash
cd /opt/paperless
docker compose logs -f webserver
```

### Eingebautes ML vs. Claude

Eingebautes ML: Klassifikator, halluziniert nicht. Claude: generativ, erzeugt Titel/neue Tags. Kombination = stark.

### Sensible Dokumente von Claude ausschließen

Beim Upload Tag `no-ai` vergeben. paperless-ai überspringt diese.

### Kostenrahmen Claude Haiku

Bei 500 Dokumenten/Jahr typischerweise unter 5 €. Im Anthropic-Dashboard monatliches Limit setzbar.

### Dokumenten-Struktur auf der Festplatte

`/var/lib/docker/volumes/paperless_media/_data/documents/`:
- `originals/` – unveränderte Original-PDFs/Bilder, Dateiname = `<Doc-ID>.pdf`
- `archive/` – OCR-durchsuchbare PDF-Versionen
- `thumbnails/` – Vorschaubilder

---

## URLs zum Merken

- Paperless-ngx: `http://192.168.178.7:8000`
- paperless-ai: `http://192.168.178.7:3000`
- Anthropic Console: `https://console.anthropic.com`

## Wichtige Pfade im LXC

- Paperless Compose: `/opt/paperless/`
- paperless-ai Compose: `/opt/paperless-ai/`
- Backup-Scripts: `/opt/scripts/`
- Mount-Script: `/opt/scripts/mount-nas.sh` (enthält NAS-Credentials inline, daher `chmod 700`)
- systemd-Service: `/etc/systemd/system/nas-mount.service`
- NAS-Mount: `/mnt/nas/`
- Docker-Volumes: `/var/lib/docker/volumes/`
- Consume-Ordner (Auto-Import): `/opt/paperless/consume/`

## NAS-Struktur nach dem Setup

Auf dem NAS im Share `Proxmox`:

```
Proxmox/
├── dump/                              (Proxmox-LXC-Backups)
│   ├── vzdump-lxc-201-YYYY_MM_DD-02_00_03.tar.zst
│   └── ...
├── paperless-documents/               (tägliche 1:1-Spiegelung)
│   ├── originals/
│   ├── archive/
│   └── thumbnails/
├── paperless-exports/                 (wöchentlich, 90 Tage)
│   ├── paperless-export-YYYYMMDD-HHMMSS.tar.gz
│   └── ...
└── paperless-archive/                 (monatlich, dauerhaft)
    ├── paperless-archive-YYYY-MM.tar.gz
    └── ...
```
