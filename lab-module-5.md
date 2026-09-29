# Lab 5 : CI/CD Fundamentals

**Tools:** GitLab.com Free account, Git for Windows, PowerShell
**Time:** ~60 minutes
**Goal:** Understand what a pipeline is, write a simple `.gitlab-ci.yml`, and watch it run.

---

## What You Will Learn

- What CI/CD means and how GitLab's pipeline model works
- The parts of `.gitlab-ci.yml`: **stages**, **jobs**, and **scripts**
- What can **trigger** a pipeline to run
- How to read the **pipeline visualisation** (the pipeline graph)

## Words to Know

| Word | Simple meaning |
|---|---|
| CI (Continuous Integration) | Automatically testing/building your code every time it changes |
| CD (Continuous Delivery/Deployment) | Automatically getting tested code ready to release, or releasing it |
| Pipeline | One full run of your automated steps, triggered by an event (like a push) |
| Stage | A phase of the pipeline, for example `test` then `build`. Stages run **one after another** |
| Job | A single task inside a stage, for example "run tests". Jobs in the **same stage** run **at the same time** |
| Script | The actual commands a job runs |
| Runner | The machine (or container) that actually executes a job. GitLab.com gives you free **shared runners** |
| Artifact | A file saved from a job so later jobs (or you) can use it |
| `.gitlab-ci.yml` | The file in your repo that describes your whole pipeline |

> **Free tier note:** GitLab.com Free gives every account a monthly quota of shared-runner **compute minutes**. This lab uses very little. You do not need your own runner for this lab — that is Module 6.

---

## Step 0: Check Your Setup

Open **PowerShell**:

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

If Git asks for a password, use a **personal access token** (GitLab: **avatar > Preferences > Access tokens**, scopes `read_repository` and `write_repository`).

---

## Step 1: Create a Project

1. **New project > Create blank project**.
2. Name: `cicd-lab`. **Private**. Tick **Initialize repository with a README**.
3. Click **Create project**.

Clone it:

```powershell
cd $HOME
git clone PASTE_THE_HTTPS_ADDRESS_HERE
cd cicd-lab
```

---

## Step 2: Your First Pipeline (One Job)

Create the pipeline file:

```powershell
@'
say-hello:
  script:
    - echo "Hello from GitLab CI/CD"
'@ | Set-Content -Path .gitlab-ci.yml -Encoding ascii
```

Push it:

```powershell
git add .gitlab-ci.yml
git commit -m "Add first pipeline"
git push origin main
```

Go to **Build > Pipelines** in GitLab.

**You should see:** a new pipeline, status changing from **pending** to **running** to **passed**. Click into it, then click the **say-hello** job to see the log with `Hello from GitLab CI/CD`.

> **Why did this run automatically?** A push to the repository is a pipeline **trigger**. GitLab saw the new `.gitlab-ci.yml` and started a pipeline.

---

## Step 3: Add Stages and More Jobs

Replace the file with:

```powershell
@'
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building the application..."

unit-test-job:
  stage: test
  script:
    - echo "Running unit tests..."

lint-job:
  stage: test
  script:
    - echo "Running lint checks..."

deploy-job:
  stage: deploy
  script:
    - echo "Deploying the application..."
'@ | Set-Content -Path .gitlab-ci.yml -Encoding ascii

git add .gitlab-ci.yml
git commit -m "Add stages and multiple jobs"
git push
```

Open the new pipeline in **Build > Pipelines**.

**You should see:** three columns — `build`, `test`, `deploy`. The `test` stage has **two boxes side by side** (`unit-test-job` and `lint-job`).

### This is the pipeline visualisation

- Stages run **left to right**, one after another.
- Jobs **inside the same stage** run **at the same time** (in parallel), each on its own runner.
- If a job in `build` fails, the `test` stage will not start.

Click **unit-test-job** to see its own separate log.

---

## Step 4: Make a Job Fail (and See What Happens)

Edit the file so `lint-job` fails on purpose:

```powershell
@'
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building the application..."

unit-test-job:
  stage: test
  script:
    - echo "Running unit tests..."

lint-job:
  stage: test
  script:
    - echo "Running lint checks..."
    - exit 1

deploy-job:
  stage: deploy
  script:
    - echo "Deploying the application..."
'@ | Set-Content -Path .gitlab-ci.yml -Encoding ascii

git add .gitlab-ci.yml
git commit -m "Break lint-job on purpose"
git push
```

**You should see:** `lint-job` fails (red). `unit-test-job` still passes (they ran independently). `deploy-job` **never starts**, because the `test` stage did not fully succeed.

### Fix it and re-run

```powershell
(Get-Content .gitlab-ci.yml) -replace '    - exit 1', '' | Set-Content .gitlab-ci.yml -Encoding ascii
git commit -am "Fix lint-job"
git push
```

**You should see:** a new pipeline where all stages pass, including `deploy-job`.

> You can also click **Retry** on a failed job instead of pushing a fix, if you just want to try again.

---

## Step 5: Scripts Can Do Real Work

Let's make a job actually build something. Add a Python file:

```powershell
"print('Hello, CI/CD')" | Set-Content -Path app.py -Encoding ascii
```

Update the `build-job` to actually run it:

```powershell
(Get-Content .gitlab-ci.yml) -replace '    - echo "Building the application\.\.\."', "    - python --version`n    - python app.py" | Set-Content .gitlab-ci.yml -Encoding ascii

git add app.py .gitlab-ci.yml
git commit -m "Run app.py in build-job"
git push
```

**You should see:** `build-job`'s log shows a Python version and `Hello, CI/CD`.

> This works because GitLab's shared runners on Linux normally have Python pre-installed. In Module 6 you will learn how the **executor** and **image** decide what tools are available.

---

## Step 6: Pipeline Triggers

A pipeline can start in more than one way. Let's try a few.

### 6.1 Push trigger (already used)

Every `git push` you did above triggered a pipeline. This is the most common trigger.

### 6.2 Manual trigger from the UI

1. Go to **Build > Pipelines**.
2. Click **Run pipeline**.
3. Choose branch `main` and click **Run pipeline** again.

**You should see:** a new pipeline start with no code change at all.

### 6.3 Manual jobs (a job that waits for a click)

Add a manual deploy step:

```powershell
@'
stages:
  - setup
  - build
  - test
  - deploy

image: python:3.12

install-python:
  stage: setup
  script:
    - python --version
    - pip --version

build-job:
  stage: build
  script:
    - python --version
    - python app.py

unit-test-job:
  stage: test
  script:
    - echo "Running unit tests..."

lint-job:
  stage: test
  script:
    - echo "Running lint checks..."

deploy-job:
  stage: deploy
  script:
    - echo "Deploying the application..."
  when: manual
'@ | Set-Content -Path .gitlab-ci.yml -Encoding ascii

git add .gitlab-ci.yml
git commit -m "Make deploy-job manual"
git push
```

**You should see:** the pipeline stops at `deploy-job` with a **play button** instead of running automatically. Click it to run that job whenever you are ready. This is common for real deployments, so nobody deploys by accident.

### 6.4 Scheduled trigger (overview)

Go to **Build > Pipeline schedules > New schedule**. You could set a pipeline to run every night, for example. Create one to see the form, then delete it (click the trash icon) so it does not keep running.

---

## Step 7: Read the Full Pipeline Visualisation

Open any pipeline and look at:

- **Pipeline graph** (default view): stages as columns, jobs as boxes, arrows showing order.
- **Job list**: a simple table view. Switch using the tabs at the top.
- Click a job's three-dot menu to see **Retry**, **Download artifacts** (none in this lab, but you will use this later), and **View log**.

**Icons to know:**

| Icon/colour | Meaning |
|---|---|
| Grey clock | Pending, waiting for a runner |
| Blue spinner | Running |
| Green check | Passed |
| Red cross | Failed |
| Play button | Manual job, waiting for a click |

---

## Checklist

- [ ] Created a project and pushed a one-job pipeline
- [ ] Added stages with jobs running in sequence and in parallel
- [ ] Watched a job fail and a later stage get skipped
- [ ] Fixed the failure and saw the pipeline pass
- [ ] Ran a job that actually executes a script (`python app.py`)
- [ ] Triggered a pipeline manually from the UI
- [ ] Used `when: manual` on a job
- [ ] Looked at a pipeline schedule (and deleted it)

## Quick Questions

1. What is the difference between a stage and a job?
2. Why did `deploy-job` not run when `lint-job` failed?
3. Name two ways a pipeline can start, besides a `git push`.
4. What does `when: manual` do?

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| No pipeline appears after push | Make sure the file is named exactly `.gitlab-ci.yml` at the **root** of the repo, not inside a folder |
| `yaml invalid` error shown on the pipeline page | Check indentation; YAML uses spaces, not tabs, and is sensitive to alignment |
| Job stuck on "pending" for a long time | GitLab.com shared runners can be briefly busy; wait a minute, or check **Build > Pipelines** for a warning about compute minutes |
| Here-string error in PowerShell | The closing `'@` must be alone at the start of its own line |
| `python: command not found` | The default shared runner image may differ; try `- echo "no python here"` instead, or continue to Module 6 to learn about images/executors |
