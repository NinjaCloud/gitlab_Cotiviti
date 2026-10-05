# Lab 11 : Auto DevOps & Kubernetes Integration

**Tools:** A web browser, a GitLab.com Free account, a free Docker Hub account, and the free Killercoda Kubernetes playground. **Nothing to install on your computer.**
**Time:** ~90 minutes
**Goal:** See what Auto DevOps automates, build a Docker image in a pipeline and store it in Docker Hub and GitLab's Container Registry, then connect GitLab to a Kubernetes cluster and deploy that image.

---

## What You Will Learn

- What **Auto DevOps** does for you out of the box
- How to build a Docker image in a pipeline and store it in a **container registry**
- How to connect GitLab to a **Kubernetes cluster** using the **GitLab agent for Kubernetes**
- How to deploy a stored image to the cluster from a pipeline

## Words to Know

| Word | Simple meaning |
|---|---|
| Docker image | A packaged app (code plus everything it needs to run) |
| Container registry | A storage place for Docker images (Docker Hub, GitLab Container Registry) |
| Tag | A label on an image version, for example `latest` or a commit ID |
| Kubernetes (K8s) | A system that runs and manages containers across machines |
| Cluster | A group of machines running Kubernetes |
| Pod / Deployment / Service | A running container group / a rule that keeps pods running / a stable network address for pods |
| `kubectl` | The command-line tool for talking to Kubernetes |
| GitLab agent for Kubernetes | A small program inside your cluster that makes an **outbound** connection to GitLab, so GitLab can work with the cluster |
| Auto DevOps | A ready-made, automatic CI/CD pipeline GitLab can run when you have no `.gitlab-ci.yml` |

---

## Free Tier: What Works and What Does Not

| Feature | On GitLab Free? | In this lab |
|---|---|---|
| Auto DevOps (build, test, code quality, SAST, secret detection) | Yes | Hands-on (Step 1) |
| Auto Dependency Scanning, Container Scanning, DAST | **No (Ultimate)** | Mentioned only |
| GitLab Container Registry | Yes | Hands-on |
| GitLab agent for Kubernetes | Yes | Hands-on |
| Docker Hub public repositories | Yes (free account) | Hands-on |

## About Killercoda (Please Read)

- Killercoda gives you a **free Kubernetes cluster in your browser**, with a terminal. It is temporary: the session ends after about an hour and everything in it is deleted. The GitLab parts of this lab are saved, so you can start a new Killercoda session and redo Step 6 if you run out of time.
- Because Killercoda is a private, short-lived cluster, **GitLab cannot call into it**. The GitLab agent solves this by connecting **out** from the cluster to GitLab, so it works fine.
- The terminal accepts paste with **Ctrl+Shift+V** (Windows/Linux) or **Cmd+V** (Mac).

---

## Step 0: Create Your Accounts

1. **GitLab:** sign in at https://gitlab.com. New accounts may be asked to verify before using shared runners. Do this before starting.
2. **Docker Hub:** sign up for free at https://hub.docker.com. Remember your **username**.
3. **Killercoda:** open https://killercoda.com and (optionally) sign in with GitHub or Google to get longer sessions.

---

## Step 1: Auto DevOps (Try and Observe)

Auto DevOps only runs when a project has **no** `.gitlab-ci.yml`, so we use a separate small project.

1. In GitLab click **New project > Create blank project**. Name: `auto-devops-lab`. **Private**. Tick **Initialize repository with a README**. Click **Create project**.
2. Click **+ > New file**. File name: `index.html`:

```html
<!DOCTYPE html>
<html>
  <body>
    <h1>Hello from Auto DevOps</h1>
  </body>
</html>
```

Commit with message `Add index.html`.

3. Create another new file named `Dockerfile`:

```
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
```

Commit with message `Add Dockerfile`.

4. Go to **Settings > CI/CD** and expand **Auto DevOps**.
5. Tick **Default to Auto DevOps pipeline** and click **Save changes**.
6. Go to **Build > Pipelines**. If no pipeline started, make a tiny edit to `README.md` and commit.
7. Open the pipeline and look at the jobs.

**You should see:** an automatic pipeline with a **build** job (it builds your Dockerfile into an image) and **test** jobs such as code quality, SAST, and secret detection. The exact job names depend on your GitLab version.

You will **not** see working deploy jobs. Auto DevOps deploys need a Kubernetes cluster and a public domain name, and we have neither here. That is expected: the goal is to see what it automates.

8. Go to **Deploy > Container Registry**.

**You should see:** the image that Auto Build created, stored in your project's own registry.

### What Auto DevOps Automates

| Stage | What it does | Free tier? |
|---|---|---|
| Auto Build | Builds a Docker image (from your Dockerfile, or with buildpacks) and stores it | Yes |
| Auto Test | Runs tests it can detect | Yes |
| Auto Code Quality | Checks code quality | Yes |
| Auto SAST / Secret Detection | Scans code for risky patterns and leaked secrets | Yes |
| Auto Dependency / Container Scanning, DAST | Scans libraries, images, and the running app | **Ultimate** |
| Auto Review Apps and Auto Deploy | Deploys to Kubernetes for review, staging, and production | Needs a cluster and domain |

9. Now turn Auto DevOps off so it does not use your minutes: **Settings > CI/CD > Auto DevOps**, untick **Default to Auto DevOps pipeline**, **Save changes**.

---

## Step 2: Set Up Docker Hub

1. Sign in at https://hub.docker.com.
2. Go to **Repositories > Create repository**. Name: `gitlab-lab-app`. Visibility: **Public**. Click **Create**.
3. Create an access token: click your avatar, then **Account settings > Personal access tokens** (the exact menu may be under **Security**). Click **Generate new token**.
   - Description: `gitlab-lab`
   - Access permissions: **Read & Write**
4. **Copy the token now.** Docker Hub only shows it once.

**You should see:** an empty repository `YOUR_USERNAME/gitlab-lab-app` and a token saved in a safe place.

---

## Step 3: Create the Project and Save the Credentials

### 3.1 Project and files

1. **New project > Create blank project**. Name: `k8s-registry-lab`. **Private**. Tick **Initialize repository with a README**. Click **Create project**.
2. Create a new file `index.html`:

```html
<!DOCTYPE html>
<html>
  <body>
    <h1>Hello from my container</h1>
    <p>Version 1</p>
  </body>
</html>
```

3. Create a new file `Dockerfile`:

```
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
```

4. Create a new file `k8s/deployment.yaml` (typing the `/` in the file name creates the folder):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lab-web
  template:
    metadata:
      labels:
        app: lab-web
    spec:
      containers:
        - name: web
          image: IMAGE_PLACEHOLDER
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: lab-web
spec:
  type: NodePort
  selector:
    app: lab-web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Commit each file with a short message (for example `Add index.html`).

> `IMAGE_PLACEHOLDER` is a word the pipeline will replace with the real image name.

### 3.2 Save credentials as CI/CD variables

1. Go to **Settings > CI/CD** and expand **Variables**. Click **Add variable**.
2. Add `DOCKERHUB_USERNAME` with your Docker Hub username as the value.
3. Add another variable `DOCKERHUB_TOKEN` with the token as the value. Tick **Mask variable** (or choose the **Masked** visibility option).

> If GitLab refuses to mask the token because of its characters, leave it unmasked but tick **Protect variable** instead. The `main` branch is protected by default, so the pipeline in this lab can still use it. Never print the token in a job.

**You should see:** two variables listed under CI/CD variables.

---

## Step 4: Build and Store the Image in a Registry

### 4.1 Write the pipeline

Go to **Build > Pipeline editor** and replace the content with:

```yaml
stages:
  - build
  - deploy

variables:
  IMAGE_NAME: $DOCKERHUB_USERNAME/gitlab-lab-app

build-image:
  stage: build
  image: docker:27
  services:
    - docker:27-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - docker build -t "$IMAGE_NAME:$CI_COMMIT_SHORT_SHA" -t "$IMAGE_NAME:latest" -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" .
    - docker push "$IMAGE_NAME:$CI_COMMIT_SHORT_SHA"
    - docker push "$IMAGE_NAME:latest"
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Click **Validate**, then commit with message `Build and push image` to `main`.

> **What this does:** `docker:27-dind` is a helper service that lets the job run Docker commands. The job logs in to **two** registries, builds the image once, and pushes it to both **Docker Hub** and **GitLab's own Container Registry**. `$CI_REGISTRY_USER`, `$CI_REGISTRY_PASSWORD`, `$CI_REGISTRY`, and `$CI_REGISTRY_IMAGE` are built-in variables that GitLab fills in for you.

### 4.2 Check both registries

1. Open **Build > Pipelines** and wait for the pipeline to pass.
2. In GitLab go to **Deploy > Container Registry**. Open the image and look at the tags.
3. On Docker Hub open your `gitlab-lab-app` repository and click **Tags**.

**You should see:** a tag with the short commit ID in **both** places, and a `latest` tag on Docker Hub.

---

## Step 5: Run Your Image on a Kubernetes Cluster (Killercoda)

1. Go to https://killercoda.com/playgrounds and open the **Kubernetes** playground. Wait for the terminal to start.
2. Check the cluster:

```
kubectl get nodes
```

**You should see:** one or two nodes with status `Ready`.

3. Run your image (replace `YOUR_DOCKERHUB_USERNAME`):

```
kubectl create deployment manual-web --image=YOUR_DOCKERHUB_USERNAME/gitlab-lab-app:latest
kubectl rollout status deployment/manual-web
kubectl expose deployment manual-web --type=NodePort --port=80
kubectl get svc manual-web
```

4. In the `PORT(S)` column you will see something like `80:31234/TCP`. The number after the colon is the NodePort. Test it:

```
curl localhost:31234
```

**You should see:** the HTML of your page, including `Hello from my container`. The cluster pulled your image from Docker Hub.

5. Clean up the manual test:

```
kubectl delete deployment manual-web
kubectl delete service manual-web
```

---

## Step 6: Connect GitLab to the Cluster

### 6.1 Create the agent configuration file

1. In `k8s-registry-lab` click **+ > New file**. File name: `.gitlab/agents/lab-agent/config.yaml`
2. Paste (replace with your own username or group name):

```yaml
ci_access:
  projects:
    - id: YOUR_NAMESPACE/k8s-registry-lab
```

3. Commit with message `Add agent config`.

> **What this does:** the folder name `lab-agent` is the agent's name. `ci_access` says which projects' pipelines may use the cluster through this agent.

### 6.2 Register the agent

1. Go to **Operate > Kubernetes clusters** and click **Connect a cluster**.
2. Choose `lab-agent` from the list and click **Register**.
3. A box shows an **agent access token** and an install command that starts with `helm repo add gitlab ...`. **Copy the whole command now.** The token is only shown once. Do not share it.

### 6.3 Install the agent in the cluster

Go back to the Killercoda terminal.

1. Check if Helm is installed:

```
helm version
```

If you see "command not found", install it:

```
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

2. Paste the command you copied from GitLab and press Enter. It looks like this (yours has a real token):

```
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install lab-agent gitlab/gitlab-agent \
    --namespace gitlab-agent-lab-agent \
    --create-namespace \
    --set config.token=YOUR_TOKEN \
    --set config.kasAddress=wss://kas.gitlab.com
```

3. Check the agent is running:

```
kubectl get pods -n gitlab-agent-lab-agent
```

**You should see:** a pod with status `Running`.

4. Back in GitLab, refresh **Operate > Kubernetes clusters**.

**You should see:** `lab-agent` with connection status **Connected**.

---

## Step 7: Deploy to the Cluster From a Pipeline

### 7.1 Add the deploy job

In the **Pipeline editor**, add this job at the bottom of the file:

```yaml
deploy-to-k8s:
  stage: deploy
  image: alpine:3.20
  environment:
    name: killercoda
  before_script:
    - apk add --no-cache curl
    - curl -fsSLo /usr/local/bin/kubectl "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    - chmod +x /usr/local/bin/kubectl
  script:
    - kubectl config get-contexts
    - kubectl config use-context "$CI_PROJECT_PATH:lab-agent"
    - sed -i "s|IMAGE_PLACEHOLDER|$IMAGE_NAME:$CI_COMMIT_SHORT_SHA|" k8s/deployment.yaml
    - kubectl apply -f k8s/deployment.yaml
    - kubectl rollout status deployment/lab-web --timeout=120s
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```

Commit with message `Add deploy job`.

> **How it works:** GitLab gives the job a `KUBECONFIG` that contains the agent connection. `kubectl config use-context` picks it. The `sed` line swaps `IMAGE_PLACEHOLDER` for the image you just pushed. The job is **manual** so you decide when to deploy (and so it does not fail when no cluster is running).

### 7.2 Run it

1. Open the new pipeline. After `build-image` passes, click the **play** button on `deploy-to-k8s`.
2. Open the job log.

**You should see:** `deployment "lab-web" successfully rolled out`.

### 7.3 Check in the cluster

In the Killercoda terminal:

```
kubectl get pods
kubectl get svc lab-web
curl localhost:30080
```

**You should see:** two `lab-web` pods `Running`, and the page text `Hello from my container` / `Version 1`.

### 7.4 Deploy a new version

1. Edit `index.html` in GitLab, change `Version 1` to `Version 2`, and commit to `main`.
2. When the pipeline's `build-image` passes, click **play** on `deploy-to-k8s`.
3. In the Killercoda terminal run `curl localhost:30080` again.

**You should see:** `Version 2`. Also open **Operate > Environments**: the `killercoda` environment lists both deployments.

---

## Step 8: Clean Up

- In Killercoda you can just close the session. The cluster is deleted automatically.
- In GitLab, go to **Operate > Kubernetes clusters**, open `lab-agent`, and delete the agent (or revoke its token under its **Access tokens**).
- On Docker Hub you can delete the access token (**Personal access tokens**) if you do not need it any more.

---

## Checklist

- [ ] Saw Auto DevOps create a pipeline and an image automatically
- [ ] Created a Docker Hub repository and access token
- [ ] Added Docker Hub credentials as CI/CD variables
- [ ] Built an image in a pipeline and pushed it to Docker Hub and the GitLab Container Registry
- [ ] Ran the image on a Killercoda cluster by hand
- [ ] Registered the GitLab agent and saw status **Connected**
- [ ] Deployed from a pipeline with `kubectl` and checked the page
- [ ] Deployed a second version and saw the environment history
- [ ] Cleaned up

## Quick Questions

1. What does Auto DevOps do when a project has no `.gitlab-ci.yml`?
2. Why does the GitLab agent connect **out** from the cluster instead of GitLab connecting in?
3. What is the difference between an image **tag** and the image **name**?
4. Why is the deploy job set to `when: manual`?
5. Which Auto DevOps stages need Ultimate?

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| Pipeline stays "pending" | New GitLab.com accounts may need verification before shared runners work; check the banner at the top of GitLab |
| `docker login` fails with unauthorized | Check `DOCKERHUB_USERNAME` is your username (not email) and the token has Read & Write access |
| Cannot mask the token variable | Leave it unmasked and use **Protect variable** instead (see Step 3.2) |
| `Cannot connect to the Docker daemon` | The job needs the `services: docker:27-dind` block and `DOCKER_TLS_CERTDIR`; check indentation |
| Docker Hub "toomanyrequests" error | Docker Hub limits downloads; wait a little and retry the pipeline |
| Agent shows **Not connected** | Check `kubectl get pods -n gitlab-agent-lab-agent`; the token may be wrong. Create a new token under the agent's **Access tokens** and run the `helm` command again |
| Deploy job: context not found | The agent name and `ci_access` project path must match exactly; the project path is case sensitive |
| Deploy job: unauthorized / forbidden | The `config.yaml` is missing, in the wrong folder, or `id:` is wrong |
| Deploy job fails after a break | Your Killercoda session probably expired. Start a new session and repeat Step 6.3 with a new agent token |
| Pods stuck in `ImagePullBackOff` | The image name or tag is wrong, or the Docker Hub repository is private. Keep it **Public** for this lab |
| `kubectl` version warning | A small version difference is fine for this lab; warnings can be ignored |
| `curl localhost:30080` fails | Wait for pods to be `Running`; if still failing use the node IP: `curl http://$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}'):30080` |
| Menu names look different | GitLab renames menus now and then. Search for similar wording (Kubernetes clusters may be under **Operate** or **Infrastructure**) |
