# ProVaCaSy Deployment

Dieses Repository enthält alles, was für den Betrieb von (Pro)VaCaSy nötig ist: die Docker Compose Dateien, die nginx Konfiguration als Reverse Proxy und die Keycloak Einstellungen mit dem Realm hse. Backend und Frontend liegen in eigenen Repositories, ihre Images kommen aus GitHub Packages.

Entstanden im Rahmen des Software-Projekts (Wintersemester 2026/27) an der Hochschule Esslingen.

## Lokal entwickeln

### Voraussetzungen

- Docker mit Docker Compose
- Java 21
- Node.js 24 (oder 22.18 und neuer)

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

### 3. Backend starten

```bash
cd Backend
./mvnw spring-boot:run
```

Flyway legt beim Start die Tabellen an.

### 4. Frontend starten

```bash
cd Frontend
npm install
npm run dev
```

## Adressen

| Dienst | Adresse |
|--------|---------|
| Frontend | http://localhost:5173 |
| Keycloak Admin | http://localhost:8080/admin (admin / admin) |
| MailHog (Test-Mails) | http://localhost:8025 |
| PostgreSQL der Anwendung | localhost:5432 (Datenbank, Benutzer und Passwort: provacasy) |

## Demo-Benutzer

Das Passwort ist jeweils der Benutzername.

| Benutzer | Gruppe |
|----------|--------|
| prof | professors |
| staff | staff |
| dekanat | dekanat |

## Nützliche Befehle

```bash
docker compose -f docker-compose.dev.yml ps                 # Status
docker compose -f docker-compose.dev.yml logs -f keycloak   # Logs
docker compose -f docker-compose.dev.yml down               # Stoppen, Daten bleiben
docker compose -f docker-compose.dev.yml down -v            # Stoppen und alles zurücksetzen
```

Nach `down -v` ist die Datenbank leer und der Realm wird neu importiert.

## Hinweise

- Die Zugangsdaten und Demo-Benutzer gelten nur für die lokale Entwicklung.
- Eigene Passwörter gehören in eine `.env` (Vorlage: `.env.example`). Die `.env` kommt nicht ins Repository.
- `V1__init.sql` im Backend nach dem Merge nicht mehr ändern. Schemaänderungen kommen in eine neue Datei `V2__...sql`.
