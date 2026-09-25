---
noteId: "505f5c20b91311f1bc33854b24ee31c6"
tags: []

---

# Local DevSecOps CI/CD Pipeline — Vite + React

A hands-on DevSecOps practice project that automates a Vite + React application's flow from GitHub source code through Jenkins CI, SonarQube analysis, Quality Gate, Docker image creation, deployment, and deployment verification.

This project was implemented locally using Docker Desktop on Windows. No AWS EC2 instance was required.

## 1. Final Pipeline

```text
GitHub Push
    |
    v
Jenkins
    |
    +--> Checkout
    |
    +--> npm ci --include=optional
    |
    +--> Vite Production Build
    |
    +--> SonarQube Analysis
    |
    +--> Quality Gate
    |
    +--> Docker Build
    |
    +--> Docker Deploy
    |
    +--> Health / HTTP Verification
    |
    v
React Application
localhost:8080
```

## 2. Technologies

| Technology | Purpose |
|---|---|
| React + Vite | Frontend application |
| GitHub | Source control |
| Jenkins | CI/CD automation |
| SonarQube | Code quality/security analysis |
| Docker | Containerization and deployment |
| Nginx | Serves the production Vite build |
| Docker Desktop | Local container runtime |
| ngrok | Public tunnel for GitHub webhook during local practice |

## 3. Local Ports

```text
React application : http://localhost:8080
Jenkins           : http://localhost:8081
SonarQube         : http://localhost:9000
Jenkins agent     : 50000
```

Jenkins uses host port `8081` because the frontend already uses `8080`.

## 4. Jenkins Configuration

The pipeline was created directly in:

```text
Jenkins
→ Job
→ Configure
→ Pipeline
→ Pipeline script
```

A Jenkinsfile stored in GitHub was not used.

The main stages are:

```text
Checkout
Install Dependencies
Build Vite App
SonarQube Analysis
Quality Gate
Docker Build
Docker Deploy
Deployment Verification
```

## 5. Dockerfile

The project uses a multi-stage Docker build:

```dockerfile
FROM node:22-alpine AS build

WORKDIR /app

COPY package*.json ./

RUN npm ci --include=optional

COPY . .

RUN npm run build

FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

HEALTHCHECK --interval=10s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost/ || exit 1
```

The first stage creates the Vite production build. The second stage serves `dist/` using Nginx.

## 6. Initial Docker Test

Before Jenkins deployment was automated, the image was tested manually:

```bash
docker build -t my-app:1.0 .
```

```bash
docker run -d --name my-app -p 8080:80 my-app:1.0
```

Application:

```text
http://localhost:8080
```

## 7. Jenkins Container

Jenkins was run using:

```text
jenkins/jenkins:lts-jdk17
```

A persistent volume was used:

```text
jenkins_home:/var/jenkins_home
```

This preserved Jenkins jobs, plugins, credentials, and configuration when the container was recreated.

Jenkins was also connected to:

```text
devops-network
```

and Docker socket access was configured so Jenkins could build and run Docker images.

## 8. Node.js Configuration

Jenkins was configured with Node.js:

```text
Node.js v22.20.0
npm 10.9.3
```

Dependencies are installed with:

```bash
npm ci --include=optional
```

## 9. SonarQube Integration

SonarQube was run with:

```text
sonarqube:lts-community
```

and exposed at:

```text
http://localhost:9000
```

A SonarQube token was configured in Jenkins.

The scanner was configured in Jenkins and added to PATH using:

```groovy
withSonarQubeEnv('SonarQube') {
    withEnv([
        "PATH+SONAR=${tool 'SonarScanner'}/bin"
    ]) {
        sh '''
            sonar-scanner               -Dsonar.projectKey=my-project               -Dsonar.projectName=my-project               -Dsonar.sources=src
        '''
    }
}
```

## 10. Quality Gate

After analysis, Jenkins waits for the SonarQube Quality Gate:

```groovy
stage('Quality Gate') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

If the Quality Gate fails, Jenkins stops the pipeline.

### SonarQube Result

The tested project reached:

```text
Quality Gate: PASSED
Bugs: 0
Vulnerabilities: 0
Security Hotspots: 0
Reliability: A
Security: A
Maintainability: A
```

There was one code smell in the tested analysis.

## 11. SonarQube Webhook

### Problem

Jenkins initially timed out at:

```text
waitForQualityGate
```

even though SonarQube had completed the analysis.

### Cause

SonarQube was not sending the Quality Gate result back to Jenkins.

### Solution

A SonarQube webhook was configured to use:

```text
/sonarqube-webhook/
```

After configuring the webhook, Jenkins successfully received the Quality Gate result.

## 12. Docker Integration with Jenkins

Initially:

```text
docker: not found
```

### Solution

The Docker CLI was installed inside the Jenkins container.

The Docker socket was mounted:

```text
/var/run/docker.sock
```

Jenkins was then able to communicate with the Docker daemon.

### Permission Problem

After installing the CLI, the pipeline produced:

```text
permission denied while trying to connect to the Docker daemon socket
```

The socket was checked:

```bash
ls -ln /var/run/docker.sock
```

It was owned by:

```text
root:root
```

while Jenkins was running as:

```text
uid=1000(jenkins)
```

The Jenkins user was given the required group access and Jenkins was restarted.

The following test then worked:

```bash
docker exec jenkins docker ps
```

This confirmed that Jenkins could control Docker.

> Note: Giving Jenkins root-level Docker socket access is acceptable for this local learning setup, but a production environment should use a safer Docker build-agent/socket configuration.

## 13. Docker Build Stage

The Jenkins pipeline uses:

```groovy
stage('Docker Build') {
    steps {
        dir('my-project') {
            sh 'docker build -t my-app:1.0 .'
        }
    }
}
```

Successful result:

```text
Successfully tagged my-app:1.0
```

## 14. Docker Deployment

The deployment stage automatically replaces the old frontend container:

```groovy
stage('Docker Deploy') {
    steps {
        sh '''
            docker stop my-app || true
            docker rm my-app || true

            docker run -d               --name my-app               -p 8080:80               my-app:1.0
        '''
    }
}
```

This removes the need to manually stop, remove, and run the frontend container after every successful build.

## 15. Deployment Verification

The deployment should be verified before Jenkins reports success.

Example:

```groovy
stage('Deployment Verification') {
    steps {
        sh '''
            echo "Waiting for application to start..."
            sleep 10

            STATUS=$(docker inspect -f '{{.State.Status}}' my-app)

            if [ "$STATUS" != "running" ]; then
                echo "ERROR: Container is not running"
                docker logs my-app
                exit 1
            fi

            echo "Container is running."

            HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://host.docker.internal:8080)

            if [ "$HTTP_STATUS" != "200" ]; then
                echo "ERROR: Application returned HTTP $HTTP_STATUS"
                docker logs my-app
                exit 1
            fi

            echo "Application is healthy."
            echo "HTTP Status: $HTTP_STATUS"
        '''
    }
}
```

Because Jenkins is running inside Docker, `localhost` inside Jenkins refers to the Jenkins container. Therefore the host route is used for the local frontend verification.

## 16. Automatic CI Trigger

The next automation layer is:

```text
Developer
   |
   | git push
   v
GitHub
   |
   | webhook
   v
Jenkins
   |
   v
Pipeline starts automatically
```

In Jenkins:

```text
Job
→ Configure
→ Build Triggers
→ GitHub hook trigger for GITScm polling
```

For local Jenkins, GitHub cannot directly call:

```text
http://localhost:8081
```

because that address is only available on the local machine.

For local practice, a public tunnel such as ngrok can expose Jenkins:

```text
GitHub
   ↓
ngrok public URL
   ↓
localhost:8081
   ↓
Jenkins
```

The GitHub webhook endpoint is:

```text
/github-webhook/
```

## 17. Problems Faced and Solutions

### Issue 1 — `node.exe: not found` during Docker build

Error:

```text
/app/node_modules/.bin/vite: exec: line 33: node.exe: not found
```

Cause:

Dependencies were generated in the Windows environment and were not suitable for the Linux Docker build environment.

Solution:

Install dependencies inside the Linux Docker build stage instead of copying local `node_modules`:

```dockerfile
COPY package*.json ./
RUN npm ci --include=optional
COPY . .
RUN npm run build
```

---

### Issue 2 — `npm: not found` in Jenkins

Cause:

Node.js was not configured as a Jenkins tool.

Solution:

Configure NodeJS in:

```text
Manage Jenkins → Tools → NodeJS installations
```

and use:

```groovy
tools {
    nodejs 'NodeJS-22'
}
```

---

### Issue 3 — Node.js engine warnings

Some packages required a newer Node.js version than `v22.0.0`.

Solution:

Use:

```text
Node.js v22.20.0
npm 10.9.3
```

---

### Issue 4 — `libatomic.so.1` missing

Error:

```text
node: error while loading shared libraries:
libatomic.so.1: cannot open shared object file
```

Cause:

The selected Node.js runtime required a system library unavailable in the Jenkins environment.

Solution:

Correct the Node.js installation/version used by Jenkins.

---

### Issue 5 — Rolldown native binding missing

Error:

```text
Cannot find native binding
```

including:

```text
@rolldown/binding-linux-x64-gnu
```

Cause:

Required optional native dependencies were not installed.

Solution:

Use:

```bash
npm ci --include=optional
```

This fixed the Vite/Rolldown build.

---

### Issue 6 — `sonar-scanner: not found`

Cause:

The SonarScanner installation existed but its `bin` directory was not on PATH.

Solution:

```groovy
withEnv([
    "PATH+SONAR=${tool 'SonarScanner'}/bin"
])
```

---

### Issue 7 — Invalid Jenkins tool types

Errors included:

```text
Invalid tool type "sonarQube"
```

and:

```text
Invalid tool type "sonarRunner"
```

Cause:

The Pipeline `tools` syntax did not match the installed Jenkins plugin's supported tool types.

Solution:

Use:

```groovy
tool 'SonarScanner'
```

and add the scanner's `bin` directory to PATH.

---

### Issue 8 — Quality Gate timeout

Error:

```text
Cancelling nested steps due to timeout
Finished: ABORTED
```

Cause:

Jenkins was waiting for SonarQube, but the SonarQube webhook was not configured correctly.

Solution:

Configure the SonarQube webhook to:

```text
/sonarqube-webhook/
```

Result:

```text
Quality Gate: PASSED
```

---

### Issue 9 — Docker CLI unavailable

Error:

```text
exec: "docker": executable file not found in $PATH
```

Solution:

Install Docker CLI inside Jenkins and provide access to the Docker socket.

---

### Issue 10 — Docker socket permission denied

Error:

```text
permission denied while trying to connect to the Docker daemon socket
```

Cause:

The Jenkins user could not access `/var/run/docker.sock`.

Solution:

Grant the Jenkins user the required Docker socket group access and restart Jenkins.

Verification:

```bash
docker exec jenkins docker ps
```

Result:

Jenkins successfully listed the running Docker containers.

---

### Issue 11 — Port conflict

The frontend already used:

```text
8080
```

Solution:

Jenkins was exposed on:

```text
8081
```

Final local ports:

```text
Frontend  → 8080
Jenkins   → 8081
SonarQube → 9000
```

## 18. Final Outcome

The project successfully implemented a local DevSecOps CI/CD workflow containing:

- GitHub source control
- Jenkins Pipeline scripting
- Node.js/npm build automation
- Vite production builds
- SonarQube code analysis
- SonarQube Quality Gate
- SonarQube webhook integration
- Docker CLI integration
- Jenkins-to-Docker daemon integration
- Docker image creation
- Automated container deployment
- Deployment verification
- Local CI/CD infrastructure without AWS EC2

The final deployment flow is:

```text
GitHub
   ↓
Jenkins
   ↓
npm ci
   ↓
Vite Build
   ↓
SonarQube
   ↓
Quality Gate
   ↓
Docker Build
   ↓
Docker Deploy
   ↓
Health / HTTP Verification
   ↓
localhost:8080
```

## 19. Future Improvements

Planned improvements:

1. Complete GitHub webhook automatic triggering.
2. Use Jenkins `BUILD_NUMBER` for Docker image tags instead of always using `1.0`.
3. Add automated frontend tests.
4. Use Docker health status instead of only checking container status.
5. Add deployment rollback on failed verification.
6. Push images to Docker Hub or another registry.
7. Add Kubernetes deployment.
8. Add Prometheus/Grafana monitoring.
9. Add proper secrets management.
10. Deploy the same pipeline to a cloud environment such as AWS.

## 20. Learning Outcomes

This project provided hands-on experience with:

- CI/CD pipeline design
- Jenkins Pipeline scripting
- GitHub integration
- GitHub webhooks
- Static code analysis
- SonarQube Quality Gates
- Docker multi-stage builds
- Docker networking
- Docker socket integration
- Container deployment
- Deployment verification
- Troubleshooting Linux/Node.js/Docker environments
- Local DevSecOps automation

The main learning outcome was understanding the complete path:

```text
Git Commit
    ↓
CI Build
    ↓
Code Quality/Security Analysis
    ↓
Quality Gate
    ↓
Containerization
    ↓
Deployment
    ↓
Application Verification
```
