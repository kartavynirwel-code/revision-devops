# Day 4 — CI/CD (Jenkins + ArgoCD) Revision Notes

## 1. Jenkinsfile Stages

### Declarative Pipeline Structure
```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "myapp"
        REGISTRY = "docker.io/myuser"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/user/repo.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SAST Scan') {
            steps {
                sh 'sonar-scanner -Dsonar.projectKey=myapp'
            }
        }

        stage('SCA Scan') {
            steps {
                sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL .'
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh """
                  docker build -t ${REGISTRY}/${DOCKER_IMAGE}:${BUILD_NUMBER} .
                  docker push ${REGISTRY}/${DOCKER_IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        stage('Update Manifest') {
            steps {
                // update image tag in GitOps repo, ArgoCD picks it up
                sh './update-k8s-manifest.sh ${BUILD_NUMBER}'
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed — notify team'
        }
        always {
            cleanWs()
        }
    }
}
```

### Key Concepts
- **`agent`**: Where the pipeline runs — `any`, a specific label, or a Docker container (`agent { docker 'maven:3.9' }`).
- **`stages` / `stage` / `steps`**: Hierarchy — a pipeline has stages, each stage has steps (actual shell commands or Jenkins DSL calls).
- **`post` block**: Runs regardless of stage outcome — `always`, `success`, `failure`, `unstable` conditions. Common use: cleanup, notifications (Slack/email).
- **`environment` block**: Defines env vars available to all stages — also where you'd reference Jenkins **credentials** (`credentials('docker-hub-creds')`) without hardcoding secrets.
- **Declarative vs Scripted**: Declarative (`pipeline { }` block, shown above) is structured and easier to read/lint. Scripted (`node { }` block with raw Groovy) is more flexible but harder to maintain — most modern pipelines use Declarative.

## 2. SAST/SCA Webhook Flow (DevSecOps pipeline)
**Typical shift-left security flow in a pipeline**:
```
Code push → Webhook triggers Jenkins →
  Checkout → Build →
  SAST (SonarQube/Semgrep - scans YOUR source code for bugs/vulnerabilities) →
  SCA (Trivy/Snyk/OWASP Dependency-Check - scans DEPENDENCIES for known CVEs) →
  Build image → Container Image Scan (Trivy on the built image) →
  Push to registry (only if all gates pass) →
  Update GitOps manifest repo →
  ArgoCD auto-syncs to cluster
```
- **SAST (Static Application Security Testing)**: Analyzes your own source code without running it — catches things like SQL injection patterns, hardcoded secrets, insecure crypto usage.
- **SCA (Software Composition Analysis)**: Scans your `pom.xml`/`package.json`/etc. dependency tree against CVE databases — catches vulnerable third-party libraries.
- **Interview point**: Explain "shift-left" — running these scans early in the pipeline (before deploy) instead of finding vulnerabilities in production. Use `--exit-code 1` type flags so the pipeline **fails the build** on HIGH/CRITICAL findings — this is what makes it a real gate, not just a report.
- **Webhook trigger**: GitHub/GitLab webhook calls a Jenkins endpoint (`/github-webhook/`) on every push, which triggers the pipeline automatically instead of manual/polling builds.

## 3. ArgoCD — GitOps Continuous Delivery

### Core Idea
- ArgoCD continuously watches a **Git repo** (the "source of truth" for desired state) and syncs the live Kubernetes cluster state to match it.
- **Pull-based model**: Unlike Jenkins pushing changes to the cluster, ArgoCD *pulls* from Git — cluster credentials never need to leave the cluster, which is a security win.

### Sync Policies
- **Manual sync**: Changes detected in Git, but ArgoCD waits for a human to click "Sync" in the UI/CLI.
- **Automated sync** (`syncPolicy.automated`): ArgoCD applies changes to the cluster automatically as soon as it detects drift from Git.
  - `prune: true` — deletes resources from the cluster that were removed from Git.
  - `selfHeal: true` — if someone manually changes something in the cluster (`kubectl edit`), ArgoCD reverts it back to match Git automatically. This is the core GitOps guarantee: **cluster state always matches Git, no manual drift survives**.

### App-of-Apps Pattern
- Instead of managing N individual ArgoCD `Application` resources manually, you create **one parent Application** whose source is a Git folder containing multiple child `Application` manifests.
- ArgoCD syncs the parent, which in turn creates/manages all the child Applications.
- **Why it matters**: Scales GitOps to many microservices/environments cleanly — add a new service by just adding a new Application YAML to the folder, parent app picks it up automatically.

### CLI Commands
```bash
argocd login <argocd-server>
argocd app list
argocd app get <app-name>                  # status, sync state, resources
argocd app sync <app-name>                 # manual sync
argocd app diff <app-name>                 # see drift between Git and live state
argocd app history <app-name>              # revision history
argocd app rollback <app-name> <revision>  # rollback to a previous Git revision
argocd app set <app-name> --sync-policy automated --auto-prune --self-heal
```

---

## Quick Self-Test (do this without looking)
1. What's the difference between a SAST scan and an SCA scan, and where does each fit in the pipeline?
2. Why does ArgoCD's pull-based model reduce security risk compared to Jenkins pushing directly to a cluster?
3. If someone manually edits a Deployment in the cluster with `kubectl edit`, what happens with `selfHeal: true` enabled?
4. What problem does the App-of-Apps pattern solve when you have 15 microservices to manage in ArgoCD?
