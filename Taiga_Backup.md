## 1. Sicherung auf dem bestehenden Taiga-Server

### 1.1 Backup-Verzeichnis anlegen

Auf dem Taiga-Server (SSH, Shell im Home-Verzeichnis des Users):

```bash
cd ~
BACKUP_DIR="taiga-backup-$(date +%F_%H%M)"
mkdir -p "$BACKUP_DIR"
echo "$BACKUP_DIR"
```

Der ausgegebene Name (z. B. `taiga-backup-2025-11-20_2142`) wird später benötigt.

---

### 1.2 PostgreSQL-Dump der Taiga-Datenbank erstellen

```bash
cd ~/taiga-docker

docker compose exec -T taiga-db \
  pg_dump -U taiga taiga | gzip > ~/"$BACKUP_DIR"/taiga-db.sql.gz
```

Ergebnis prüfen:

```bash
ls -lh ~/"$BACKUP_DIR"/taiga-db.sql.gz
```

---

### 1.3 Media-Dateien (Uploads, Avatare, Wiki-Bilder usw.) sichern

```bash
cd ~/taiga-docker

docker compose exec taiga-back bash -lc \
  'cd /taiga-back && tar czf - media' \
  > ~/"$BACKUP_DIR"/taiga-media.tar.gz
```

Optional zusätzlich Static-Files:

```bash
docker compose exec taiga-back bash -lc \
  'cd /taiga-back && tar czf - static' \
  > ~/"$BACKUP_DIR"/taiga-static.tar.gz
```

---

### 1.4 Konfiguration sichern

Im Home-Verzeichnis:

```bash
cd ~

# Taiga-Docker-Konfiguration
cp taiga-docker/.env "$BACKUP_DIR"/taiga-dotenv.backup
cp taiga-docker/docker-compose.yml "$BACKUP_DIR"/
cp taiga-docker/docker-compose-inits.yml "$BACKUP_DIR"/ 2>/dev/null || true
cp taiga-docker/docker-compose*.yml "$BACKUP_DIR"/ 2>/dev/null || true
```

Falls eine Nginx-Site für Taiga existiert:

```bash
sudo cp /etc/nginx/sites-available/taiga.conf ~/"$BACKUP_DIR"/nginx-taiga.conf
```

---

### 1.5 Backup-Struktur prüfen

```bash
cd ~
ls -lh "$BACKUP_DIR"
```

Typische Inhalte:

* `taiga-db.sql.gz`
* `taiga-media.tar.gz`
* evtl. `taiga-static.tar.gz`
* `taiga-dotenv.backup`
* `docker-compose*.yml`
* `nginx-taiga.conf`

---

## 2. Backup vom Server auf lokalen Windows-Rechner herunterladen

Voraussetzungen:

* Zugriff per SSH (Passwort oder Key) auf den Server
* `pscp.exe` (PuTTY SCP) auf Windows installiert
* Pfad von `pscp.exe` bekannt (z. B. `C:\Program Files\PuTTY\pscp.exe`)

### 2.1 Absoluten Pfad des Backup-Verzeichnisses feststellen

Auf dem Server:

```bash
cd ~
pwd          # z.B. /home/lohdens
ls -d taiga-backup*
```

Ergebnis z. B.:

* Home: `/home/lohdens`
* Backup-Ordner: `/home/lohdens/taiga-backup-2025-11-20_2142`

---

### 2.2 Download mit `pscp` (Windows, PowerShell oder CMD)

In PowerShell:

```powershell
cd C:\Users\lukas\Downloads
```

#### Variante A: Login mit Passwort

```powershell
pscp -r lohdens@apm.hs-emden-leer.de:/home/lohdens/taiga-backup-2025-11-20_2142 .
```

* `-r` kopiert rekursiv (Ordner)
* Der Punkt `.` am Ende steht für das aktuelle lokale Verzeichnis.

Nach Eingabe des Linux-Passworts wird der Ordner `taiga-backup-2025-11-20_2142` im lokalen Download-Verzeichnis angelegt.

#### Variante B: Login mit SSH-Key (PPK wie in PuTTY)

```powershell
pscp -i C:\Pfad\zu\deinem_key.ppk -r lohdens@apm.hs-emden-leer.de:/home/lohdens/taiga-backup-2025-11-20_2142 .
```

Pfad zur `.ppk`-Datei entsprechend anpassen.

---

## 3. Wiederherstellung auf einem neuen („frischen“) Taiga-Server

Ziel: Vollständige Wiederherstellung der Taiga-Instanz mit allen Projekten und Dateien, sodass das Projekt „APM Demo Project – Real-Life Pacman Robot“ wieder normal im Taiga-Frontend erscheint.

### 3.1 Backupdateien auf den neuen Server hochladen

Auf dem neuen Server sollte ein Benutzer (z. B. wieder `lohdens`) mit Home-Verzeichnis existieren.

Auf dem Windows-Rechner im Verzeichnis mit dem Backup-Ordner:

```powershell
cd C:\Users\lukas\Downloads

# Beispiel: Upload des gesamten Backup-Ordners auf den neuen Server
pscp -r taiga-backup-2025-11-20_2142 lohdens@NEUER_SERVER:/home/lohdens/
# oder mit Key:
# pscp -i C:\Pfad\zu\key.ppk -r taiga-backup-2025-11-20_2142 lohdens@NEUER_SERVER:/home/lohdens/
```

Auf dem neuen Server prüfen:

```bash
ssh lohdens@NEUER_SERVER
cd ~
ls -d taiga-backup*
```

---

### 3.2 `taiga-docker` auf dem neuen Server einrichten

1. Repository klonen:

   ```bash
   cd ~
   git clone https://github.com/taigaio/taiga-docker.git
   cd taiga-docker
   git checkout stable
   ```

2. `.env` aus dem Backup übernehmen:

   ```bash
   cp ~/taiga-backup-2025-11-20_2142/taiga-dotenv.backup .env
   ```

3. In `.env` Domain/URLs bei Bedarf anpassen:

   * `TAIGA_SCHEME` (`http`/`https`)
   * `TAIGA_DOMAIN` (z. B. `apm.hs-emden-leer.de`)
   * `SUBPATH`
   * `WEBSOCKETS_SCHEME`

   Bearbeitung z. B. mit `nano .env`.

SECRET_KEY und DB-Parameter sollten aus dem Backup übernommen werden, damit die Instanz konsistent bleibt.

---

### 3.3 Taiga-Datenbank auf dem neuen Server initial starten

Nur die Datenbank zunächst starten:

```bash
cd ~/taiga-docker
docker compose up -d taiga-db
```

Warten, bis `taiga-db` läuft:

```bash
docker compose ps
```

---

### 3.4 DB-Dump in neue Datenbank einspielen

Annahme: Backup liegt unter `~/taiga-backup-2025-11-20_2142/taiga-db.sql.gz`.

```bash
cd ~/taiga-docker

gunzip -c ~/taiga-backup-2025-11-20_2142/taiga-db.sql.gz \
  | docker compose exec -T taiga-db psql -U taiga taiga
```

Nach Abschluss des Imports ist die Datenbank (inkl. aller Projekte, Nutzer, Einstellungen) auf dem neuen Server vorhanden.

Optional kann danach ein einmaliger Migrationslauf erfolgen, insbesondere wenn die Taiga-Version auf dem neuen Server leicht neuer ist:

```bash
./taiga-manage.sh migrate
```

---

### 3.5 Media-Dateien auf dem neuen Server wiederherstellen

Zunächst den Stack (mindestens `taiga-back`) starten:

```bash
cd ~/taiga-docker
docker compose up -d
```

Danach die Media-Archive in den `taiga-back`-Container entpacken:

```bash
docker compose exec -T taiga-back bash -lc 'cd /taiga-back && tar xzf -' \
  < ~/taiga-backup-2025-11-20_2142/taiga-media.tar.gz
```

Falls `taiga-static.tar.gz` gesichert wurde:

```bash
docker compose exec -T taiga-back bash -lc 'cd /taiga-back && tar xzf -' \
  < ~/taiga-backup-2025-11-20_2142/taiga-static.tar.gz
```

Damit liegen alle Uploads (Attachments, Avatare, Wiki-Bilder usw.) wieder am erwarteten Ort.

---

### 3.6 Stack vollständig starten und prüfen

```bash
cd ~/taiga-docker
docker compose up -d
docker compose ps
```

Bei Verwendung eines Host-Nginx (wie in der vorherigen Einrichtung):

* Nginx-Konfiguration aus dem Backup übernehmen (optional anpassen):

  ```bash
  sudo cp ~/taiga-backup-2025-11-20_2142/nginx-taiga.conf /etc/nginx/sites-available/taiga.conf
  sudo ln -sf /etc/nginx/sites-available/taiga.conf /etc/nginx/sites-enabled/taiga.conf
  sudo nginx -t
  sudo systemctl reload nginx
  ```

Anschließend sollte der Zugriff über den konfigurierten Hostnamen (z. B. `http://apm.hs-emden-leer.de`) möglich sein.

---

### 3.7 Sichtbarkeit des Projekts prüfen

Nach erfolgreichem Start:

* Anmeldung über die bereits im Backup enthaltenen Benutzerkonten (Passwörter wurden mit der DB übernommen).
* Das Projekt **„APM Demo Project – Real-Life Pacman Robot“** sollte in der Projektliste erscheinen.
* Inhalte (User Stories, Tasks, Wiki, Anhänge) werden aus der wiederhergestellten DB und den Media-Dateien bedient.

Falls die Anmeldung scheitert (z. B. Passwort unklar), kann das Passwort eines bekannten Users direkt auf dem Server neu gesetzt werden:

```bash
cd ~/taiga-docker
./taiga-manage.sh changepassword BENUTZERNAME
```