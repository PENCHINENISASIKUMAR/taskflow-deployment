# TaskFlow

TaskFlow is a React application backed by Spring Boot user and task services. Eureka provides service discovery, and the Spring Cloud Gateway is the browser-facing API entry point. User authentication uses short-lived JWT access tokens in an HttpOnly cookie.

## Requirements

- Java 17 and Maven 3.8+
- Node.js 18+
- MySQL 8

The user and task services use separate MySQL schemas. Set `DB_URL`, `DB_USERNAME`, and `DB_PASSWORD` for each service. Each service can create its schema when the configured MySQL account has permission.

The user service, task service, and gateway load the same local JWT key from `.taskflow-dev-secrets.properties` at the repository root. That file is git-ignored, so the key persists across restarts without entering it in each terminal. A `JWT_SECRET_BASE64` environment variable overrides the local file if you have already configured one. For a fresh checkout, create the local file once in PowerShell:

```powershell
$bytes = New-Object byte[] 32
$rng = [Security.Cryptography.RandomNumberGenerator]::Create()
$rng.GetBytes($bytes)
$secret = [Convert]::ToBase64String($bytes)
Set-Content -Path .taskflow-dev-secrets.properties -Value "JWT_SECRET_BASE64=$secret" -Encoding ascii
```

Do not commit or log the secret. `JWT_EXPIRATION_SECONDS` defaults to 3600. `JWT_COOKIE_SECURE` defaults to `false` for local HTTP; enable it behind HTTPS.

## Start the applications

Run each command in a separate PowerShell terminal, from the repository root. Launch Eureka first, then the two services and gateway:

```powershell
cd eureka-server
mvn spring-boot:run
```

```powershell
cd user-service
$env:DB_URL='jdbc:mysql://localhost:3306/taskflow_users?serverTimezone=UTC'
$env:DB_USERNAME='your-local-db-user'
$env:DB_PASSWORD='your-local-db-password'
mvn spring-boot:run
```

```powershell
cd task-service
$env:DB_URL='jdbc:mysql://localhost:3306/taskflow_tasks?serverTimezone=UTC'
$env:DB_USERNAME='your-local-db-user'
$env:DB_PASSWORD='your-local-db-password'
mvn spring-boot:run
```

```powershell
cd api-gateway
mvn spring-boot:run
```

The services register with Eureka and the gateway routes by service name. Internal service ports are intended for local development; the frontend only calls the gateway.

Start the React frontend:

```powershell
cd frontend
npm install
npm run dev
```

The frontend runs at `http://localhost:5173`. During development, Vite proxies `/api` requests to the gateway at `127.0.0.1:8080`, keeping browser requests same-origin. For a separately hosted frontend, set `VITE_API_URL` to the gateway base URL (ending in `/api`).

## Endpoints and documentation

- Gateway: `http://localhost:8080`
- Eureka dashboard: `http://localhost:8761`
- Register: `POST /api/auth/register` (also supported at `POST /api/users/register`)
- Login: `POST /api/auth/login`
- Current authenticated user: `GET /api/auth/me`
- Logout: `POST /api/auth/logout`
- Tasks: `/api/tasks/**`
- User Swagger UI via gateway: `http://localhost:8080/docs/user/swagger-ui.html`
- Task Swagger UI via gateway: `http://localhost:8080/docs/task/swagger-ui.html`
- Gateway OpenAPI: `http://localhost:8080/v3/api-docs`

The browser receives the JWT in an HttpOnly, SameSite=Strict cookie and cannot read it from JavaScript. Cookie-authenticated state-changing requests use the `XSRF-TOKEN` cookie and `X-XSRF-TOKEN` header; the frontend Axios client handles these. Use HTTPS and set `JWT_COOKIE_SECURE=true` outside local development. The gateway also accepts bearer tokens for API clients and Swagger's Authorize control. Access tokens expire after the configured lifetime; the frontend clears the session on a 401 response.

Task ownership is derived from the authenticated token subject. The task API does not trust a submitted `userId`; a user cannot read, update, or delete another user's task.

## Build and verify

From each backend module (`eureka-server`, `user-service`, `task-service`, and `api-gateway`), run:

```powershell
mvn clean verify
```

In `frontend`, run `npm run build`.

## Local configuration

Database credentials and the JWT secret are supplied through environment variables. Do not commit production secrets. For deployment, expose only the gateway publicly, use HTTPS, set the secure cookie option, and configure CORS to the deployed frontend origin.

## Docker Compose

Docker Desktop (or Docker Engine with the Compose plugin) is required. The stack builds five application images and runs one MySQL 8.4 container with separate `taskflow_users` and `taskflow_tasks` databases. The database uses the persistent `mysql_data` volume. Docker Compose waits for MySQL, Eureka, and the application health checks before starting their dependants; backend containers also restart after startup failures so they can retry transient database or registry outages.

Jenkins is optional and excluded from the normal application startup. Start its containers separately with the `ci` profile as described below.

The Compose file has development-only defaults so a fresh checkout can start with the requested one-command workflow. For local overrides, copy `.env.example` to `.env` and set your own database passwords and a random Base64 JWT secret. One way to generate a key in PowerShell is:

```powershell
$bytes = New-Object byte[] 32
$rng = [Security.Cryptography.RandomNumberGenerator]::Create()
$rng.GetBytes($bytes)
$jwtSecret = [Convert]::ToBase64String($bytes)
$rng.Dispose()
$jwtSecret
```

Copy the printed value into `.env` as `JWT_SECRET_BASE64=...`. Never use the committed development defaults outside local development.

Build and start the complete stack:

```powershell
docker compose up --build -d
```

Inspect container state and health:

```powershell
docker compose ps
```

Follow all logs or one service:

```powershell
docker compose logs -f
docker compose logs -f api-gateway
```

The frontend is at `http://localhost`; the gateway is at `http://localhost:8080`; and the Eureka dashboard is at `http://localhost:8761`. Swagger is available at `http://localhost:8080/docs/user/swagger-ui.html` and `http://localhost:8080/docs/task/swagger-ui.html`.

Build one image or start an individual service with its dependencies:

```powershell
docker compose build task-service
docker compose up --build -d task-service
```

Restart a service after changing configuration or code:

```powershell
docker compose up --build -d api-gateway
```

Stop containers while preserving database data, then start them again:

```powershell
docker compose down
docker compose up -d
```

To intentionally delete persisted database contents, use `docker compose down -v`. This removes the MySQL volume. The database initialization script runs only when MySQL creates an empty data directory; changing credentials later does not change existing MySQL users in a populated volume.

Basic availability checks in PowerShell:

```powershell
Invoke-WebRequest http://localhost -UseBasicParsing
Invoke-RestMethod http://localhost:8761/actuator/health
Invoke-RestMethod http://localhost:8080/actuator/health
```

If startup fails, inspect `docker compose ps` and `docker compose logs mysql user-service task-service eureka-server api-gateway frontend`. A database connection failure usually means MySQL is still initializing or existing volume credentials do not match the current `.env`; keep the volume and credentials aligned, or remove it only if you intend to discard its data. If a host port is already occupied, stop the conflicting process or change the published port on the corresponding Compose service. Use `docker compose config` to inspect the resolved configuration without starting containers.

## Jenkins CI/CD

Jenkins runs in Docker containers and is not installed on the host. The root `Jenkinsfile` checks out a GitHub branch, packages and tests the backend services, installs frontend dependencies, runs any configured frontend test script, builds the frontend, builds five Docker images, and publishes images from `main`, `master`, or `develop`. Each published image gets a 12-character commit SHA tag and a moving `dev` tag. The frontend package currently has no `test` script, so the pipeline reports frontend tests as skipped.

The Jenkins controller uses the official `jenkins/jenkins:lts-jdk21` image with required plugins preinstalled. A separate inbound-agent image supplies Java 17, Maven, Git, Node/npm, and Docker CLI. An isolated Docker-in-Docker service builds images for the agent. Docker-in-Docker requires privileged mode; its daemon is internal to the Compose network and is not published on the host. The Jenkins home and Docker build cache use persistent named volumes. See [Jenkins' official Docker instructions](https://www.jenkins.io/doc/book/installing/docker/) for the controller and Docker daemon arrangement.

### Start Jenkins

Start Docker Desktop in Linux-container mode, then start the Jenkins controller and its Docker daemon:

```powershell
docker compose --profile ci up --build -d jenkins docker-dind
```

Wait for Jenkins to become healthy, then open `http://localhost:8085`. Get the first-run password with:

```powershell
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Complete the setup wizard and create an administrator account. The controller image preinstalls Pipeline, Multibranch, Stage View, GitHub, Credentials Binding, JUnit, and Timestamper plugins. Jenkins state persists in the `jenkins_home` volume across container restarts.

### Configure the Docker build agent

In Jenkins, go to **Manage Jenkins > Nodes > New Node** and create a permanent agent with:

- Name: `taskflow-docker`
- Remote root directory: `/home/jenkins/agent`
- Label: `taskflow-docker`
- Number of executors: `1`
- Launch method: **Launch agent by connecting it to the controller** with WebSocket enabled

Copy that agent's secret into the ignored local `.env` file as `JENKINS_AGENT_SECRET=<agent-secret>`. Do not commit `.env` or share the secret. Then build and start the agent:

```powershell
docker compose --profile ci up --build -d jenkins-agent
```

In **Manage Jenkins > Tools**, define a JDK named `jdk17` at `/opt/java/openjdk` and Maven named `maven3` at `/usr/share/maven`. The agent image provides `git`, `node`, `npm`, and `docker` on its PATH. Docker runs inside the dedicated Docker-in-Docker service, so a host Docker socket is not mounted into Jenkins.

### Credentials and Docker Hub repositories

In **Manage Jenkins > Credentials > System > Global credentials**, add:

- A GitHub username/token or SSH key credential that can read the repository. Select it as the branch source credential in the Multibranch Pipeline job.
- A **Username with password** credential with ID `dockerhub-credentials`. Use the Docker Hub username and a Docker Hub access token as the password.

Set the Jenkins global environment property `DOCKERHUB_NAMESPACE` to the Docker Hub user or organization that owns these repositories. Create the following repositories under that namespace before the first publish:

- `user-service`
- `task-service`
- `api-gateway`
- `eureka-server`
- `taskflow-frontend`

Do not put GitHub or Docker Hub tokens in the Jenkinsfile or repository. Jenkins exposes credentials to the pipeline by ID and only during the publishing stage. See the [Jenkins credentials guide](https://www.jenkins.io/doc/book/using-credentials/) and [Credentials Binding reference](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/).

### GitHub job and webhook

Create a **Multibranch Pipeline** job, add the existing GitHub repository as a branch source, select the read-only GitHub credential, and set the repository's correct branch discovery rules. This workspace does not contain a Git remote URL, so enter the URL of the existing TaskFlow GitHub repository in Jenkins. The job discovers the root `Jenkinsfile` from each branch and runs the pipeline on branch updates.

For automatic push notifications, configure a GitHub repository webhook with payload URL `https://<public-jenkins-url>/github-webhook/`, content type `application/json`, and the **Pushes** event. Jenkins must be reachable by GitHub over HTTPS for webhook delivery; for a private network, configure an approved inbound relay or use periodic multibranch scans. The [GitHub Branch Source plugin](https://plugins.jenkins.io/github-branch-source/) provides the multibranch GitHub integration.

Check the Stage View, console output, and Maven Surefire reports in Jenkins. Backend and frontend build/test failures fail the pipeline and prevent publishing. Only `main`, `master`, and `develop` publish Docker Hub images; every image is tagged with the commit SHA and those branches also update `dev`. The declarative Pipeline's `post` section reports JUnit XML and records build artifacts even after a failure, following [Jenkins Pipeline syntax](https://www.jenkins.io/doc/book/pipeline/syntax/).

The Jenkins controller and agent images are ready, but this workspace has no Git remote URL. Enter the existing TaskFlow GitHub repository URL in the Multibranch Pipeline job. GitHub checkout, webhook delivery, Docker Hub login/push, and a complete Jenkins run still need verification after Docker Desktop is running. Do not treat Phase 4 as complete until the pipeline successfully builds, tests, and publishes images.

Stop Jenkins containers without deleting configuration or build cache:

```powershell
docker compose stop jenkins jenkins-agent docker-dind
```

Do not run `docker compose down -v` unless you intend to delete Jenkins home, the Docker build cache, and the TaskFlow database volumes.
