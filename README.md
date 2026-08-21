# Keycloak Quick Start with Docker Compose

Get started with [Keycloak](https://www.keycloak.org/) quickly and securely – from `docker run` to a
production-ready Docker Compose setup with PostgreSQL.

> Based on the technical article by Nils Bergmann and Phillip Conrad (SMF, Finance & Public segment):
> <https://www.smf.de/keycloak-quick-start/> (German version of this Readme)


Keycloak is an open-source solution for **Identity & Access Management (IAM)**. It enables
centralized management of user logins, authentication and authorization, and supports
**OAuth 2.0**, **OpenID Connect** and **SAML** – with no licensing costs.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Start Keycloak with Docker](#step-1-start-keycloak-with-docker)
- [Step 2: Run Keycloak with a database](#step-2-run-keycloak-with-a-database)
- [Step 3: Start Keycloak via Docker Compose](#step-3-start-keycloak-via-docker-compose)
- [Best practices for Compose files](#best-practices-for-compose-files)
- [From test system to production](#from-test-system-to-production)
- [Use cases](#use-cases)
- [Conclusion](#conclusion)
- [Support & Links](#support--links)

---

## Prerequisites

- **Docker** installed (local or remote) including `docker compose`
- **Internet access** to pull the images
- A free **port** (default `8080`, or any alternative)

---

## Step 1: Start Keycloak with Docker

The fastest way to start – Keycloak in development mode, without an external database:

```bash
docker run --name keycloak \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  -p 8080:8080 \
  -d quay.io/keycloak/keycloak:latest start-dev
```

What happens here:

1. The Keycloak image is pulled from `quay.io/keycloak/keycloak`.
2. The admin user and password are set via environment variables
   (`KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD`).
3. Container port `8080` is mapped to host port `8080`.
4. The admin console is available at <http://localhost:8080/>.

> **Note:** If port `8080` is already in use, map to a different host port,
> e.g. `-p 9090:8080`.

Open <http://localhost:8080> and sign in:

- Username: `admin`
- Password: `admin`

You can then manage users, roles, clients and other settings.

> ⚠️ `start-dev` is intended for development and testing only – **not** for production.

---

## Step 2: Run Keycloak with a database

Without an external database, data is lost when the container is removed. For persistent storage and
running multiple Keycloak instances, an external database is recommended – here **PostgreSQL**.

**2.1 Create a network**

```bash
docker network create keycloak-network
```

**2.2 Start the PostgreSQL container**

```bash
docker run --name keycloak-db \
  --network keycloak-network \
  -e POSTGRES_USER=keycloak \
  -e POSTGRES_PASSWORD=keycloak \
  -e POSTGRES_DB=keycloak \
  -d postgres:latest
```

**2.3 Start Keycloak with the database connection**

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

**2.4 Environment variables explained**

| Variable          | Meaning                                                     |
| ----------------- | ---------------------------------------------------------- |
| `KC_DB`           | Database driver (here `postgres`)                          |
| `KC_DB_URL_HOST`  | Database address (here the name of the PostgreSQL container) |
| `KC_DB_USERNAME`  | Username for database access                               |
| `KC_DB_PASSWORD`  | Password for database access                               |

---

## Step 3: Start Keycloak via Docker Compose

With Docker Compose you can configure and start both containers from a single file. The matching
files are included in this repo:

- [`docker-compose.yaml`](docker-compose.yaml) – Keycloak + PostgreSQL
- [`.env.example`](.env.example) – configuration template

**3.1 Prepare the configuration**

```bash
cp .env.example .env
# open .env and replace all placeholders with secure values
```

**3.2 Start the environment**

```bash
docker compose up
```

Keycloak is then available again at <http://localhost:8080>.

> **Important:** Replace the placeholders in `.env` with secure values – especially for production
> environments. For secrets, ideally use a tool such as [SOPS](https://github.com/getsops/sops).

---

## Best practices for Compose files

- **Use version control (Git):** reduces the risk of losing configuration and makes changes
  traceable.
- **No secrets in the Compose file:** move passwords into a separate `.env` file and keep it
  **outside** version control (see [`.gitignore`](.gitignore)).
- **Pin image versions:** avoid `latest` – this prevents unexpected updates and ensures
  reproducibility.
- **Logging & monitoring:** plan for log retention, e.g. via `journald` or the ELK stack.

---

## From test system to production

A test environment with persistent storage is a good start. For production use, the requirements for
security, availability, compliance and integrations increase. Keycloak follows the **"Secure by
Default"** principle – the production mode is therefore more demanding.

1. **Switch from development to production mode** – instead of `start-dev`:

   ```yaml
   command: start
   ```

2. **Set a fixed hostname:**

   ```bash
   KC_HOSTNAME=keycloak.example.com
   ```

3. **Enable TLS (HTTPS)** – configure TLS, or allow HTTP via `KC_HTTP_ENABLED` if TLS is terminated
   at the reverse proxy.

4. **Define a backup and restore concept** – production data needs a backup concept; PostgreSQL
   offers automatic dumps. Prefer dedicated database servers.

5. **Logging and monitoring strategy** – monitor login attempts and errors (`journald` or the ELK
   stack); observe data protection rules for log retention. Without logging, security incidents go
   unnoticed.

---

## Use cases

What your Keycloak environment can now do:

1. **Test new authentication flows** – SSO for internal applications, seamless navigation.
2. **Multi-factor authentication (MFA)** – connect YubiKeys, SMS or other MFA solutions.
3. **External directory services** – connect Active Directory, LDAP or Azure AD (hybrid IT).
4. **Integrate OAuth2 clients** – test applications via OAuth2 / OpenID Connect.
5. **Test database backups** – verify PostgreSQL integration and automated backups.
6. **System monitoring** – Prometheus & Grafana for logins, role changes and errors.

---

## Conclusion

This Quick Start gives you a fast technical entry into Keycloak. Production-grade operation, however,
remains multifaceted: handling configuration parameters after updates, GDPR-compliant logging,
integrating third-party applications, migrating older versions, realm structure, MFA, existing user
data sources, Azure integrations, and clean container operation with horizontal scaling.

---

## Support & Links

- [Keycloak Security Scanner](https://www.smf.de/keycloak-scanner/) – test your configuration for
  vulnerabilities
- [Keycloak Consulting](https://www.smf.de/keycloak-beratung/) – from production setup to integration
- [Keycloak – Official Documentation](https://www.keycloak.org/documentation)
