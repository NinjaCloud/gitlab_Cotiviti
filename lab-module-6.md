# Lab 6 : GitLab Runners

**Tools:** GitLab.com Free account, Git for Windows, PowerShell (as **Administrator** for parts of this lab)
**Time:** ~75 minutes
**Goal:** Understand what a runner is, install one on your own Windows computer, register it, and use it to run a pipeline.

---

## What You Will Learn

- The difference between **shared**, **group**, and **project** runners
- How to install and register a GitLab Runner
- The difference between the **Shell**, **Docker**, and **Kubernetes** executors
- How to write a basic `.gitlab-ci.yml` and run your first pipeline on your own runner

## Words to Know

| Word | Simple meaning |
|---|---|
| Runner | A program that picks up pipeline jobs and executes them |
| Executor | *How* the runner runs a job: directly on the machine (Shell), inside a container (Docker), or on a cluster (Kubernetes) |
| Shared runner | Provided by GitLab.com, available to every project, with a monthly free-minutes quota |
| Group runner | Registered by an admin for one **group** and all its projects |
| Project runner | Registered for **one specific project** — the type we will create today |
| Registration / authentication token | A secret code that connects your runner to your GitLab project |
| Tag | A label on a runner (and on a job) so GitLab knows which jobs go to which runner |

> **Free tier note:** Registering your **own project runner** works fully on GitLab Free — this is not a Premium feature. What is limited on Free is the **shared runner minutes** GitLab provides you each month, and some admin-level runner management for whole instances.

---

## Step 0: Check Your Setup

```powershell
git --version
```

If missing:

```powershell
winget install --id Git.Git -e
```

Reopen PowerShell, then (one time only):

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

## Step 1: Create a Project

1. **New project > Create blank project**.
2. Name: `runner-lab`. **Private**. Tick **Initialize repository with a README**.
3. Click **Create project**.

Clone it:

```powershell
cd $HOME
git clone PASTE_THE_HTTPS_ADDRESS_HERE
cd runner-lab
```

---

## Step 2: Runner Architecture — Shared, Group, and Project

Before installing anything, look at what already exists.

1. Go to **Settings > CI/CD** and expand **Runners**.
2. You should see a section listing **shared runners** — these belong to GitLab.com, available to your project automatically, with a monthly free-minutes quota.

| Type | Who manages it | Who can use it |
|---|---|---|
| **Shared** | GitLab.com | Every project on the instance (with a Free-tier minutes quota) |
| **Group** | A group Owner | Every project inside that group |
| **Project** | You, the project Maintainer/Owner | Only that one project (unless you allow more later) |

Today you will create a **project runner** — the type an individual student or a small project team is most likely to set up.

---

## Step 3: Install GitLab Runner on Windows

1. Create a folder for the runner:

```powershell
New-Item -ItemType Directory -Path "C:\GitLab-Runner" -Force
cd "C:\GitLab-Runner"
```

2. Download the runner:

```powershell
Invoke-WebRequest -Uri "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-windows-amd64.exe" -OutFile "gitlab-runner.exe"
```

3. Check it downloaded correctly:

```powershell
.\gitlab-runner.exe --version
```

**You should see:** version information printed, with no error.

> If `Invoke-WebRequest` is blocked by your network, download the file manually from **https://docs.gitlab.com/runner/install/windows.html** and save it as `gitlab-runner.exe` in `C:\GitLab-Runner`.

---

## Step 4: Register the Runner as a Project Runner

1. In GitLab, go to **Settings > CI/CD > Runners** and click **New project runner**.
2. Fill in the form:
   - **Tags:** `windows-shell` (this lets you target this runner from a job)
   - **Description:** `My laptop runner`
   - Leave **Run untagged jobs** ticked, so it can pick up simple jobs too.
3. Click **Create runner**.
4. GitLab shows you a **registration command** with a token. It looks like this (yours will have a real token):

```
gitlab-runner register --url https://gitlab.com --token glrt-XXXXXXXXXXXXXXXX
```

5. Copy your real command, then run it in PowerShell (you can paste the whole command, or answer the prompts one at a time):

```powershell
.\gitlab-runner.exe register --url "https://gitlab.com" --token "PASTE_YOUR_TOKEN_HERE"
```

6. When asked for an **executor**, type:

```
shell
```

**You should see:** `Runner registered successfully.` in PowerShell, and on the GitLab **Runners** page your runner listed with a green dot (online).

> Keep that token private — do not commit it to a file or share it in chat.

---

## Step 5: Install the Runner as a Windows Service

This makes the runner keep working in the background, even after you close PowerShell.

Open PowerShell **as Administrator** (right-click PowerShell, **Run as administrator**):

```powershell
cd "C:\GitLab-Runner"
.\gitlab-runner.exe install
.\gitlab-runner.exe start
```

Check it is running:

```powershell
.\gitlab-runner.exe status
```

**You should see:** a message saying the service is running.

---

## Step 6: Executors — Shell, Docker, and Kubernetes

You just registered a runner using the **shell** executor. Here is how all three compare:

| Executor | How it runs jobs | Good for | Needs |
|---|---|---|---|
| **Shell** | Runs the script directly on the runner's own machine, using whatever is already installed | Learning, simple scripts, small teams | Nothing extra — used in this lab |
| **Docker** | Runs each job inside a fresh **container**, built from an `image:` you choose in `.gitlab-ci.yml` | Consistent, clean environments; different jobs can use different languages/tools | Docker installed on the runner's machine |
| **Kubernetes** | Runs each job as a **pod** on a Kubernetes cluster | Large teams, heavy or many parallel jobs, cloud-scale CI | A Kubernetes cluster — out of scope for this lab |

> **In this lab we use Shell**, because it needs no extra software. If you have **Docker Desktop** installed, try the optional Docker section at the end. Kubernetes is explained here conceptually only — setting up a cluster is beyond a beginner lab.

---

## Step 7: Hands-on Lab — Write a Basic `.gitlab-ci.yml` and Run Your First Pipeline

### 7.1 Write the pipeline file

```powershell
cd $HOME\runner-lab

@'
stages:
  - build
  - test

build-job:
  stage: build
  tags:
    - windows-shell
  script:
    - echo "Building on my own runner..."
    - echo "Computer name:" $env:COMPUTERNAME

test-job:
  stage: test
  tags:
    - windows-shell
  script:
    - echo "Running a simple test..."
    - if (1 -eq 1) { echo "Test passed" } else { throw "Test failed" }
'@ | Set-Content -Path .gitlab-ci.yml -Encoding ascii
```

> The `tags:` line tells GitLab: *"only run this job on a runner tagged `windows-shell`"* — that is your laptop.

### 7.2 Push it

```powershell
git add .gitlab-ci.yml
git commit -m "Add pipeline for my own runner"
git push origin main
```

### 7.3 Watch it run

Go to **Build > Pipelines** in GitLab, open the pipeline, and click into `build-job`.

**You should see:** the log shows your **own computer's name**, proving the job ran on your machine, not on a GitLab.com shared runner. On the **Runners** page, your runner's **"Jobs"** count goes up by one each time.

---

## Step 8: Compare With the Shared Runner

Remove the `tags:` lines so the job can run on any runner:

```powershell
@'
stages:
  - build
  - test

build-job:
  stage: build
  script:
    - echo "Building on any available runner..."

test-job:
  stage: test
  script:
    - echo "Running a simple test..."
'@ | Set-Content -Path .gitlab-ci.yml -Encoding ascii

git add .gitlab-ci.yml
git commit -m "Remove tags, allow any runner"
git push
```

**You should see:** this time the job may run on a GitLab.com **shared runner** instead of your laptop (check the job details — it will show a different runner description, and it will not print your computer name the same way if you kept the old script). This shows how `tags:` controls **which** runner picks up a job.

---

## Step 9 (Optional): Try the Docker Executor

Only do this if you already have **Docker Desktop** installed and running.

1. Register a second runner, this time choosing `docker` as the executor:

```powershell
.\gitlab-runner.exe register --url "https://gitlab.com" --token "PASTE_A_NEW_TOKEN_HERE"
```

When asked for the **default Docker image**, type `alpine:latest`. When asked for the executor, type `docker`.

2. Add a job that uses an `image:`:

```powershell
@'
stages:
  - build

docker-job:
  stage: build
  image: alpine:latest
  tags:
    - windows-shell
  script:
    - echo "Running inside a container"
    - cat /etc/os-release
'@ | Add-Content -Path .gitlab-ci.yml -Encoding ascii
```

3. Push and watch the log show Alpine Linux details — proof the job ran inside a fresh container, not directly on Windows.

---

## Step 10: Clean Up

Stop and remove the runner service so it does not keep running after the lab:

```powershell
cd "C:\GitLab-Runner"
.\gitlab-runner.exe stop
.\gitlab-runner.exe uninstall
```

In GitLab, go to **Settings > CI/CD > Runners**, open your runner, and click **Delete runner**.

---

## Checklist

- [ ] Looked at the shared runners already available to your project
- [ ] Downloaded and checked `gitlab-runner.exe`
- [ ] Created a project runner in GitLab and copied its token
- [ ] Registered the runner with the **shell** executor
- [ ] Installed and started the runner as a Windows service
- [ ] Wrote a `.gitlab-ci.yml` with `tags:` pointing at your runner
- [ ] Ran a pipeline and saw it execute on your own machine
- [ ] Removed the tags and compared with a shared runner
- [ ] (Optional) Tried the Docker executor
- [ ] Cleaned up: stopped the service and deleted the runner

## Quick Questions

1. What is the difference between a shared, group, and project runner?
2. What does the `tags:` field do, on both a runner and a job?
3. Why might a team choose the Docker executor over Shell?
4. Why is Kubernetes usually chosen only by larger teams?

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| `gitlab-runner.exe` not recognized | Make sure you are in `C:\GitLab-Runner` and typing `.\gitlab-runner.exe`, not just `gitlab-runner.exe` |
| `register` command fails with 401/403 | Token was mistyped or already used; create a new runner in GitLab and copy the token again |
| Runner shows offline (grey) in GitLab | Run `.\gitlab-runner.exe status` as Administrator; if stopped, run `.\gitlab-runner.exe start` |
| Job stays "pending" forever | No runner has a matching tag, or your runner is offline; check **Settings > CI/CD > Runners** |
| `install` command needs Administrator | Right-click PowerShell and choose **Run as administrator** |
| Docker job fails, "Cannot connect to the Docker daemon" | Docker Desktop is not running; start it first |
| Forgot your token | You cannot view it again; delete the runner in GitLab and create a new one |
