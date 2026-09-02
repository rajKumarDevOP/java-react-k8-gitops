 
# Feedback Application — GitOps Deployment

## 📌 Project Overview

This project demonstrates the complete **production deployment workflow** for a Feedback Application consisting of:

* **Frontend:** React + Vite
* **Backend:** Java Spring Boot
* **Containerization:** Docker
* **Container Registry:** AWS ECR
* **Orchestration:** Kubernetes
* **Ingress Controller:** Traefik
* **External Reverse Proxy:** Nginx
* **CI:** GitHub Actions
* **GitOps:** Git repository containing Kubernetes manifests
* **CD:** Argo CD

The deployment follows a **GitOps-based CI/CD architecture**, where application source code, container images, Kubernetes manifests, and deployment synchronization are handled through separate stages.

---

# 🏗️ Architecture

```text
                         ┌──────────────────────┐
                         │      Developer       │
                         │   Git Push / PR      │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │      GitHub          │
                         │ Application Source   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │   GitHub Actions     │
                         │                      │
                         │  1. Build            │
                         │  2. Test             │
                         │  3. Docker Build     │
                         │  4. Docker Push      │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │       AWS ECR        │
                         │                      │
                         │ React Image          │
                         │ Java Image            │
                         └──────────┬───────────┘
                                    │
                                    │ Image Tag
                                    ▼
                         ┌──────────────────────┐
                         │     GitOps Repo      │
                         │                      │
                         │ Kubernetes YAML      │
                         │ Updated Image Tag    │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │       Argo CD        │
                         │                      │
                         │ Detect Git Change    │
                         │ Sync Kubernetes      │
                         └──────────┬───────────┘
                                    │
                                    ▼
                    ┌──────────────────────────────┐
                    │         Kubernetes            │
                    │                              │
                    │  ┌────────────────────────┐  │
                    │  │ React Frontend Pod      │  │
                    │  └────────────────────────┘  │
                    │                              │
                    │  ┌────────────────────────┐  │
                    │  │ Java Backend Pod        │  │
                    │  └────────────────────────┘  │
                    │                              │
                    │  Services + Traefik Ingress │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                         ┌──────────────────────┐
                         │   External Nginx     │
                         │   Reverse Proxy      │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │      End Users       │
                         │                      │
                         │ /feedback/           │
                         │ /feedbackapi/        │
                         └──────────────────────┘
```

---

# 📂 Repository Structure

The project is separated into application and GitOps repositories.

### Application Repository

```text
feedback-frontend/
├── src/
├── public/
├── package.json
├── vite.config.js
├── Dockerfile
└── ...
```

```text
feedback-backend/
├── src/
├── pom.xml
├── mvnw
├── application.properties
├── Dockerfile
└── ...
```

### GitOps Repository

```text
gitops/
└── feedback/
    ├── namespace.yaml
    ├── frontend/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── ingress.yaml
    │
    └── backend/
        ├── configmap.yaml
        ├── secret.yaml
        ├── deployment.yaml
        ├── service.yaml
        ├── middleware.yaml
        └── ingress.yaml
```

---

# 🔄 Complete CI/CD and GitOps Flow

## 1. Developer Push

Development starts with a code change in either:

```text
feedback-frontend
```

or:

```text
feedback-backend
```

A Git push triggers the corresponding GitHub Actions workflow.

---

# 2. GitHub Actions — Build

The CI pipeline performs the following operations:

```text
Git Push
   ↓
Checkout Source
   ↓
Install Dependencies
   ↓
Build Application
   ↓
Run Tests
   ↓
Build Docker Image
```

---

# 3. Frontend Build

The frontend is a **React + Vite** application.

Vite environment variables prefixed with `VITE_` are evaluated during the build.

For the UAT environment:

```text
VITE_API_BASE_URL=https://uat.svkm.ac.in/feedbackapi
```

This variable is supplied **before `npm run build`**.

Example:

```dockerfile
ENV VITE_API_BASE_URL=https://uat.svkm.ac.in/feedbackapi

RUN npm run build
```

The API URL is therefore embedded into the generated frontend JavaScript during the Vite build.

### Important

Because Vite variables are build-time variables:

```text
VITE_API_BASE_URL
        ↓
npm run build
        ↓
dist/
```

Changing the value normally requires rebuilding the frontend image.

---

# 4. Backend Build

The backend is a Java Spring Boot application.

The Docker image uses a multi-stage build:

```text
Java Source
    ↓
Maven Build
    ↓
JAR
    ↓
Java Runtime Image
    ↓
Docker Image
```

Example:

```dockerfile
FROM eclipse-temurin:21-jdk AS build

WORKDIR /app

COPY . .

RUN chmod +x mvnw
RUN ./mvnw clean package -DskipTests


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=build /app/target/*.jar backend.jar

ENTRYPOINT ["java", "-jar", "backend.jar"]
```

---

# 5. Container Security

The backend container runs as a **non-root user**.

The Kubernetes security context is configured accordingly:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  fsGroup: 10001
```

Container privilege escalation is disabled:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

This provides a more secure runtime configuration.

---

# 6. Docker Image Creation

After successful application builds, Docker images are created.

Example:

```text
feedback/frontend:<commit-sha>
feedback/backend:<commit-sha>
```

Using the Git commit SHA as the image tag provides traceability.

Example:

```text
feedback/backend:c14609fb2a53023ef876e74fc2e671c8ec358523
```

This makes it possible to identify exactly which source commit produced a running container.

---

# 7. Push Images to AWS ECR

The CI pipeline authenticates with AWS ECR and pushes the generated images.

Example:

```text
GitHub Actions
      ↓
Docker Build
      ↓
AWS ECR
      ↓
feedback/backend:<commit-sha>
```

The Kubernetes cluster subsequently pulls the image from ECR.

---

# 8. GitOps Manifest Update

After successfully pushing the Docker image, the CI pipeline updates the image tag inside the GitOps repository.

Before:

```yaml
image: 925963414980.dkr.ecr.ap-south-1.amazonaws.com/feedback/backend:v1
```

After:

```yaml
image: 925963414980.dkr.ecr.ap-south-1.amazonaws.com/feedback/backend:c14609fb2a53023ef876e74fc2e671c8ec358523
```

The GitHub Actions workflow then commits and pushes the change:

```bash
git add .

git commit \
  -m "Update feedback backend image to ${IMAGE_TAG}"

git push
```

The GitOps repository therefore becomes the **source of truth for the Kubernetes deployment state**.

---

# 9. Argo CD Synchronization

Argo CD continuously watches the GitOps repository.

The flow is:

```text
GitOps Repository
       ↓
Manifest Change
       ↓
Argo CD Detects Change
       ↓
Application becomes OutOfSync
       ↓
Argo CD Sync
       ↓
Kubernetes Deployment Updated
```

With automated synchronization enabled, Argo CD applies the updated manifests automatically.

---

# 10. Kubernetes Deployment

The application is deployed into the `feedback` namespace.

```bash
kubectl get pods -n feedback
```

Expected components:

```text
feedback-frontend-xxxxx
feedback-backend-xxxxx
```

Services expose the applications internally.

### Frontend

```text
feedback-frontend-service
```

### Backend

```text
feedback-backend-service
```

The backend container listens on:

```text
8088
```

The Kubernetes Service exposes it internally on:

```text
80 → 8088
```

---

# 11. Traefik Ingress

Traefik is used as the Kubernetes Ingress Controller.

The public backend path is:

```text
/feedbackapi/
```

The frontend path is:

```text
/feedback/
```

The backend application itself does not require the `/feedbackapi` prefix.

Therefore Traefik uses a `StripPrefix` Middleware.

```text
/feedbackapi/login
        ↓
Traefik
        ↓
StripPrefix
        ↓
/login
        ↓
Spring Boot
```

Example Middleware:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: feedbackapi-strip-prefix
  namespace: feedback
spec:
  stripPrefix:
    prefixes:
      - /feedbackapi
```

The backend Ingress references this Middleware.

---

# 12. External Nginx

An external Nginx server acts as the public reverse proxy.

Frontend:

```nginx
location /feedback/ {
    proxy_pass http://127.0.0.1:<frontend-nodeport>;
}
```

Backend:

```nginx
location /feedbackapi/ {
    proxy_pass http://127.0.0.1:<backend-ingress-nodeport>;

    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
}
```

The important point is that the backend `proxy_pass` preserves `/feedbackapi` so that Traefik can process and strip the prefix.

---

# 🌐 Final Request Flow

## Frontend

```text
https://uat.svkm.ac.in/feedback/
                ↓
          External Nginx
                ↓
             Traefik
                ↓
      Frontend Service
                ↓
        React Frontend Pod
```

## Backend API

```text
https://uat.svkm.ac.in/feedbackapi/login
                ↓
          External Nginx
                ↓
             Traefik
                ↓
       StripPrefix Middleware
                ↓
              /login
                ↓
      Backend Service :80
                ↓
        Java Pod :8088
```

---

# 🩺 Health Checks

Kubernetes probes are configured to verify application availability.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 8088

livenessProbe:
  httpGet:
    path: /
    port: 8088
```

The application startup time was considered when configuring the probe delays.

A production implementation can use Spring Boot Actuator endpoints:

```text
/actuator/health/readiness
/actuator/health/liveness
```

These endpoints can be explicitly permitted through Spring Security.

---

# 🔐 Configuration Management

Application configuration is separated from the container image using Kubernetes ConfigMaps and Secrets.

Example ConfigMap:

```yaml
data:
  SERVER_PORT: "8088"
  DB_URL: "jdbc:mysql://<database-host>:3306/usermanagement"
  CORS_ALLOWED_ORIGINS: "https://uat.svkm.ac.in"
```

Sensitive credentials are stored in Kubernetes Secrets rather than directly inside the Deployment manifest.

---

# 🛠️ Deployment Verification

After Argo CD synchronization, verify the deployment.

### Check pods

```bash
kubectl get pods -n feedback
```

### Check services

```bash
kubectl get svc -n feedback
```

### Check ingress

```bash
kubectl get ingress -n feedback
```

### Check Traefik Middleware

```bash
kubectl get middleware -n feedback
```

### Check deployment

```bash
kubectl get deployment -n feedback
```

### Check pod logs

```bash
kubectl logs -n feedback <pod-name>
```

### Enter a container

```bash
kubectl exec -it -n feedback <pod-name> -- /bin/sh
```

---

# 🔍 Troubleshooting Performed During Deployment

Several production deployment issues were identified and resolved during implementation.

## 1. Docker User / Kubernetes Security Context

The container initially used UID/GID values that conflicted with the base image.

The solution was to use an explicit application UID/GID and match it in Kubernetes.

```text
Docker USER
      ↓
UID/GID
      ↓
Kubernetes runAsUser/runAsGroup
```

---

## 2. Log File Permission

Spring Boot Logback initially failed with:

```text
FileNotFoundException:
logs/feedback-backend.log
Permission denied
```

The solution was to ensure that the application user owns the log directory:

```dockerfile
RUN mkdir -p /app/logs \
    && chown -R 10001:10001 /app
```

---

## 3. Spring Boot Datasource Configuration

The backend initially failed because Spring Boot could not determine the datasource configuration.

The datasource environment variables and MySQL configuration were corrected.

After correction, the application successfully started on:

```text
Tomcat started on port 8088
```

---

## 4. Kubernetes Readiness/Liveness Probes

The application startup time was longer than the initial liveness configuration.

Additionally, Spring Security returned:

```text
403 Forbidden
```

for the probe endpoint.

The probe configuration and application security configuration were reviewed to ensure Kubernetes health checks receive an appropriate response.

---

## 5. GitHub Actions Shell Syntax

The GitOps commit step initially failed with:

```text
syntax error: unexpected end of file
```

The issue was a missing:

```bash
fi
```

after the shell `if` block.

The corrected pipeline successfully committed and pushed the updated image tag.

---

## 6. Traefik Path Rewriting

The backend public URL uses:

```text
/feedbackapi/
```

while Spring Boot expects:

```text
/login
/users
/etc.
```

A Traefik `StripPrefix` Middleware was introduced:

```text
/feedbackapi/login
        ↓
StripPrefix
        ↓
/login
```

This allowed the public URL structure to remain unchanged while keeping the backend application paths clean.

---

# 📊 Deployment Responsibilities

| Component         | Responsibility                |
| ----------------- | ----------------------------- |
| GitHub            | Application source code       |
| GitHub Actions    | CI/CD automation              |
| Docker            | Container image creation      |
| AWS ECR           | Container image registry      |
| GitOps Repository | Kubernetes desired state      |
| Argo CD           | GitOps synchronization        |
| Kubernetes        | Container orchestration       |
| Traefik           | Kubernetes ingress/routing    |
| Middleware        | `/feedbackapi` prefix removal |
| Nginx             | External reverse proxy        |
| ConfigMap         | Non-sensitive configuration   |
| Secret            | Sensitive configuration       |

---

# 🚀 Production Deployment Lifecycle

The complete production lifecycle is:

```text
1. Developer writes code
          ↓
2. Git push
          ↓
3. GitHub Actions triggered
          ↓
4. Application build
          ↓
5. Tests
          ↓
6. Docker image build
          ↓
7. Image tagged with Git SHA
          ↓
8. Image pushed to AWS ECR
          ↓
9. GitOps manifest image tag updated
          ↓
10. GitOps commit pushed
          ↓
11. Argo CD detects Git change
          ↓
12. Argo CD synchronizes Kubernetes
          ↓
13. Kubernetes performs rolling deployment
          ↓
14. Readiness/Liveness checks
          ↓
15. Traefik routes traffic
          ↓
16. External Nginx exposes application
          ↓
17. Production application available
```

---

# ✅ Final Production State

The deployment successfully provides:

* ✅ React frontend containerization
* ✅ Java Spring Boot backend containerization
* ✅ Non-root container execution
* ✅ Kubernetes security context
* ✅ AWS ECR image storage
* ✅ Git SHA based image versioning
* ✅ GitHub Actions CI/CD
* ✅ GitOps-based Kubernetes deployment
* ✅ Argo CD synchronization
* ✅ Kubernetes Services
* ✅ Traefik Ingress
* ✅ Traefik StripPrefix Middleware
* ✅ External Nginx reverse proxy
* ✅ Frontend `/feedback/` routing
* ✅ Backend `/feedbackapi/` routing
* ✅ Backend path rewriting
* ✅ Kubernetes health checks
* ✅ ConfigMap-based configuration
* ✅ Secret-based sensitive configuration
* ✅ Rolling deployment strategy

---

# 🎯 Key DevOps Principles Demonstrated

This implementation demonstrates the following production DevOps practices:

### Infrastructure as Code

Kubernetes resources are maintained as version-controlled YAML manifests.

### GitOps

Git is the source of truth for the desired Kubernetes state.

### Immutable Deployments

Each container image is identified by a unique Git commit SHA.

### Separation of CI and CD

```text
CI
GitHub Actions
     ↓
Build + Test + Image

CD
Argo CD
     ↓
GitOps + Kubernetes
```

### Container Security

Applications run as non-root users with restricted Linux capabilities.

### Traceability

```text
Git Commit
    ↓
Docker Image SHA
    ↓
GitOps Manifest
    ↓
Argo CD
    ↓
Kubernetes Pod
```

This provides an auditable path from source code to the running production workload.

---

# 🏁 Conclusion

The Feedback Application has been successfully deployed using a complete **containerized, Kubernetes-based GitOps architecture**.

The final solution separates application development, container image creation, image storage, deployment configuration, and Kubernetes synchronization into clearly defined stages.

This architecture provides a repeatable and traceable deployment process where a developer's Git commit can be followed all the way to the running production workload.

```text
Developer Commit
       ↓
GitHub Actions
       ↓
Docker Image
       ↓
AWS ECR
       ↓
GitOps Repository
       ↓
Argo CD
       ↓
Kubernetes
       ↓
Traefik
       ↓
Nginx
       ↓
Production
```
