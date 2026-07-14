# Multibranch-Pipeline

A hands-on Jenkins Multibranch Pipeline project. Jenkins automatically discovers every branch in this repository, and each branch with a `Jenkinsfile` gets its own independent CI pipeline that builds a Docker image for the app.

![GitHub repository overview](screenshots/github-repo-readme.png)

## Project structure

```
Multibranch-Pipeline/
├── Jenkinsfile     # Pipeline definition (declarative, Groovy)
├── Dockerfile      # Container image definition for the app
└── app.js          # Simple Node.js HTTP server
```

## What the app does

A minimal Node.js server that responds with a greeting on port 3000.

```javascript
const http = require('http');
http.createServer((req, res) => {
  res.end('Hello from Jenkins multibranch pipeline!\n');
}).listen(3000);
console.log('Server running on port 3000');
```

## What the pipeline does

Defined in `Jenkinsfile`, run automatically per branch:

1. **Checkout** — pulls the branch's code via `checkout scm`
2. **Build Docker Image** — runs `docker build -t my-app:<branch-name> .`

![Jenkinsfile viewed on GitHub](screenshots/jenkinsfile-github.png)

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = "my-app:${env.BRANCH_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }
    }

    post {
        always {
            echo "Pipeline finished for branch: ${env.BRANCH_NAME}"
        }
    }
}
```

> A "Run Container" test stage (spinning up the built image and curling it) was
> originally included but removed after hitting Docker-networking issues
> between the Jenkins container and the app container. See **Known issues**
> below if you want to add it back.

## Environment

- Jenkins running as a Docker container (`jenkins/jenkins:lts`) on a Ubuntu VM inside VirtualBox
- Jenkins UI accessible at `http://localhost:8080` (from inside the VM)
- Docker available to Jenkins via the host's Docker socket (Docker-outside-of-Docker setup)
- Source hosted on GitHub, authenticated via a Personal Access Token

## Setup guide

### 1. Prerequisites

- A VM (VirtualBox) with Docker installed
- A GitHub repository containing `Jenkinsfile`, `Dockerfile`, and `app.js`

### 2. Run Jenkins as a container with Docker access

Jenkins needs access to the host's Docker daemon to build images. This is done by mounting the Docker socket into the Jenkins container:

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v /opt/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

Install the Docker CLI inside the container (the daemon itself stays on the host):

```bash
docker exec -u root -it jenkins bash
apt-get update && apt-get install -y docker.io
usermod -aG docker jenkins
exit
docker restart jenkins
```

> **GID mismatch gotcha:** the `docker` group inside the container may have a
> different GID than the one that owns `/var/run/docker.sock` on the host,
> which causes `permission denied` errors even after adding `jenkins` to the
> group. Fix by matching them:
> ```bash
> getent group docker          # check host GID
> docker exec -u root jenkins groupmod -g <host_GID> docker
> docker restart jenkins
> ```

Once it's running, confirm Jenkins is reachable at `http://localhost:8080`:

![Jenkins dashboard](screenshots/jenkins-dashboard.png)

### 3. Install required Jenkins plugins

- Git plugin
- GitHub plugin
- Docker Pipeline
- Pipeline: Multibranch (usually bundled)

### 4. Add GitHub credentials

Anonymous GitHub API access is limited to 60 requests/hour, which causes
Jenkins to sleep repeatedly during branch scans. Adding a token raises this
to 5,000 requests/hour.

1. GitHub → **Settings → Developer settings → Personal access tokens (classic)**
2. Generate a token with the **repo** scope

   ![GitHub personal access token scopes](screenshots/github-pat-token.png)

3. Jenkins → **Manage Jenkins → Credentials → (global) → Add Credentials**
   - Kind: `Secret text` (or `Username with password` if your GitHub source requires it)
   - Secret / Password: the generated token
   - ID: `github-token`

   ![Add Secret text credential in Jenkins](screenshots/add-secret-text.png)

### 5. Create the Multibranch Pipeline job

1. Jenkins dashboard → **New Item** → name it → select **Multibranch Pipeline**

   ![Creating a new Multibranch Pipeline item](screenshots/new-item-multibranch.png)

2. **Branch Sources → Add source → GitHub** (or Git)
   - Repository URL: your repo's HTTPS URL
   - Credentials: select the credential created above
3. **Build Configuration → Script Path**: `Jenkinsfile`
4. **Scan Multibranch Pipeline Triggers**: enable periodic scanning (e.g. every 1 minute) for local testing
5. Save

   ![Branch Sources configuration with credentials attached](screenshots/branch-sources-config.png)

Jenkins scans the repo, finds every branch containing a `Jenkinsfile`, and creates a pipeline job for each one automatically.

## Testing it

```bash
git checkout -b dev
echo "// dev branch tweak" >> app.js
git commit -am "Test change on dev branch"
git push -u origin dev
```

Trigger a rescan in Jenkins (or wait for the periodic scan) — a new job appears for `dev` with its own independent build history.

A successful build shows a green Stage View, with each stage timed individually:

![Successful main branch build with stage view](screenshots/main-branch-build-success.png)

## Known issues / things to revisit

- **`docker: not found` in pipeline steps** — happened when the Docker socket
  wasn't mounted into the Jenkins container, or when the container was
  recreated after installing the Docker CLI (wiping the install since it
  wasn't in a persistent volume). Fix: mount the socket first, then install
  the CLI inside the running container, in that order.
- **Container name conflicts / `localhost` curl failures** — the removed
  "Run Container" stage tried to `curl http://localhost:3001` from inside the
  Jenkins pipeline shell to test the app container. Because Jenkins itself
  runs in a separate container talking to the host's Docker daemon via the
  socket (Docker-outside-of-Docker), `localhost` inside the Jenkins shell
  isn't guaranteed to reach a port published by a *different* container the
  same way it would on the host. A more reliable approach would be checking
  the container via `docker exec <container> curl localhost:3000` (inside
  its own network namespace) or curling the host's actual IP instead of
  `localhost`.
- **Jenkins container not restarting after VM reboot** — fixed by setting a
  restart policy:
  ```bash
  docker update --restart=unless-stopped jenkins
  ```

## Useful commands

| Task | Command |
|---|---|
| Check Jenkins container status | `docker ps` |
| View Jenkins logs | `docker logs jenkins --tail 100` |
| Restart Jenkins | `docker restart jenkins` |
| Confirm Docker bridge works | `docker exec jenkins docker ps` |