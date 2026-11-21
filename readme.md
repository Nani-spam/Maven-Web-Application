# SE Lab Internal II — Quick Command Guide

Commands below are for Windows PowerShell unless noted. Replace example names/paths with the ones in the exam question. This is a quick reference for the practice we completed; if the question specifies different tools, ports, repo, or image, follow the question.

## 1. Jenkins: Maven Freestyle build, Poll SCM, artifact archive

### Freestyle job setup

1. Jenkins → **New Item** → choose **Freestyle project**.
2. **Source Code Management → Git**: enter the repository URL; set **Branches to build** to `*/main` (or `*/master`, matching the repo).
3. **Build Triggers → Poll SCM**: enter `H/5 * * * *` to check every five minutes.
4. **Build Steps → Invoke top-level Maven targets**: choose configured Maven and enter only goals, such as `clean package` (do not type `mvn` here).
5. **Post-build Actions → Archive the artifacts**: use the actual output extension, e.g. `target/*.jar` or `target/*.war`.
6. Save → **Build Now**. Check **Console Output** and the build’s **Artifacts** link.

### Verify Poll SCM

- Open job → **Git Polling Log** to see whether Jenkins checked the repository.
- To prove a change triggers a build, commit a harmless change to the selected Git branch and wait for the polling interval.
- **Build Now** is a manual build; it does not prove Poll SCM triggered it.

### Common errors

- `Unknown lifecycle phase "mvn"`: Maven is already invoked by Jenkins. Set Goals to `clean package`, not `mvn package`.
- `target/*.war doesn't match anything`: the build created a JAR. Set archive pattern to `target/*.jar`.
- If a repo has no Maven wrapper, use the configured Maven installation / Maven build step; do not call a missing `mvnw.cmd`.

### Declarative Pipeline used earlier (Windows agent)

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Nani-spam/simple-java-maven-app.git'
            }
        }
        stage('Build and Test') {
            steps {
                bat 'mvn -B clean package'
            }
        }
    }
    post {
        success { archiveArtifacts artifacts: 'target/*.jar', fingerprint: true }
    }
}
```

Use the repo’s real default branch (`master` or `main`) and artifact extension. The Jenkins build above succeeded after using `mvn -B clean package`.

## 2. Docker Compose: WordPress + MySQL

In the folder containing `compose.yaml`, start and check the app:

```powershell
docker compose up -d
docker compose ps
```

Open `http://localhost:8081` (we used 8081 because Jenkins already used 8080). Verify the database health/status with `docker compose ps`.

Stop/remove containers and network while keeping named-volume database data:

```powershell
docker compose down
docker compose up -d
```

Do not add `-v` to `down` when you need to keep the database volume/data.

Useful Docker checks:

```powershell
docker ps
docker ps -a
docker logs CONTAINER_NAME
```

## 3. Nagios in Docker

```powershell
docker run -d -p 8082:80 --name nagios jasonrivers/nagios
docker ps
```

Open `http://localhost:8082`. Default demo login used by this image: `nagiosadmin` / `nagios`.

Stop/remove later if needed:

```powershell
docker stop nagios
docker rm nagios
```

The dashboard opening demonstrates the basic run. Full host/service checks and thresholds require extra Nagios configuration if explicitly asked.

## 4. ngrok tunnel (attempted; skipped after timeout)

Expose the local WordPress port:

```powershell
ngrok http 8081
```

Keep the ngrok terminal running and open the HTTPS **Forwarding** URL printed there. Do not append `:8081` to the public URL. The tunnel stops when ngrok exits. We did not finish validating this task in practice.

## 5. Nginx in Docker

```powershell
docker run -d --name nginx -p 8083:80 nginx
docker ps
```

Open `http://localhost:8083` for the Nginx welcome page. Port 8083 avoids conflicts with Jenkins, WordPress, and Nagios.

## 6. Kubernetes with Minikube (Docker driver)

### Start/check cluster

```powershell
kubectl version --client
minikube version
minikube start --driver=docker
kubectl get nodes
```

Wait until the Minikube node status is `Ready`.

### Deployment: 2 replicas

```powershell
kubectl create deployment library --image=nginx --replicas=2
kubectl get deployments
kubectl get pods
```

Wait until both pods show `1/1 Running`.

### Expose with NodePort Service

```powershell
kubectl expose deployment library --type=NodePort --port=80
kubectl get services
minikube service library
```

`minikube service library` opens the app in a browser. `kubectl get services` shows the assigned NodePort.

### Scale to 5 replicas

```powershell
kubectl scale deployment library --replicas=5
kubectl get pods
```

### Demonstrate self-healing

```powershell
kubectl get pods
kubectl delete pod POD_NAME
kubectl get pods
```

Replace `POD_NAME` with one actual library pod name. Kubernetes creates a replacement to maintain the desired count. Wait until five pods are `Running`.

### Rolling update and rollback

```powershell
kubectl set image deployment/library nginx=nginx:1.27
kubectl rollout status deployment/library
kubectl rollout history deployment/library
```

Rollback to the previous revision:

```powershell
kubectl rollout undo deployment/library
kubectl rollout status deployment/library
```

If `kubectl set image` reports that container `nginx` was not found, inspect the deployment with `kubectl describe deployment library` and use the actual container name.

### Cleanup (only after you are done demonstrating)

```powershell
kubectl delete service library
kubectl delete deployment library
minikube stop
```

## 7. AWS EC2

Skipped in our practice because you said you would use a friend’s instance. **Never paste or upload a `.pem` private key.** For an exam instance, use the access method and credentials authorized by its owner/instructor.

## Quick concepts to say during demonstration

- **Deployment:** keeps the requested number of pod replicas running.
- **Pod:** smallest deployable Kubernetes unit; it can contain one or more containers.
- **Service:** stable network endpoint that routes traffic to matching pods.
- **Scaling:** changes desired replica count (2 → 5 here).
- **Self-healing:** Deployment/ReplicaSet creates a replacement after a pod is deleted or fails.
- **Rolling update:** gradually replaces old pods with the new image version.
- **Rollback:** returns a Deployment to its previous revision.
- **Poll SCM:** Jenkins periodically checks Git for changes; a detected change can start a build.
- **Artifact archiving:** saves build outputs in Jenkins, such as `target/*.jar`.

## Exam reminder

First identify what the question asks. Reuse the command pattern, but match its repository URL, branch, image/tag, port, artifact extension, and requested replica count. Verify results with `docker ps`, `docker compose ps`, Jenkins Console Output, `kubectl get nodes`, `kubectl get deployments`, `kubectl get pods`, and `kubectl get services`.
