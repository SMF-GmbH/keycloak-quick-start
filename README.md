# Keycloak Quick Start mit Docker Compose

Schnell und sicher starten mit [Keycloak](https://www.keycloak.org/) – von `docker run` bis zur
produktionsvorbereiteten Docker-Compose-Umgebung mit PostgreSQL.

> Basiert auf dem Fachbeitrag von Nils Bergmann und Phillip Conrad (SMF, Segment Finance & Public):
> <https://www.smf.de/keycloak-quick-start/>
>
> English version: [README.md](README_EN.md)

Keycloak ist eine Open-Source-Lösung für **Identity & Access Management (IAM)**. Sie ermöglicht die
zentrale Verwaltung von Benutzeranmeldungen, Authentifizierung und Autorisierung und unterstützt
**OAuth 2.0**, **OpenID Connect** und **SAML** – ganz ohne Lizenzkosten.

**Kernfunktionen**

- Benutzerrollen definieren
- Single Sign-On (SSO) implementieren
- Benutzerkonten verwalten
- Admin-Oberfläche und APIs
- Self-Service-Funktionen für Benutzer

---

## Inhaltsverzeichnis

- [Voraussetzungen](#voraussetzungen)
- [Schritt 1: Keycloak mit Docker starten](#schritt-1-keycloak-mit-docker-starten)
- [Schritt 2: Keycloak mit einer Datenbank betreiben](#schritt-2-keycloak-mit-einer-datenbank-betreiben)
- [Schritt 3: Keycloak via Docker Compose starten](#schritt-3-keycloak-via-docker-compose-starten)
- [Best Practices für Compose-Dateien](#best-practices-für-compose-dateien)
- [Vom Testsystem zur Produktion](#vom-testsystem-zur-produktion)
- [Use Cases](#use-cases)
- [Fazit](#fazit)
- [Support & Links](#support--links)

---

## Voraussetzungen

- Installiertes **Docker** (lokal oder remote) inkl. `docker compose`
- **Internetzugang** zum Laden der Images
- Ein freier **Port** (Standard `8080`, alternativ frei wählbar)

---

## Schritt 1: Keycloak mit Docker starten

Der schnellste Einstieg – Keycloak im Entwicklungsmodus, ohne externe Datenbank:

```bash
docker run --name keycloak \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  -p 8080:8080 \
  -d quay.io/keycloak/keycloak:latest start-dev
```

Was hier passiert:

1. Das Keycloak-Image wird von `quay.io/keycloak/keycloak` geladen.
2. Admin-Benutzer und -Passwort werden per Umgebungsvariablen gesetzt
   (`KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD`).
3. Container-Port `8080` wird auf Host-Port `8080` gemappt.
4. Die Admin-Oberfläche ist erreichbar unter <http://localhost:8080/>.

> **Hinweis:** Ist Port `8080` bereits belegt, mappen Sie auf einen anderen Host-Port,
> z. B. `-p 9090:8080`.

Öffnen Sie <http://localhost:8080> und melden Sie sich an:

- Benutzername: `admin`
- Passwort: `admin`

Danach können Sie Benutzer, Rollen, Clients und weitere Einstellungen verwalten.

> ⚠️ `start-dev` ist ausschließlich für Entwicklung und Tests gedacht – **nicht** für die Produktion.

---

## Schritt 2: Keycloak mit einer Datenbank betreiben

Ohne externe Datenbank gehen Daten beim Entfernen des Containers verloren. Für persistente
Speicherung und mehrere Keycloak-Instanzen wird eine externe Datenbank empfohlen – hier
**PostgreSQL**.

**2.1 Netzwerk erstellen**

```bash
docker network create keycloak-network
```

**2.2 PostgreSQL-Container starten**

```bash
docker run --name keycloak-db \
  --network keycloak-network \
  -e POSTGRES_USER=keycloak \
  -e POSTGRES_PASSWORD=keycloak \
  -e POSTGRES_DB=keycloak \
  -d postgres:latest
```

**2.3 Keycloak mit DB-Anbindung starten**

```bash
docker run --name keycloak \
  --network keycloak-network \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  -e KC_DB=postgres \
  -e KC_DB_URL_HOST=keycloak-db \
  -e KC_DB_USERNAME=keycloak \
  -e KC_DB_PASSWORD=keycloak \
  -p 8080:8080 \
  -d quay.io/keycloak/keycloak:latest start-dev
```

**2.4 Umgebungsvariablen erklärt**

| Variable          | Bedeutung                                                        |
| ----------------- | --------------------------------------------------------------- |
| `KC_DB`           | Datenbank-Treiber (hier `postgres`)                             |
| `KC_DB_URL_HOST`  | Adresse der Datenbank (hier der Name des PostgreSQL-Containers) |
| `KC_DB_USERNAME`  | Benutzername für den Datenbankzugriff                           |
| `KC_DB_PASSWORD`  | Passwort für den Datenbankzugriff                               |

---

## Schritt 3: Keycloak via Docker Compose starten

Mit Docker Compose lassen sich beide Container in einer Datei konfigurieren und gemeinsam starten.
Die passenden Dateien liegen in diesem Repo:

- [`docker-compose.yaml`](docker-compose.yaml) – Keycloak + PostgreSQL
- [`.env.example`](.env.example) – Vorlage für die Konfiguration

**3.1 Konfiguration vorbereiten**

```bash
cp .env.example .env
# .env öffnen und alle Platzhalter durch sichere Werte ersetzen
```

**3.2 Umgebung starten**

```bash
docker compose up
```

Anschließend ist Keycloak wieder unter <http://localhost:8080> erreichbar.

> **Wichtig:** Ersetzen Sie die Platzhalter in `.env` durch sichere Werte – besonders für
> produktive Umgebungen. Nutzen Sie für Secrets idealerweise ein Werkzeug wie
> [SOPS](https://github.com/getsops/sops).

---

## Best Practices für Compose-Dateien

- **Versionskontrolle nutzen (Git):** verringert das Risiko von Konfigurationsverlust und macht
  Änderungen nachvollziehbar.
- **Keine Secrets in der Compose-Datei:** Passwörter in eine separate `.env`-Datei auslagern und
  diese **außerhalb** der Versionskontrolle halten (siehe [`.gitignore`](.gitignore)).
- **Feste Versionsnummern für Images:** kein `latest` verwenden – das vermeidet unerwartete Updates
  und sichert Nachvollziehbarkeit.
- **Logging & Monitoring:** Aufbewahrung von Protokollen einplanen, z. B. via `journald` oder
  ELK-Stack.

---

## Vom Testsystem zur Produktion

Eine Testumgebung mit persistenter Datenhaltung ist ein guter Anfang. Für den Produktionsbetrieb
steigen die Anforderungen an Sicherheit, Verfügbarkeit, Compliance und Integrationen. Keycloak folgt
dem Prinzip **„Secure by Default"** – der Produktionsmodus ist daher anspruchsvoller.

1. **Vom Entwicklungs- in den Produktionsmodus wechseln** – statt `start-dev`:

   ```yaml
   command: start
   ```

2. **Festen Hostnamen setzen:**

   ```bash
   KC_HOSTNAME=keycloak.example.com
   ```

3. **TLS (HTTPS) aktivieren** – TLS konfigurieren oder HTTP via `KC_HTTP_ENABLED` erlauben, wenn TLS
   am Reverse Proxy terminiert wird.

4. **Backup- und Restore-Konzept definieren** – Produktivdaten brauchen ein Backup-Konzept;
   PostgreSQL bietet automatische Dumps. Dedizierte Datenbankserver bevorzugen.

5. **Logging- und Monitoring-Strategie** – Anmeldeversuche und Fehler überwachen (`journald` oder
   ELK-Stack); dabei Datenschutzrichtlinien zur Log-Aufbewahrung beachten. Ohne Logging bleiben
   Sicherheitsvorfälle unbemerkt.

---

## Use Cases

Was Ihre Keycloak-Umgebung jetzt leisten kann:

1. **Neue Authentifizierungsprozesse testen** – SSO für interne Anwendungen, nahtlose Navigation.
2. **Mehrfaktor-Authentifizierung (MFA)** – YubiKeys, SMS oder andere MFA-Lösungen anbinden.
3. **Externe Verzeichnisdienste** – Active Directory, LDAP oder Azure AD verbinden (hybride IT).
4. **OAuth2-Clients integrieren** – Testanwendungen via OAuth2 / OpenID Connect prüfen.
5. **Datenbank-Backups testen** – PostgreSQL-Integration und automatisierte Backups verifizieren.
6. **Systemmonitoring** – Prometheus & Grafana für Logins, Rollenänderungen und Fehler.

---

## Fazit

Mit diesem Quick Start gelingt ein schneller technischer Einstieg in Keycloak. Der
produktionstaugliche Einsatz bleibt jedoch vielseitig: Umgang mit Konfigurationsparametern nach
Updates, DSGVO-konforme Protokollierung, Anbindung von Drittanwendungen, Migration älterer Stände,
Realm-Struktur, MFA, bestehende Benutzerdatenquellen, Azure-Anbindungen und sauberer Containerbetrieb
mit horizontaler Skalierung.

---

## Support & Links

- [Keycloak Security Scanner](https://www.smf.de/keycloak-scanner/) – Konfiguration auf
  Schwachstellen testen
- [Keycloak Beratung](https://www.smf.de/keycloak-beratung/) – vom Produktivsetup bis zur Integration
- [Keycloak – Offizielle Dokumentation](https://www.keycloak.org/documentation)

**Kontakt**
Phillip Conrad · Segment Manager Finance & Public
SMF GmbH · Paul-Henri-Spaak-Str. 5 · 44263 Dortmund · <info@smf.de> · +49 231 9644-0
