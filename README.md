# OAuth Server

A Spring Boot OAuth/JWT authentication service backed by PostgreSQL, deployed to a self-hosted Raspberry Pi Kubernetes (K3s) cluster. Includes a lightweight built-in web UI for registration, login, and a protected dashboard.

---

## Tech Stack

- **Java 21**
- **Spring Boot 4.1** (Web, Security, Validation, Data JPA)
- **PostgreSQL** — persistent, shared user storage across replicas
- **JJWT 0.12.6** (`io.jsonwebtoken`) — JWT signing/validation
- **H2** (test scope) — in-memory database for automated tests, isolated from the real Postgres config
- **Docker** — multi-arch builds (`linux/amd64` + `linux/arm64`) for ARM64 Raspberry Pi nodes
- **Kubernetes / K3s** — 3-node Raspberry Pi cluster

---

## Project Structure

```
src/main/java/com/example/oauthserver/
├── OauthServerApplication.java
├── config/
│   ├── JacksonConfig.java            # ObjectMapper bean
│   └── PasswordConfig.java           # PasswordEncoder bean (BCrypt)
├── entity/
│   └── User.java                     # JPA entity (id, username, password, role)
├── repository/
│   └── UserRepository.java           # Spring Data JPA repository
├── service/
│   └── UserService.java              # Registration logic, password hashing
├── security/
│   ├── SecurityConfig.java           # Filter chain, permitAll rules, stateless sessions
│   ├── JwtAuthenticationFilter.java  # Validates Bearer tokens per-request
│   ├── JwtService.java               # Token generation/validation
│   ├── JwtProperties.java            # Binds jwt.* properties
│   ├── RefreshTokenService.java      # Refresh token generation/validation
│   └── CustomUserDetailsService.java # Loads users for Spring Security
└── controller/
    ├── AuthController.java           # POST /auth/register
    ├── LoginController.java          # POST /auth/login, /auth/refresh
    └── ProtectedApiController.java   # GET /api/hello, /api/me (requires a valid token)

src/main/resources/
├── application.properties
└── static/
    ├── login.html                    # Login form
    ├── register.html                 # Registration form
    ├── dashboard.html                 # Post-login landing page
    ├── auth.js                       # Shared token/session helpers
    └── style.css
```

---

## API Endpoints

### Register
```
POST /auth/register
Content-Type: application/json

{ "username": "john", "password": "yourpassword" }
```
Passwords are hashed with BCrypt before storage. New users are assigned `ROLE_USER`.

### Login
```
POST /auth/login
Content-Type: application/json

{ "username": "john", "password": "yourpassword" }
```
Response:
```json
{
  "accessToken": "...",
  "refreshToken": "...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "username": "john"
}
```

### Refresh
```
POST /auth/refresh
Content-Type: application/json

{ "refreshToken": "..." }
```

### Protected endpoints
Require `Authorization: Bearer <accessToken>`:
```
GET /api/hello   → plain text welcome message
GET /api/me      → { "username": "...", "authorities": [...] }
```

---

## Web UI

A minimal built-in UI is served directly from the same jar — no separate frontend project or build step:

- **`/login.html`** — sign in, stores the returned tokens in `sessionStorage`
- **`/register.html`** — create a new account
- **`/dashboard.html`** — calls `/api/me` and `/api/hello` using the stored token; redirects to login if no valid session exists

Since the app uses stateless JWTs (not cookies/sessions), the page shells themselves are publicly loadable — the actual protection happens client-side (`auth.js`'s `requireAuth()`/`authFetch()`) *and* is enforced server-side on the real data endpoints (`/api/me`, `/api/hello`), which correctly reject requests without a valid Bearer token. This split matters because a plain browser navigation (e.g. `window.location.href`) cannot attach a custom `Authorization` header — only JavaScript-initiated `fetch()` calls can — so gating the HTML shell itself server-side would break normal navigation for every legitimate user.

---

## Configuration

`application.properties`:
```properties
spring.application.name=oauth-server

jwt.secret=${JWT_SECRET:...}
jwt.access-expiration=${JWT_ACCESS_EXPIRATION:3600000}
jwt.refresh-expiration=${JWT_REFRESH_EXPIRATION:604800000}

spring.datasource.url=jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:oauthdb}
spring.datasource.username=${DB_USERNAME:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

All sensitive values are environment-variable-driven with local-dev fallbacks. In Kubernetes, `JWT_SECRET` and the `DB_*` values are injected via Secrets — never hardcoded in the deployed image.

**JWT secret requirement:** must decode to at least 256 bits for HMAC-SHA, or the app throws `WeakKeyException` on first token generation:
```bash
openssl rand -base64 32
```

---

## Running Locally

```bash
mvn clean package
mvn spring-boot:run
```
Requires a reachable Postgres instance (local Docker container, or override `DB_HOST`/etc. as needed). Tests run against an isolated in-memory H2 database (`src/test/resources/application.properties`) and don't require Postgres to be running.

---

## Building the Docker Image

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/oauth-server-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Multi-arch build (required for ARM64 Raspberry Pi nodes)

```bash
docker buildx create --use --name multiarch-builder
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t <dockerhub-username>/oauth-server:<tag> \
  --push .
```
Always bump the tag for each new build — reusing a tag risks nodes serving stale cached images instead of pulling the new content.

---

## Kubernetes Deployment (K3s)

### Secrets
```bash
kubectl create secret generic oauth-secrets \
  --from-literal=JWT_SECRET='<256-bit-secret>' \
  --from-literal=JWT_ACCESS_EXPIRATION='3600000' \
  --from-literal=JWT_REFRESH_EXPIRATION='604800000'

kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_DB=oauthdb \
  --from-literal=POSTGRES_USER=oauthuser \
  --from-literal=POSTGRES_PASSWORD='<password>'
```

### Postgres (Deployment + PVC + Service)
Runs as its own pod with persistent storage via a PVC. Exposed only internally (`ClusterIP`) as `postgres:5432`.

### oauth-server (Deployment + Service)
```yaml
env:
  - name: DB_HOST
    value: postgres
  - name: DB_PORT
    value: "5432"
  - name: DB_NAME
    valueFrom: { secretKeyRef: { name: postgres-secret, key: POSTGRES_DB } }
  - name: DB_USERNAME
    valueFrom: { secretKeyRef: { name: postgres-secret, key: POSTGRES_USER } }
  - name: DB_PASSWORD
    valueFrom: { secretKeyRef: { name: postgres-secret, key: POSTGRES_PASSWORD } }
```
Exposed via a `NodePort` Service so it's reachable from the local network without port-forwarding.

```bash
kubectl apply -f postgres-pvc.yaml
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f oauth-deployment.yaml
kubectl apply -f oauth-service.yaml
```

---

## Horizontal Scaling

Since all replicas share the same Postgres backend, `oauth-server` scales safely:
```bash
kubectl scale deployment oauth-server --replicas=3
```
Verified under load with JMeter — 3 replicas showed ~2.4x higher throughput and roughly a third of the average response time compared to a single replica, with 0% error rate across both register and login flows.

---

## Accessing the Service

- **In-cluster:** `oauth-service:8080` (ClusterIP + internal DNS)
- **Local network:** `http://<pi-static-ip>:<nodeport>/login.html`
- **From anywhere:** a Cloudflare Tunnel (`cloudflared tunnel --url http://localhost:<nodeport>`) provides a temporary public HTTPS URL with no router configuration or domain required

---

## Testing

- **Unit/integration tests** run against H2 in-memory, isolated from the production Postgres config
- **BDD (Cucumber)** — Gherkin feature files describing registration and login behavior in plain English, backed by Spring-Boot-integrated step definitions (`cucumber-spring`, `RANDOM_PORT` web environment)
- **Load testing (JMeter)** — a corrected test plan generates a unique username once per iteration (via a properly-scoped JSR223 PreProcessor) and reuses it across the register→login pair, avoiding a username-mismatch bug that otherwise produces a false 100% login failure rate

---

## Known Limitations / Lessons Learned

- **Raspberry Pi node IP stability matters a lot.** Nodes on DHCP can silently receive a new IP after a reboot, breaking kubelet's self-registration and cascading into `NotReady` nodes and DNS failures across the cluster. All nodes now use static IPs (`nmcli`, since these Pis run NetworkManager rather than `dhcpcd`).
- **NIC checksum offloading** on some Pi WiFi hardware can corrupt VXLAN-encapsulated UDP traffic (used by Flannel's default backend), breaking cross-node DNS resolution intermittently. Fixed via `ethtool -K eth0 tx off rx off`, made persistent through a systemd unit (`fix-checksum-offload.service`) since the setting doesn't survive reboots on its own.
- **Postgres is a single point of failure** — one replica, no automatic failover. Acceptable for a personal/learning cluster; would need replication or a managed service for anything beyond that.
- **No HTTPS/TLS at the app layer** — the app itself serves plain HTTP. External exposure should terminate TLS in front of it (Cloudflare Tunnel does this automatically).
