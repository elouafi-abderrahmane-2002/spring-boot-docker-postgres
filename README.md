# 🐳 Spring Boot + PostgreSQL + Docker — Production-Ready Setup

"It works on my machine" is not a deployment strategy.

This project is my answer to that problem: a fully containerized Spring Boot API with
PostgreSQL, Keycloak for OAuth2/OIDC security, pgAdmin for database inspection, and
a Docker Compose setup that spins up the entire environment with a single command.

No manual DB setup. No environment-specific config leaking into code. Just `docker-compose up`.

---

## What runs in the containers

```
  docker-compose up
         │
         ├──► 🟢 spring-app    (port 8080)
         │         │
         │         │  JPA / Hibernate
         │         ▼
         ├──► 🐘 postgres      (port 5432)
         │         │
         │         │  SQL admin UI
         │         ▼
         ├──► 🛠️  pgadmin       (port 5050)
         │
         └──► 🔐 keycloak      (port 8180)
                   │
                   │  OpenID Connect / OAuth2
                   ▼
             Token endpoint:
             /auth/realms/myapp/protocol
             /openid-connect/token
```

---

## How the auth flow works

```
  Client
    │
    │  POST /auth/realms/myapp/protocol/openid-connect/token
    │  { grant_type: client_credentials, client_id, client_secret }
    │
    ▼
  Keycloak
    │
    │  200 OK → { access_token: "eyJ...", expires_in: 300 }
    │
    ▼
  Client
    │
    │  GET /api/automobiles
    │  Authorization: Bearer eyJ...
    │
    ▼
  Spring Boot
    │
    │  Spring Security validates JWT against Keycloak public key
    │  Token valid? ──► No ──► 401 Unauthorized
    │        │
    │        ▼ Yes
    │  Controller → Service → JPA Repository → PostgreSQL
    │
    ▼
  JSON Response
```

---

## Running it

```bash
git clone https://github.com/elouafi-abderrahmane-2002/spring-boot-docker-postgres.git
cd spring-boot-docker-postgres

# Build and start all containers
mvn clean install
docker-compose up

# Check running containers
docker-compose ps
```

| Service     | URL                        | Credentials            |
|-------------|----------------------------|------------------------|
| Spring API  | http://localhost:8080      | —                      |
| Swagger UI  | http://localhost:8080/swagger-ui | —               |
| pgAdmin     | http://localhost:5050      | admin / admin          |
| Keycloak    | http://localhost:8180/auth | admin / Pa55w0rd       |

---

## Project structure

```
spring-boot-docker-postgres/
│
├── src/main/java/
│   ├── controller/         ← REST endpoints (Automobile CRUD)
│   ├── model/              ← JPA entities
│   ├── repository/         ← Spring Data JPA
│   ├── service/            ← business logic
│   └── config/             ← Security, CORS, OpenAPI config
│
├── src/main/resources/
│   ├── application.yml     ← datasource, JPA, Keycloak settings
│   └── data.sql            ← optional seed data
│
├── docker-compose.yml      ← defines all 4 services + volumes
├── Dockerfile              ← multi-stage build (Maven → JRE)
└── pom.xml
```

---

## The Dockerfile (multi-stage)

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline        # cache dependencies layer
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run (smaller image)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Multi-stage build keeps the final image lean — no Maven, no source code, just the JRE + JAR.

---

## What I learned doing this

Keycloak integration was the hardest part. The official docs are thorough but assume you already
know how OAuth2 realms, clients, and scopes work. I spent a good chunk of time just understanding
the difference between a *confidential* client (has a secret, used by server-side apps) and a
*public* client (no secret, used by SPAs). Getting the `Access Type` setting right in the Keycloak
admin console was the key.

Also: Docker networking. When your Spring app container tries to connect to `localhost:5432`,
it looks for Postgres *inside its own container*. You have to use the service name defined
in `docker-compose.yml` (`db`) as the hostname instead. This is one of those "obvious in
hindsight" things that trip everyone up the first time.

---

*Final-year engineering project — ENSET Mohammedia, Big Data & Cloud Computing*
*By **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
