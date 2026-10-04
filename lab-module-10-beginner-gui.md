# Lab 10 (Beginner, GUI-only): Deployment & Environments

**Tools:** A web browser and a GitLab.com Free account. **No PowerShell, no Git commands, no downloads.**
**Time:** ~75 minutes
**Goal:** Set up a deploy stage with environments, a review app for each merge request, and publish a static website with GitLab Pages.

---

## What You Will Learn

- What an **environment** is and how GitLab tracks **deployments**
- How **review apps** (dynamic environments) work for merge requests
- The difference between **manual**, **continuous**, and **rolling** deployments
- How to publish a static site with **GitLab Pages**

## Words to Know

| Word | Simple meaning |
|---|---|
| Environment | A named place where your app runs, such as `staging` or `production` |
| Deployment | One time your code was put into an environment |
| Static environment | A fixed one that always exists (`staging`, `production`) |
| Dynamic environment | One created on demand, with a name that changes (for example one per merge request) |
| Review app | A temporary copy of your app for one merge request, so reviewers can try the change |
| GitLab Pages | A free way to host a static website (HTML, CSS, JavaScript) from your repo |
| Rollback | Re-deploying an earlier, known-good version |

---

## Free Tier: What Works and What Does Not

| Feature | On GitLab Free? | In this lab |
|---|---|---|
| Environments and deployment history, rollback | Yes | Hands-on |
| Review apps / dynamic environments | Yes | Hands-on (simulated, see note below) |
| Manual jobs (`when: manual`) | Yes | Hands-on |
| GitLab Pages | Yes | Hands-on |
| Protected environments, deployment approvals, deploy boards | **No (Premium+)** | Mentioned only |
| Kubernetes-based rolling and canary rollouts | Needs a Kubernetes cluster | Concept only |

> **A note about "simulated" deployments:** a real deployment copies your app to a server. Students on Free usually do not have a server, so in this lab the deploy jobs only **pretend** to deploy (they print messages and keep the site as a downloadable file). GitLab still tracks the environments, deployments, and URLs for real, which is what this module teaches. The production deployment uses **real GitLab Pages**.

---

## Step 0: Two Tools You Will Use All Lab

1. **Pipeline Editor** (**Build > Pipeline editor**): edit `.gitlab-ci.yml` with a **Validate** button.
2. **New file / Edit** on the repository page (**Code > Repository**): create or change other files. After each edit, add a **commit message** and click **Commit changes**.

---

## Step 1: Create the Project and a Tiny Website

1. **New project > Create blank project**. Name: `deploy-lab`. **Private**. Tick **Initialize repository with a README**. Click **Create project**.
2. On the project page click **+ > New file**. File name: `index.html`. Paste:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>My Deploy Lab Site</title>
  </head>
  <body>
    <h1>Hello from the Deploy Lab</h1>
    <p>Commit: __COMMIT__</p>
    <p>Branch: __BRANCH__</p>
  </body>
</html>
```

3. Commit with message `Add index.html`.

**You should see:** `index.html` in your repository. The words `__COMMIT__` and `__BRANCH__` are placeholders that the pipeline will fill in.

---

## Step 2: Build the Site and Deploy to Staging

### 2.1 Write the pipeline

Go to **Build > Pipeline editor** and replace the content with:

```yaml
stages:
  - build
  - deploy

build-site:
  stage: build
  image: alpine:latest
  script:
    - mkdir -p public
    - cp index.html public/index.html
    - sed -i "s/__COMMIT__/$CI_COMMIT_SHORT_SHA/" public/index.html
    - sed -i "s/__BRANCH__/$CI_COMMIT_REF_SLUG/" public/index.html
  artifacts:
    paths:
      - public
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-staging:
  stage: deploy
  script:
    - echo "Deploying commit $CI_COMMIT_SHORT_SHA to staging"
  artifacts:
    paths:
      - public
  environment:
    name: staging
    url: https://gitlab.com/$CI_PROJECT_PATH/-/jobs/$CI_JOB_ID/artifacts/browse/public/
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Click **Validate**, then commit with message `Add build and staging deploy` to `main`.

> **What `environment:` does:** it tells GitLab "this job is a deployment to `staging`". GitLab then records every run as a deployment and shows it on the Environments page. The `url:` adds an **Open** button.
>
> **Why `deploy-staging` lists `artifacts`:** it re-saves the built site so there is a page to browse. A real deployment would copy the files to a server instead.

### 2.2 Look at the environment

1. Open **Build > Pipelines** and wait for the pipeline to pass.
2. Go to **Operate > Environments**.

**You should see:** an environment called **staging**, marked **Available**, with the commit that was deployed, who deployed it, and an **Open** button.

3. Click **Open**. It shows the file browser for the built site. Click `index.html`.

**You should see:** the page with your commit ID filled in where `__COMMIT__` was.

### 2.3 Deploy twice and use rollback

1. Edit `index.html`, change the heading to `Hello again from the Deploy Lab`, and commit to `main`.
2. Wait for the pipeline, then go back to **Operate > Environments** and click on **staging**.

**You should see:** a **deployment history** with two entries, each linked to a commit.

3. On the **older** deployment click **Rollback** (or **Re-deploy to environment**), and confirm.

**You should see:** a new deployment appears running the **old** commit again. This is how teams quickly return to a known-good version.

---

## Step 3: Review Apps (Dynamic Environments)

A review app is a temporary environment for **one merge request**. Its name changes per branch, which is what makes it **dynamic**.

### 3.1 Add the review jobs

In the **Pipeline editor**, add these two jobs at the bottom of the file:

```yaml
deploy-review:
  stage: deploy
  script:
    - echo "Deploying review app for branch $CI_COMMIT_REF_NAME"
  artifacts:
    paths:
      - public
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://gitlab.com/$CI_PROJECT_PATH/-/jobs/$CI_JOB_ID/artifacts/browse/public/
    on_stop: stop-review
    auto_stop_in: 1 day
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop-review:
  stage: deploy
  script:
    - echo "Stopping review app for $CI_COMMIT_REF_NAME"
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
  allow_failure: true
```

Commit with message `Add review app jobs` to `main`.

> **How it fits together:** the environment name `review/$CI_COMMIT_REF_SLUG` becomes `review/my-branch` for a branch called `my-branch`, so each branch gets its own environment. `on_stop` points to the job that cleans it up. `auto_stop_in` removes it automatically after a day.

### 3.2 Create a merge request

1. Go to **Code > Branches > New branch**. Name: `change-heading`, based on `main`.
2. Open `index.html` on that branch (use the branch selector at the top left), click **Edit**, change the heading to `Hello from my review app`, and commit with message `Change heading`.
3. Click **Create merge request** and then **Create merge request** again.

### 3.3 Find the review app

1. In the merge request, wait for the pipeline to finish.
2. Look for the **deployment** line in the merge request page, with a **View app** button.
3. Also check **Operate > Environments**.

**You should see:** a new environment named `review/change-heading` next to `staging`. Click **View app** (or **Open**), then `index.html`.

**You should see:** your changed heading and the branch name `change-heading` filled in. A reviewer can check the change **without** pulling the code. That is the point of a review app.

### 3.4 Stop the review app

1. On the merge request pipeline, click the **play** button next to `stop-review`, **or**
2. In **Operate > Environments**, click the **Stop** button for the review environment.

**You should see:** the environment moves to the **Stopped** tab.

> If you merge the merge request and delete the branch, GitLab also runs the stop job for you. Review apps clean up after themselves.

---

## Step 4: Production With GitLab Pages (Manual Deployment)

### 4.1 Add a production job

Merge your merge request first (click **Merge** in the MR), then go back to the **Pipeline editor** on `main`. Change the `stages:` list at the top to:

```yaml
stages:
  - build
  - deploy
  - production
```

Then add this job at the bottom:

```yaml
pages:
  stage: production
  script:
    - echo "Publishing the site to GitLab Pages"
  artifacts:
    paths:
      - public
  environment:
    name: production
    url: $CI_PAGES_URL
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```

Commit with message `Add production deploy with GitLab Pages`.

> **Why the job is named `pages`:** GitLab Pages looks for a job called `pages` that saves a folder called `public` as an artifact. Anything in that folder becomes your website.
>
> **Why `when: manual`:** the job waits until a person clicks **play**. That is a **manual deployment**.

### 4.2 Run it

1. Open the new pipeline on `main`. After `build-site` and `deploy-staging` finish, the pipeline pauses at `pages` with a **play** button (a "blocked" or "manual" status).
2. Click the **play** button on `pages`.
3. Wait for it to finish (a `pages:deploy` job may also appear; let it finish).

### 4.3 Open your live site

1. Go to **Deploy > Pages** and copy the site address (or use **Operate > Environments > production > Open**).
2. Open it in a new browser tab.

**You should see:** your web page, now on a real GitLab Pages address, showing the commit ID and branch `main`.

> A private project's Pages site is normally visible only to project members, so you may need to be signed in to GitLab. You can change this in **Settings > General > Visibility, project features, permissions > Pages**.

**You should see** in **Operate > Environments** three kinds of entries: `staging`, `production`, and (if not stopped or cleaned up) review environments.

---

## Step 5: Deployment Strategies (Read and Match)

| Strategy | How it works | In this lab |
|---|---|---|
| **Manual deployment** | A person decides when to deploy and clicks a button | The `pages` job (`when: manual`) |
| **Continuous delivery** | Every change that passes tests is **ready** to deploy, and a person approves the last step | Staging deploys automatically, production waits for a click |
| **Continuous deployment** | Every change that passes tests is deployed to production **automatically** | Remove `when: manual` from the `pages` job to get this |
| **Rolling deployment** | Replace the old version **a few servers at a time**, so the app never goes fully down | Concept only (needs several servers, often Kubernetes) |

Two related ideas you will meet in real teams:

- **Canary release:** send a small share of users to the new version first.
- **Blue-green:** run old and new versions side by side, then switch all traffic at once.

> **On GitLab:** manual and continuous deployments work on Free. Deployment approvals and **protected environments** (limit who may deploy to `production`) are **Premium**. Rolling and canary rollouts usually need Kubernetes, which is outside this lab.

---

## Step 6: Hands-on Review (Whole Flow in One File)

Your final `.gitlab-ci.yml` on `main` should look like this. Compare it with yours:

```yaml
stages:
  - build
  - deploy
  - production

build-site:
  stage: build
  image: alpine:latest
  script:
    - mkdir -p public
    - cp index.html public/index.html
    - sed -i "s/__COMMIT__/$CI_COMMIT_SHORT_SHA/" public/index.html
    - sed -i "s/__BRANCH__/$CI_COMMIT_REF_SLUG/" public/index.html
  artifacts:
    paths:
      - public
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-staging:
  stage: deploy
  script:
    - echo "Deploying commit $CI_COMMIT_SHORT_SHA to staging"
  artifacts:
    paths:
      - public
  environment:
    name: staging
    url: https://gitlab.com/$CI_PROJECT_PATH/-/jobs/$CI_JOB_ID/artifacts/browse/public/
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-review:
  stage: deploy
  script:
    - echo "Deploying review app for branch $CI_COMMIT_REF_NAME"
  artifacts:
    paths:
      - public
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://gitlab.com/$CI_PROJECT_PATH/-/jobs/$CI_JOB_ID/artifacts/browse/public/
    on_stop: stop-review
    auto_stop_in: 1 day
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop-review:
  stage: deploy
  script:
    - echo "Stopping review app for $CI_COMMIT_REF_NAME"
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
  allow_failure: true

pages:
  stage: production
  script:
    - echo "Publishing the site to GitLab Pages"
  artifacts:
    paths:
      - public
  environment:
    name: production
    url: $CI_PAGES_URL
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```

Run one last test: make a second branch and merge request, check its review app, then merge it, and confirm the production site updates after you click **play** on `pages`.

---

## Checklist

- [ ] Built a small site in a pipeline and saved it as an artifact
- [ ] Deployed to a `staging` environment and saw it on the Environments page
- [ ] Used **Rollback** to re-deploy an older version
- [ ] Created a merge request and found its **review app**
- [ ] Stopped the review app
- [ ] Published the site with **GitLab Pages** through a manual job
- [ ] Opened the live Pages address
- [ ] Can explain manual, continuous, and rolling deployments

## Quick Questions

1. What does the `environment:` keyword tell GitLab?
2. Why is a review app called a "dynamic" environment?
3. What does `when: manual` do, and when would a team want it?
4. What is the difference between continuous delivery and continuous deployment?
5. What must a job be called, and what must it save, to publish with GitLab Pages?
6. Name two deployment features that need Premium.

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| No **Operate > Environments** menu item | Environments appear after the first job with `environment:` runs; also check the menu name (it may be under **Deploy** in some versions) |
| Environment URL opens an error page | The link goes to the job's artifacts, so the job must have finished and its artifacts must not have expired. Re-run the pipeline |
| No review app appears in the merge request | Pipelines for merge requests need the `rules:` lines shown; check the MR pipeline (not only the branch pipeline) ran, and look in **Operate > Environments** |
| Review app does not stop on merge | Check `on_stop` and the `stop-review` job use the **same** environment `name` |
| `pages` job never shows a play button | Check `when: manual` is under a `rules:` item and that the pipeline is on `main` |
| Pages site shows 404 | Wait a minute after the job finishes; confirm the job saved a `public` folder with `index.html` inside |
| Cannot open the Pages site | For private projects you must be signed in as a project member, or change Pages visibility in the project settings |
| `sed` error in `build-site` | The branch placeholder must match exactly (`__BRANCH__`, `__COMMIT__`) with two underscores each side |
| Environment `url:` not accepted | Check the line is indented under `environment:` and starts with `https://` |
