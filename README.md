# Point of Sale System

A standalone desktop Point of Sale application. Written entirely in Java 17, it uses Hibernate ORM directly against MySQL — no application server, no Spring — and delivers a modern UI through Java Swing with the FlatLaf look and feel.

---

## Overview

This is a fat-JAR desktop application that runs offline on any machine with a JRE 17+ and a reachable MySQL instance. The UI layer is built on Java Swing styled with [FlatLaf](https://www.formdev.com/flatlaf/) (light/dark-capable), MigLayout for responsive panel composition, and Material Design icon fonts for a contemporary visual appearance. Persistence is handled by Hibernate 6 mapped directly to MySQL — no JPA abstraction layer is interposed. Environment-specific configuration (database credentials, connection URL) is externalised to a `.env` file read at startup via `java-dotenv`, keeping secrets out of compiled artefacts.

---

## Technology Stack

| Concern | Technology |
|---|---|
| Language | Java 17 |
| Build | Apache Maven 3 (Maven Wrapper included) |
| UI Toolkit | Java Swing |
| Look & Feel | FlatLaf 3.5.2 + FlatLaf Extras 3.4.1 |
| Layout | MigLayout 11.4.2 (core + Swing) |
| Icons | JIconFont Google Material Design Icons 2.2.0.2, gradient-icon-font |
| SVG Rendering | svgSalamander 1.1.4, jsvg 1.6.1 |
| Animations | TimingFramework (Swing) 7.3.1 |
| Notifications | swing-toast-notifications 1.0.2 |
| ORM | Hibernate ORM 6.1.2.Final |
| Database | MySQL (mysql-connector-j 9.1.0) |
| Password Hashing | jBCrypt 0.4 |
| Environment Config | java-dotenv 5.2.2 |
| Date Picker | JDatePicker 1.3.4 |
| Native Access | JNA 5.5.0 + JNA Platform 5.5.0 |
| Code Generation | Lombok 1.18.34 |
| Testing | JUnit Jupiter 5.11.3 |
| Packaging | maven-shade-plugin 3.2.4 (fat JAR) |

---

## Architecture

The application follows a layered desktop architecture. There is no web server, REST layer, or Spring context — the process is a single JVM that owns both the UI and database connection pool.

```
┌──────────────────────────────────────────────┐
│              Swing UI (FlatLaf)               │
│   org.POS.frontend.src.raven.application.*   │
│   Panels / Forms / Dialogs / Tables          │
└──────────────┬───────────────────────────────┘
               │  calls
┌──────────────▼───────────────────────────────┐
│             Service / Controller Layer        │
│          Business logic, validation,          │
│          transaction coordination             │
└──────────────┬───────────────────────────────┘
               │  EntityManager / Session
┌──────────────▼───────────────────────────────┐
│           Hibernate ORM 6 (no JPA wrapper)    │
│         Entities, repositories, queries       │
└──────────────┬───────────────────────────────┘
               │  JDBC
┌──────────────▼───────────────────────────────┐
│                  MySQL Database               │
└──────────────────────────────────────────────┘
```

The entry point is `org.POS.frontend.src.raven.application.Application`. The `raven` sub-package naming is consistent with the "Raven" Swing UI template pattern, where a main `Application` class bootstraps the FlatLaf theme, loads the `.env` configuration, initialises the Hibernate `SessionFactory`, and launches the main `JFrame`.

---

## Project Structure

```
pos-desktop-application/
├── src/
│   └── main/
│       └── java/
│           └── org/POS/
│               └── frontend/src/raven/
│                   └── application/
│                       └── Application.java   ← main entry point
├── libs/                                      ← vendored JARs not on Maven Central
├── out/
│   └── artifacts/POS_jar/                 ← pre-built fat JAR
├── .env                                       ← database credentials (not committed in production)
├── .mvn/wrapper/                              ← Maven Wrapper binaries
├── nbactions.xml                              ← NetBeans run/debug/profile actions
├── pom.xml
└── todo.txt
```

The `libs/` directory contains dependencies sourced outside Maven Central (JDatePicker, JNA, swing-toast-notifications, gradient-icon-font) registered under the `com.example` group in `pom.xml`. These are resolved from the local repository rather than a remote registry.

---

## Prerequisites

- **Java 17** or later (`java -version`)
- **Maven 3.8+** — or use the bundled `mvnw` / `mvnw.cmd` wrappers (no separate Maven install required)
- **MySQL 8+** running and accessible
- A created database schema (DDL managed by Hibernate `hbm2ddl` or manual script — confirm with your schema files)

---

## Local Development Setup

### 1. Clone the repository

```bash
git clone https://github.com/michaelangelozara/pos-desktop-application.git
cd pos-desktop-application
```

### 2. Install local library dependencies

The vendored JARs in `libs/` must be installed into your local Maven repository before the first build:

```bash
mvn install:install-file -Dfile=libs/jdatepicker-1.3.4.jar \
    -DgroupId=com.example -DartifactId=jdatepicker -Dversion=1.3.4 -Dpackaging=jar

mvn install:install-file -Dfile=libs/jna-5.5.0.jar \
    -DgroupId=com.example -DartifactId=jna -Dversion=5.5.0 -Dpackaging=jar

mvn install:install-file -Dfile=libs/jna-platform-5.5.0.jar \
    -DgroupId=com.example -DartifactId=jna-platform -Dversion=5.5.0 -Dpackaging=jar

mvn install:install-file -Dfile=libs/swing-toast-notifications-1.0.2.jar \
    -DgroupId=com.example -DartifactId=swing-toast-notifications -Dversion=1.0.2 -Dpackaging=jar

mvn install:install-file -Dfile=libs/gradient-icon-font-1.0.0.jar \
    -DgroupId=com.example -DartifactId=gradient-icon-font -Dversion=1.0.0 -Dpackaging=jar
```

> Adjust filenames to match the exact JARs present in `libs/`. Check `pom.xml` for the definitive artifact coordinates.

### 3. Configure the environment

Copy or edit `.env` in the project root:

```dotenv
DB_URL=jdbc:mysql://localhost:3306/pos_db
DB_USERNAME=your_db_user
DB_PASSWORD=your_db_password
```

> **Important:** The `.env` file is committed to this repository (visible in the file tree). Ensure production credentials are never stored here. Rotate credentials and use environment-level secrets for any shared or deployed environment.

### 4. Set up the database

Create the target schema in MySQL:

```sql
CREATE DATABASE pos_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Hibernate's DDL generation will create tables on first run if `hbm2ddl.auto` is set to `create` or `update` in the Hibernate configuration. Verify the setting in your Hibernate configuration resource before running against a populated database to avoid accidental schema drops.

### 5. Build

```bash
# Using the Maven Wrapper (recommended — no separate Maven install needed)
./mvnw clean package        # Linux / macOS
mvnw.cmd clean package      # Windows
```

This produces a fat JAR via `maven-shade-plugin` in `target/POS-1.0-SNAPSHOT.jar`, embedding all dependencies.

### 6. Run

```bash
java -jar target/POS-1.0-SNAPSHOT.jar
```

Or via Maven directly (as configured in `nbactions.xml`):

```bash
./mvnw process-classes exec:exec
```

---

## Running in an IDE

### NetBeans

The `nbactions.xml` file defines **Run**, **Debug**, and **Profile** actions that invoke `exec-maven-plugin` with `org.POS.frontend.src.raven.application.Application` as the main class. Open the project as a Maven project and use the standard Run/Debug toolbar buttons.

### IntelliJ IDEA

Open the project root as a Maven project. Create a run configuration of type **Application** with:

- **Main class**: `org.POS.frontend.src.raven.application.Application`
- **Working directory**: project root (required for `.env` resolution)

---

## Building a Distributable JAR

The `maven-shade-plugin` is configured to produce a single shaded JAR containing all dependencies. The pre-built artefact is also committed under `out/artifacts/Zeusled_POS_jar/` for direct distribution without requiring a build step.

```bash
./mvnw clean package -DskipTests
# Output: target/POS-1.0-SNAPSHOT.jar
```

Distribute the fat JAR alongside a populated `.env` (or instruct operators to set environment variables) and a JRE 17+ runtime.

---

## Authentication

User passwords are hashed with **jBCrypt** (bcrypt, work factor 10 by default). Plaintext passwords are never stored or compared. On login, the entered password is verified against the stored BCrypt hash via `BCrypt.checkpw()`.

---

## Testing

Tests are written with **JUnit Jupiter 5** and executed via `maven-surefire-plugin 3.2.5`:

```bash
./mvnw test
```

---

## Configuration Reference

All runtime-sensitive configuration is loaded from the `.env` file in the working directory at application startup via `java-dotenv`. The following keys are expected (exact names depend on Hibernate configuration wiring in source):

| Key | Description |
|---|---|
| `DB_URL` | JDBC connection URL (e.g. `jdbc:mysql://localhost:3306/pos_db`) |
| `DB_USERNAME` | MySQL username |
| `DB_PASSWORD` | MySQL password |

If a key is missing, `java-dotenv` will throw at startup, preventing the application from running with incomplete configuration.

---

## Known Active Work

From `todo.txt`:

> `net total JLabel in edit order`

The **Edit Order** screen has a pending UI fix — the net total label is not yet wired to recalculate dynamically when order lines are modified.

---

## Security Considerations

- The `.env` file containing database credentials is currently tracked by Git. Before sharing this repository or deploying to any environment beyond a local workstation, remove `.env` from version control (`git rm --cached .env`) and add it to `.gitignore`.
- BCrypt is used for password storage — do not downgrade to MD5, SHA-1, or any non-adaptive hash.
- JNA is included for native platform calls; review the specific use case if deploying on restricted or hardened systems.

---

## License

No license file is present in this repository. All rights are reserved by the author unless explicitly stated otherwise.
