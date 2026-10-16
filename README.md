# Todo Summary Assistant - DevOps Delivery & Operations

A full-stack application that manages to-do tasks, generates AI-powered summaries of pending tasks using **Cohere LLM**, and automatically posts the summaries to a **Slack** channel.

This repository contains the complete production-style DevOps pipeline, multi-stage Docker containerization, AWS cloud architecture, CI/CD automation via Jenkins, and Prometheus/Grafana monitoring.

---

## 1. Project Architecture

* **Frontend**: React application served with unprivileged Nginx reverse proxy (Port 80/3000)
* **Backend**: Java 17 Spring Boot REST API (Port 8080)
* **Database**: MySQL 8.0 (Local container) / AWS RDS MySQL (Production)
* **AI Engine**: Cohere API for task summarization
* **Notifications**: Slack Incoming Webhooks
* **CI/CD**: Automated Jenkins pipeline (`Jenkinsfile`)
* **Observability**: Prometheus (Port 9090) and Grafana (Port 3001)

---

## 2. DevOps Enablement Changes

As per the assessment guidelines, business logic was preserved while making the following DevOps enablement enhancements:
1. **Prometheus Metrics Endpoint**: Added `spring-boot-starter-actuator` and `micrometer-registry-prometheus` to [`pom.xml`](Backend/todo-summary-assistant/pom.xml) and exposed `/actuator/prometheus` and `/actuator/health`.
2. **Environment Variable Configuration**: Externalized database URLs, passwords, and API keys from `application.properties` to system environment variables with fallback defaults.
3. **Frontend API URL Flexibility**: Updated frontend API service to support `REACT_APP_API_BASE_URL` with a default relative `/api/todos` path through an Nginx reverse proxy.
4. **Multi-Stage Non-Root Dockerization**: Rebuilt both backend and frontend Dockerfiles using minimal alpine base images and non-root users (`appuser` UID 10001 and `nginx` UID 101).

---

## 3. Runtime Requirements & Prerequisites

* **Docker & Docker Compose** (version 20.10+ recommended)
* *Optional for bare-metal local development*:
  * Java JDK 17+
  * Node.js 18+ and npm
  * MySQL 8.0 Server

---

## 4. Environment Variables

Copy `.env.example` to `.env` to configure local or production variables:

```bash
cp .env.example .env
```

| Variable | Description | Default / Example |
| :--- | :--- | :--- |
| `DB_URL` | MySQL JDBC connection string | `jdbc:mysql://mysql:3306/todo_db?createDatabaseIfNotExist=true` |
| `DB_USERNAME` | Database username | `todo_user` |
| `DB_PASSWORD` | Database user password | `SecurePassword123!` |
| `COHERE_API_KEY` | API Key from Cohere.ai | `your_cohere_api_key` |
| `SLACK_WEBHOOK_URL`| Slack Incoming Webhook URL | `https://hooks.slack.com/services/...` |
| `PORT` | Backend application port | `8080` |
| `REACT_APP_API_BASE_URL`| Frontend API endpoint | `/api/todos` |

---

## 5. Local Execution Guide

### Option A: Run Entire Stack with Docker Compose (Recommended)

Start the entire application, database, and monitoring stack with a single command:

```bash
docker compose up --build -d
```

#### Access Services Locally:
* **React Frontend**: [http://localhost:3000](http://localhost:3000)
* **Spring Boot Backend**: [http://localhost:8080/api/todos](http://localhost:8080/api/todos)
* **Backend Health Check**: [http://localhost:8080/actuator/health](http://localhost:8080/actuator/health)
* **Prometheus Metrics**: [http://localhost:8080/actuator/prometheus](http://localhost:8080/actuator/prometheus)
* **Prometheus UI**: [http://localhost:9090](http://localhost:9090)
* **Grafana Dashboard**: [http://localhost:3001](http://localhost:3001) (User: `admin` / Password: `admin`)

To stop the containers:
```bash
docker compose down
```

---

### Option B: Run Manually on Host (Bare-Metal)

1. **Start MySQL Server**:
   Ensure MySQL is running on port 3306 with database `todo_db`.

2. **Run Backend**:
   ```bash
   cd Backend/todo-summary-assistant
   ./mvnw spring-boot:run
   ```

3. **Run Frontend**:
   ```bash
   cd Frontend/todo
   npm install
   npm start
   ```

---

## 6. CI/CD Pipeline (Jenkins)

The pipeline is defined in [`Jenkinsfile`](Jenkinsfile) with 5 stages:
1. **Test**: Runs backend unit tests (`mvn test`).
2. **Build**: Builds multi-stage Docker images tagged with the build number and `latest`.
3. **Push**: Pushes images to Docker Hub using Jenkins credentials.
4. **Deploy**: Connects to AWS EC2 over SSH, pulls the new images, and restarts containers.
5. **Health Check**: Queries `http://<EC2_IP>:8080/actuator/health` to confirm the deployment is healthy.

---

## 7. Assumptions Made

1. **Database Persistence**: MySQL is containerized only for local testing. In production, AWS RDS MySQL is used for high availability, security, and automated backups.
2. **Networking**: The backend container and Nginx frontend share a Docker bridge network (`todo-network`) where Nginx reverse-proxies `/api/` requests to `http://backend:8080/api/`.
3. **API Keys**: Cohere API and Slack Webhook integrations are optional for core CRUD testing; if keys are empty or dummy values, CRUD operations continue to function normally.
