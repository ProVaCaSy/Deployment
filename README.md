# ProVaCaSy Deployment

Dieses Repository enthält alles, was für den Betrieb von (Pro)VaCaSy nötig ist: die Docker Compose Dateien, die nginx Konfiguration als Reverse Proxy und die Keycloak Einstellungen mit dem Realm hse. Backend und Frontend liegen in eigenen Repositories, ihre Images kommen aus GitHub Packages.

Entstanden im Rahmen des Software-Projekts (Wintersemester 2026/27) an der Hochschule Esslingen.

## Lokal entwickeln

### Voraussetzungen

- Docker mit Docker Compose
- Java 21
- Node.js 24 (oder 22.18 und neuer)
- Git

Versionen prüfen:

```bash
docker compose version
java -version
node -v
```

### 1. Repositories klonen

In einen gemeinsamen Ordner, zum Beispiel `Project_Github`:

```bash
git clone https://github.com/ProVaCaSy/Backend.git
git clone https://github.com/ProVaCaSy/Frontend.git
git clone https://github.com/ProVaCaSy/Deployment.git
```

### 2. Datenbank, Keycloak und MailHog starten

```bash
cd Deployment
docker compose -f docker-compose.dev.yml up -d
```

Keycloak braucht beim ersten Start etwa eine Minute. Der Realm hse wird dabei automatisch importiert.

> **Hinweis:** Für die lokale Entwicklung wird keine Env-Datei gebraucht, es gelten die Standardwerte. Die Einstellungen für die Test-VM stehen in `.env.bwcloud` und werden nur mit `--env-file` geladen (siehe [Komplette Umgebung im Container](#komplette-umgebung-im-container-test-vm)).

### 3. Backend starten

```bash
cd ../Backend
./mvnw spring-boot:run
```

Unter Windows (PowerShell):

```powershell
cd ..\Backend
.\mvnw.cmd spring-boot:run
```

Flyway legt beim Start die Tabellen an. Das Backend läuft lokal auf Port 8081 (`server.port` in `application.properties`), weil Keycloak Port 8080 belegt.

### 4. Frontend starten

```bash
cd ../Frontend
npm install
npm run dev
```

Unter Windows (PowerShell) ist nur der Pfad anders: `cd ..\Frontend`.

## Adressen

| Dienst | Adresse |
|--------|---------|
| Frontend | http://localhost:5173 |
| Backend | http://localhost:8081 (ohne Token antwortet es mit 401) |
| Keycloak Admin | http://localhost:8080/admin (admin / admin) |
| Keycloak Konto-Seite (Login testen) | http://localhost:8080/realms/hse/account |
| MailHog (Test-Mails) | http://localhost:8025 |
| PostgreSQL der Anwendung | localhost:5432 (Datenbank, Benutzer und Passwort: provacasy) |

## Demo-Benutzer

Das Passwort ist jeweils der Benutzername. Die Benutzer gibt es nur in der lokalen Entwicklung (Realm `keycloak/hse-realm.json`). Neue Benutzer erscheinen erst nach `docker compose -f docker-compose.dev.yml down -v` und neuem Start, weil Keycloak den Realm nur in eine leere Datenbank importiert.

| Benutzer | Name | Gruppe | Rolle in der App |
|----------|------|--------|------------------|
| prof | pro fessor | professors | Professor |
| becker | Anna Becker | professors | Professor |
| weber | Jonas Weber | professors | Professor |
| lschmidt | Lea Schmidt | professors | Professor |
| fischer | Max Fischer | professors | Professor |
| jost | Klaus Günther Jost | professors | Professor |
| staff | sta ff | staff | Sekretariat |
| aschmidt | Anna Schmidt | staff | Sekretariat |
| dekanat | deka nat | dekanat | Dekanat |
| hoffmann | Paul Hoffmann | dekanat | Dekanat |
| richter | Klaus Richter | dekanat und professors | Dekanat (die höchste Rolle zählt) |
| student | stu dent | students | keine Rolle, sieht nur die Landing Page |
| gast | Gast Ohnerolle | keine | keine Rolle, sieht nur die Landing Page |
| realm-admin | realm admin | keine | verwaltet nur den Realm hse unter /admin |

Zum Testen http://localhost:5173 öffnen (Backend und Frontend müssen laufen) und mit einem der Benutzer einloggen, am besten in einem privaten Fenster, damit keine alte Anmeldung stört.

## Nützliche Befehle

```bash
docker compose -f docker-compose.dev.yml ps                 # Status
docker compose -f docker-compose.dev.yml logs -f keycloak   # Logs
docker compose -f docker-compose.dev.yml down               # Stoppen, Daten bleiben
docker compose -f docker-compose.dev.yml down -v            # Stoppen und alles zurücksetzen
```

Nach `down -v` ist die Datenbank leer und der Realm wird neu importiert.

## Komplette Umgebung im Container (Test-VM)

`docker-compose.yml` startet alles als Container, so wie es auf der Test-VM in der BW Cloud läuft: nginx als einziger Eingang, dazu Frontend, Backend, Keycloak, beide Datenbanken und MailHog. nginx leitet anhand der Domain weiter, deshalb braucht jeder Dienst eine eigene Domain.

> **Achtung:** Der Realm in `keycloak/hse-realm.json` enthält die Demo-Benutzer mit Passwort gleich Benutzername. Für eine von außen erreichbare VM ist er nicht geeignet. Ein eigener Realm ohne Demo-Benutzer und HTTPS sind in Arbeit.

### 1. .env.bwcloud anlegen

```bash
cd Deployment
cp .env.example .env.bwcloud
```

In der `.env.bwcloud` alle Werte setzen. Die Passwörter dürfen nicht `change-me` bleiben, die Domains ohne `http://` und ohne Port eintragen.

Docker Compose liest diese Datei nicht automatisch. Deshalb braucht jeder `docker compose` Befehl für die Test-VM `--env-file .env.bwcloud`. Fehlt die Angabe oder ein Wert, bricht Compose mit der Meldung `... fehlt, in .env.bwcloud setzen und mit --env-file .env.bwcloud starten` ab.

### 2. Lokal testen ohne echte Domains

Für einen Test auf dem eigenen Rechner reichen Test-Domains, die auf den eigenen Rechner zeigen. Dazu in die hosts-Datei eintragen (Linux und macOS: `/etc/hosts`, Windows: `C:\Windows\System32\drivers\etc\hosts`, Editor als Administrator starten):

```
127.0.0.1 app.test api.test auth.test
```

Und in der `.env.bwcloud`:

```
APP_DOMAIN=app.test
API_DOMAIN=api.test
AUTH_DOMAIN=auth.test
```

### 3. Starten

Beide Compose-Dateien nutzen dieselben Volumes. Lief vorher die Dev-Umgebung, diese zuerst mit `down -v` beenden. Das löscht die lokalen Datenbanken, sonst passen Passwörter und Realm nicht.

```bash
docker compose -f docker-compose.dev.yml down -v
docker compose --env-file .env.bwcloud pull
docker compose --env-file .env.bwcloud up -d
```

### 4. Prüfen

Keycloak braucht etwa eine Minute. Bis dahin antwortet nginx mit `502`.

```bash
docker compose --env-file .env.bwcloud ps
curl -i -H "Host: api.test" http://localhost:8080/                      # Backend: 401 ist richtig
curl -i -H "Host: auth.test" http://localhost:8080/realms/hse/account   # Keycloak: 302 oder Seite
docker compose --env-file .env.bwcloud exec nginx cat /etc/nginx/conf.d/default.conf   # Sind die Domains eingesetzt?
```

Im Browser: http://app.test:8080 und http://auth.test:8080/realms/hse/account

### MailHog auf der VM

Die Weboberfläche ist nur auf der VM selbst erreichbar. Von außen geht es über einen SSH-Tunnel:

```bash
ssh -L 8025:localhost:8025 <benutzer>@<vm-adresse>
```

Danach im eigenen Browser http://localhost:8025 öffnen.

## Images aus GitHub Packages

`docker compose pull` lädt `ghcr.io/provacasy/backend` und `ghcr.io/provacasy/frontend`. Die Pipelines bauen sie nach jedem Merge auf `main` mit den Tags `latest` und der Commit-SHA. Eine bestimmte Version läuft mit `IMAGE_TAG` in der `.env.bwcloud`.

Meldet `docker compose pull` den Fehler `denied` oder `unauthorized`, ist das Paket privat. Dann mit einem eigenen GitHub Personal Access Token (Scope `read:packages`) anmelden und den Token als Passwort eingeben:

```bash
docker login ghcr.io -u <github-benutzername>
```

Den Token nie ins Repository oder in die `.env.bwcloud` schreiben.

## Häufige Fehler

| Meldung | Ursache | Lösung |
|---------|---------|--------|
| `required variable ... is missing a value` | `--env-file .env.bwcloud` vergessen oder ein Wert fehlt in der Datei. | Befehl mit `--env-file .env.bwcloud` wiederholen und die Werte in der Datei prüfen. |
| `password authentication failed for user "provacasy"` | Die Datenbank wurde mit einem anderen Passwort angelegt, meist weil vorher die Test-VM-Variante lief (beide nutzen dieselben Volumes) oder noch eine alte `.env` im Ordner liegt. PostgreSQL setzt das Passwort nur beim ersten Start. | Eine alte `.env` in `.env.bwcloud` umbenennen, dann `docker compose -f docker-compose.dev.yml down -v` und neu starten. Löscht die lokalen Daten. |
| `Port 8080 was already in use` beim Backend | Keycloak belegt lokal schon Port 8080. | Backend auf dem aktuellen Stand von `main` starten, dort steht `server.port=8081`. Ältere Stände mit `./mvnw spring-boot:run -Dspring-boot.run.arguments=--server.port=8081` starten. |
| `502 Bad Gateway` von nginx | Keycloak oder Backend starten noch. | Etwa eine Minute warten, `docker compose --env-file .env.bwcloud logs -f keycloak` ansehen. |
| Jede Anfrage landet beim Frontend | Der Host passt zu keiner Domain, dann antwortet der erste Server-Block in nginx. | Bei `curl` den Header `-H "Host: api.test"` mitgeben. Domains in der `.env.bwcloud` prüfen, nach einer Änderung speichern und `docker compose --env-file .env.bwcloud up -d` erneut ausführen. |
| Änderungen an `hse-realm.json` wirken nicht | Der Realm wird nur importiert, wenn er noch nicht existiert. | `down -v` und neu starten. Löscht die Daten. |
| Backend antwortet mit `401`, obwohl man eingeloggt ist | Der Aussteller (`iss`) im Token passt nicht zur `issuer-uri` im Backend. | `AUTH_DOMAIN` und `HTTP_PORT` in der `.env.bwcloud` prüfen. |
| `denied` oder `unauthorized` beim Pull | Das Paket in GitHub Packages ist privat. | `docker login ghcr.io` (siehe oben). |
| `manifest unknown` oder `not found` beim Pull | Das Image wurde noch nie gebaut. | Pipeline auf `main` abwarten oder im jeweiligen Repo lokal bauen, zum Beispiel `docker build -t ghcr.io/provacasy/frontend:latest .` und dann `docker compose --env-file .env.bwcloud up -d` ohne `pull`. |
| `permission denied` bei `/var/run/docker.sock` (Linux) | Die Sitzung kennt die Gruppe `docker` noch nicht. | Neu anmelden oder `newgrp docker`. |
| `./mvnw verify` findet keine Datenbank | Die Dev-Umgebung läuft nicht. | Zuerst `docker compose -f docker-compose.dev.yml up -d`. |
| `release version 21 not supported` | Falsche Java-Version aktiv. | `java -version` prüfen und Java 21 aktivieren. |

## Hinweise

- Die Zugangsdaten und Demo-Benutzer gelten nur für die lokale Entwicklung.
- Die `.env.bwcloud` (Vorlage: `.env.example`) ist für die Test-VM gedacht und steht in der `.gitignore`. Lokal wird keine Env-Datei gebraucht.
- Wer noch eine `.env` von früher im Ordner `Deployment` hat, benennt sie um: `mv .env .env.bwcloud` (Windows: `ren .env .env.bwcloud`).
- `V1__init.sql` im Backend nach dem Merge nicht mehr ändern. Schemaänderungen kommen in eine neue Datei `V2__...sql`.
