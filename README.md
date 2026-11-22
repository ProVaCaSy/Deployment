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

> **Achtung:** Docker Compose liest eine `.env` im Ordner `Deployment` automatisch mit. Liegt dort eine `.env` für die Test-VM, bekommt die lokale Datenbank deren Passwort und das Backend kann sich nicht mehr anmelden (siehe [Häufige Fehler](#häufige-fehler)). Für die lokale Entwicklung braucht es keine `.env`. Ist trotzdem eine vorhanden, unter Linux und macOS so starten:
>
> ```bash
> docker compose --env-file /dev/null -f docker-compose.dev.yml up -d
> ```
>
> Unter Windows die `.env` vorher umbenennen.

### 3. Backend starten

```bash
cd ../Backend
./mvnw spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

Unter Windows (PowerShell):

```powershell
cd ..\Backend
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.arguments=--server.port=8081"
```

Flyway legt beim Start die Tabellen an. Das Backend läuft lokal auf Port 8081, weil Keycloak schon Port 8080 belegt. Die endgültige Portverteilung ist noch offen.

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

Das Passwort ist jeweils der Benutzername.

| Benutzer | Gruppe |
|----------|--------|
| prof | professors |
| staff | staff |
| dekanat | dekanat |
| student | students (keine Rolle, sieht nur die Landing Page) |
| realm-admin | verwaltet nur den Realm hse unter /admin |

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

### 1. .env anlegen

```bash
cd Deployment
cp .env.example .env
```

In der `.env` alle Werte setzen. Die Passwörter dürfen nicht `change-me` bleiben, die Domains ohne `http://` und ohne Port eintragen. Fehlt ein Pflichtwert, bricht Compose mit einer Meldung ab.

### 2. Lokal testen ohne echte Domains

Für einen Test auf dem eigenen Rechner reichen Test-Domains, die auf den eigenen Rechner zeigen. Dazu in die hosts-Datei eintragen (Linux und macOS: `/etc/hosts`, Windows: `C:\Windows\System32\drivers\etc\hosts`, Editor als Administrator starten):

```
127.0.0.1 app.test api.test auth.test
```

Und in der `.env`:

```
APP_DOMAIN=app.test
API_DOMAIN=api.test
AUTH_DOMAIN=auth.test
```

### 3. Starten

Beide Compose-Dateien nutzen dieselben Volumes. Lief vorher die Dev-Umgebung, diese zuerst mit `down -v` beenden. Das löscht die lokalen Datenbanken, sonst passen Passwörter und Realm nicht.

```bash
docker compose -f docker-compose.dev.yml down -v
docker compose pull
docker compose up -d
```

### 4. Prüfen

Keycloak braucht etwa eine Minute. Bis dahin antwortet nginx mit `502`.

```bash
docker compose ps
curl -i -H "Host: api.test" http://localhost:8080/                      # Backend: 401 ist richtig
curl -i -H "Host: auth.test" http://localhost:8080/realms/hse/account   # Keycloak: 302 oder Seite
docker compose exec nginx cat /etc/nginx/conf.d/default.conf           # Sind die Domains eingesetzt?
```

Im Browser: http://app.test:8080 und http://auth.test:8080/realms/hse/account

### MailHog auf der VM

Die Weboberfläche ist nur auf der VM selbst erreichbar. Von außen geht es über einen SSH-Tunnel:

```bash
ssh -L 8025:localhost:8025 <benutzer>@<vm-adresse>
```

Danach im eigenen Browser http://localhost:8025 öffnen.

## Images aus GitHub Packages

`docker compose pull` lädt `ghcr.io/provacasy/backend` und `ghcr.io/provacasy/frontend`. Die Pipelines bauen sie nach jedem Merge auf `main` mit den Tags `latest` und der Commit-SHA. Eine bestimmte Version läuft mit `IMAGE_TAG` in der `.env`.

Meldet `docker compose pull` den Fehler `denied` oder `unauthorized`, ist das Paket privat. Dann mit einem eigenen GitHub Personal Access Token (Scope `read:packages`) anmelden und den Token als Passwort eingeben:

```bash
docker login ghcr.io -u <github-benutzername>
```

Den Token nie ins Repository oder in die `.env` schreiben.

## Häufige Fehler

| Meldung | Ursache | Lösung |
|---------|---------|--------|
| `password authentication failed for user "provacasy"` | Die Datenbank wurde mit einem anderen Passwort angelegt, zum Beispiel aus der `.env` oder von der Test-VM-Variante. PostgreSQL setzt das Passwort nur beim ersten Start. | `docker compose -f docker-compose.dev.yml down -v`, danach ohne `.env` starten (siehe oben). Löscht die lokalen Daten. |
| `Port 8080 was already in use` beim Backend | Keycloak belegt lokal schon Port 8080. | Backend mit Port 8081 starten (siehe Schritt 3). |
| `502 Bad Gateway` von nginx | Keycloak oder Backend starten noch. | Etwa eine Minute warten, `docker compose logs -f keycloak` ansehen. |
| Jede Anfrage landet beim Frontend | Der Host passt zu keiner Domain, dann antwortet der erste Server-Block in nginx. | Bei `curl` den Header `-H "Host: api.test"` mitgeben. Domains in der `.env` prüfen, nach einer Änderung speichern und `docker compose up -d` erneut ausführen. |
| Änderungen an `hse-realm.json` wirken nicht | Der Realm wird nur importiert, wenn er noch nicht existiert. | `down -v` und neu starten. Löscht die Daten. |
| Backend antwortet mit `401`, obwohl man eingeloggt ist | Der Aussteller (`iss`) im Token passt nicht zur `issuer-uri` im Backend. | `AUTH_DOMAIN` und `HTTP_PORT` in der `.env` prüfen. |
| `denied` oder `unauthorized` bei `docker compose pull` | Das Paket in GitHub Packages ist privat. | `docker login ghcr.io` (siehe oben). |
| `manifest unknown` oder `not found` bei `docker compose pull` | Das Image wurde noch nie gebaut. | Pipeline auf `main` abwarten oder im jeweiligen Repo lokal bauen, zum Beispiel `docker build -t ghcr.io/provacasy/frontend:latest .` und dann `docker compose up -d` ohne `pull`. |
| `permission denied` bei `/var/run/docker.sock` (Linux) | Die Sitzung kennt die Gruppe `docker` noch nicht. | Neu anmelden oder `newgrp docker`. |
| `./mvnw verify` findet keine Datenbank | Die Dev-Umgebung läuft nicht. | Zuerst `docker compose -f docker-compose.dev.yml up -d`. |
| `release version 21 not supported` | Falsche Java-Version aktiv. | `java -version` prüfen und Java 21 aktivieren. |

## Hinweise

- Die Zugangsdaten und Demo-Benutzer gelten nur für die lokale Entwicklung.
- Die `.env` (Vorlage: `.env.example`) ist für die Test-VM gedacht und kommt nicht ins Repository. Lokal wird keine `.env` gebraucht.
- `V1__init.sql` im Backend nach dem Merge nicht mehr ändern. Schemaänderungen kommen in eine neue Datei `V2__...sql`.
