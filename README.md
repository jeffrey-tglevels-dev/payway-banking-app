# Core Banking Application

A full-stack banking application built with Spring Boot and React. Version 1 covers the essentials: users can register, log in, check their balance and transfer money. Later versions will harden the application for production (see [Roadmap](#roadmap)).

> **Status:** early development. The Spring Boot project is scaffolded; v1 features are not implemented yet.

---

## v1 Scope

| Area | Feature | Description |
|------|---------|-------------|
| Authentication | Registration | Create a new customer account with validated input |
| Authentication | Login | Authenticate with credentials, secured by Spring Security |
| Operations | Check balance | View the current balance of the logged-in user's account |
| Operations | Transfer money | Send money from the user's account to another account |
| Frontend | React app | UI for registration, login, balance and transfers |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Java 25 |
| Framework | Spring Boot 4.1.2 |
| Database | MySQL (run with Docker) |
| Frontend | React.js |
| Build | Maven (wrapper included: `mvnw` / `mvnw.cmd`) |

### Backend dependencies

| Dependency | Purpose |
|------------|---------|
| Spring Web MVC | Exposes the HTTP/REST API the frontend calls |
| Spring Data JPA | Reads and writes database records |
| MySQL Driver | Connects the application to MySQL |
| Spring Security | Authentication and authorization |
| Validation | Validates incoming request data |
| Flyway Migration | Versions and applies database schema changes |
| Lombok | Removes boilerplate Java code (getters, constructors, etc.) |
| Spring Boot Actuator | Health checks and monitoring endpoints |
| Spring AI (OpenAI) | AI integration through OpenAI models (version 2.0.1) |

---

## Project Structure

The codebase uses **package by feature**: each business feature has its own package that holds all of its layers (controller, service, repository, entity, DTOs), instead of grouping code by technical layer.

Planned layout:

```
src/main/java/com/banking/core/
├── CoreApplication.java
├── auth/            # registration, login, security config
├── account/         # account entity, balance lookup
├── transfer/        # money transfers between accounts
└── common/          # shared exceptions, error handling, utilities

src/main/resources/
├── application.properties
└── db/migration/    # Flyway scripts (V1__init.sql, V2__..., ...)
```

---

## Getting Started

### Prerequisites

- JDK 25
- Docker
- Node.js and npm (for the React frontend)

### 1. Start MySQL

```bash
docker run -d --name banking-mysql \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=banking \
  -p 3306:3306 \
  mysql:8
```

### 2. Configure the application

Add the datasource and OpenAI settings to `src/main/resources/application.properties` (or supply them as environment variables):

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/banking
spring.datasource.username=root
spring.datasource.password=root

spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true

spring.ai.openai.api-key=${OPENAI_API_KEY}
```

> Keep secrets such as `OPENAI_API_KEY` and database passwords out of version control.

### 3. Run the backend

```bash
./mvnw spring-boot:run        # macOS / Linux / Git Bash
mvnw.cmd spring-boot:run      # Windows
```

The API starts on `http://localhost:8080`. Check health at `http://localhost:8080/actuator/health`.

### 4. Run the tests

```bash
./mvnw test
```

---

## Roadmap

v1 delivers the core features listed above. Later versions focus on making the application production-ready. See [roadmap.md](roadmap.md) for details.
