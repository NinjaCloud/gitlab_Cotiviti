# Lab 7 : Advanced Pipeline Configuration


## What You Will Learn

- How to store **variables and secrets** in GitLab and scope them to environments
- How to pass **artifacts** between jobs, and how **caching** speeds up pipelines
- How **rules** and **workflow** control whether a pipeline or job runs at all
- How **parallel jobs**, **needs**, and **DAG pipelines** make pipelines faster

## Words to Know

| Word | Simple meaning |
|---|---|
| CI/CD variable | A named value (like a setting or password) GitLab stores and gives to your jobs |
| Masked variable | A variable whose value is hidden (shown as `[MASKED]`) in job logs |
| Protected variable | A variable only available on protected branches/tags |
| Environment scope | Limits a variable to a specific environment, e.g. only `production` |
| Artifact | A file a job saves so **later jobs** (or you) can download it |
| Cache | Files kept between pipeline **runs** to save time (e.g. downloaded packages) |
| Rule (`rules:`) | A condition that decides if a **job** runs |
| `workflow: rules` | A condition that decides if the **whole pipeline** runs at all |
| `needs:` | Lets a job start as soon as the specific jobs it needs are done, instead of waiting for the whole previous stage |
| DAG (Directed Acyclic Graph) pipeline | A pipeline where jobs run based on `needs:` relationships instead of strict stage order |

---

## Step 0: Two Tools You Will Use All Lab

1. **Pipeline Editor** — the easiest way to edit `.gitlab-ci.yml`. It has a built-in **Validate** button and shows you the pipeline graph as you type. Find it at **Build > Pipeline editor**.
2. **File editor** — for any other file, open it in **Code > Repository**, click the file, then click **Edit**.

Every time you finish editing, scroll down, write a short **commit message**, and click **Commit changes**. This is the GUI equivalent of `git commit` + `git push` in one click.

---

## Step 1: Create the Project

1. Click **New project > Create blank project**.
2. Name: `advanced-pipelines-lab`. Visibility: **Private**. Tick **Initialize repository with a README**.
3. Click **Create project**.

**You should see:** your new project's overview page.

---

## Step 2: Create a Starting Pipeline

1. Go to **Build > Pipeline editor**.
2. If it says there is no `.gitlab-ci.yml` yet, it will offer to create one — accept, or click **Configure pipeline**.
3. Replace whatever is in the editor with:

```yaml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building the app"

test-job:
  stage: test
  script:
    - echo "Testing the app"

deploy-job:
  stage: deploy
  script:
    - echo "Deploying the app"
```

4. Click **Lint** or **Validate** near the top to check the YAML is valid (should say *"Pipeline syntax is valid"*).
5. Scroll down, add commit message `Add starting pipeline`, and click **Commit changes** (commit directly to `main`).

**You should see:** the editor switches to the **Visualize** tab showing three boxes: `build-job`, `test-job`, `deploy-job`, connected in order. Go to **Build > Pipelines** to see it actually run.

---

## Step 3: Variables and Secrets

### 3.1 Add a plain variable

1. Go to **Settings > CI/CD**.
2. Expand the **Variables** section.
3. Click **Add variable**.
4. Fill in:
   - **Key:** `APP_ENV`
   - **Value:** `staging`
   - Leave **Protect variable** and **Mask variable** unticked for now.
5. Click **Add variable**.

### 3.2 Add a "secret" variable (masked)

1. Click **Add variable** again.
2. Fill in:
   - **Key:** `API_KEY`
   - **Value:** `sk_test_1234567890abcdef`
   - Tick **Mask variable** (this hides the value in job logs — GitLab requires the value to look sufficiently secret-like to allow masking).
3. Click **Add variable**.

> If GitLab refuses to mask a short or simple value, make it longer (at least 8 characters, no spaces) and try again.

### 3.3 Use the variables in your pipeline

Go back to **Build > Pipeline editor** and update `build-job`:

```yaml
build-job:
  stage: build
  script:
    - echo "Environment is $APP_ENV"
    - echo "Using API key $API_KEY"
```

Commit the change. Open the new pipeline, click into `build-job`.

**You should see:** the log prints `Environment is staging`, but the API key line shows `[MASKED]` instead of the real value.

### 3.4 Environment scope

1. Go back to **Settings > CI/CD > Variables**.
2. Click **Add variable** once more:
   - **Key:** `APP_ENV`
   - **Value:** `production`
   - **Environment scope:** click the dropdown and choose (or type) `production` instead of `All (default)`.
3. Click **Add variable**.

**You now have two `APP_ENV` variables** — one for all environments (`staging`) and one that only applies when a job's `environment:` is `production`. Try it:

```yaml
deploy-job:
  stage: deploy
  environment:
    name: production
  script:
    - echo "Deploying to $APP_ENV"
```

Commit and run. **You should see:** `deploy-job`'s log now prints `Deploying to production`, while `build-job` (no `environment:` set) still sees `staging`.

---

## Step 4: Caching and Artifacts Between Jobs

### 4.1 Artifacts — pass a file from one job to the next

```yaml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Environment is $APP_ENV"
    - mkdir output
    - echo "Build finished at $(date)" > output/build-report.txt
  artifacts:
    paths:
      - output/build-report.txt
    expire_in: 1 hour

test-job:
  stage: test
  script:
    - echo "Reading the file build-job created..."
    - cat output/build-report.txt

deploy-job:
  stage: deploy
  environment:
    name: production
  script:
    - echo "Deploying to $APP_ENV"
```

Commit and open the new pipeline.

**You should see:** `test-job`'s log prints the contents of `build-report.txt`, even though `test-job` never created that file — it downloaded the **artifact** from `build-job` automatically.

Click on `build-job` in the pipeline view and look for a **Browse** or download icon next to "Job artifacts" in the right panel — you can download the file yourself.

### 4.2 Cache — reuse files across pipeline runs

Add a `cache:` block to `build-job`:

```yaml
build-job:
  stage: build
  cache:
    key: build-cache
    paths:
      - output/
  script:
    - echo "Environment is $APP_ENV"
    - mkdir -p output
    - echo "Build finished at $(date)" > output/build-report.txt
  artifacts:
    paths:
      - output/build-report.txt
    expire_in: 1 hour
```

Commit, then run the pipeline **twice** (use **Run pipeline** in **Build > Pipelines** the second time).

**You should see:** in `build-job`'s log on the **second** run, near the top, lines about *"Checking cache..."* and *"Downloading cache..."* — the runner reused files instead of starting empty.

> **Artifacts vs cache, in one sentence:** artifacts move files **forward between jobs** in the same pipeline and are always kept (until they expire); cache saves time **between separate pipeline runs** and is a best-effort speed-up, not guaranteed storage.

---

## Step 5: Rules and Workflow Keywords

### 5.1 Only run a job on `main`

```yaml
deploy-job:
  stage: deploy
  environment:
    name: production
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - echo "Deploying to $APP_ENV"
```

Commit this on `main`. **You should see:** `deploy-job` still runs, because you are on `main`.

### 5.2 Test the opposite case with a branch

1. Go to **Code > Branches > New branch**. Name it `try-rules`, based on `main`.
2. Go to **Build > Pipeline editor**, switch the branch selector (top of the page) to `try-rules`.
3. No change needed — just click **Run pipeline** at the bottom, or go to **Build > Pipelines > Run pipeline** and pick branch `try-rules`.

**You should see:** the pipeline runs, but `deploy-job` does **not** appear (or shows as skipped), because `$CI_COMMIT_BRANCH` is `try-rules`, not `main`.

### 5.3 `workflow: rules` — control the whole pipeline

Add this near the top of the file (still on `try-rules` branch, or switch back to `main` — try both):

```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

Commit on `main` first. **You should see:** pipelines still run on `main` as normal.

Now go to **Code > Merge requests > New merge request**, source branch `try-rules`, target `main`, and create it.

**You should see:** a pipeline runs **for the merge request** (its pipeline source is `merge_request_event`), visible on the MR page itself, not just under **Build > Pipelines**.

Now try committing directly to a **third**, unrelated branch (create one called `no-pipeline`, edit the README slightly, commit). **You should see:** no pipeline is created at all for that branch, because neither `workflow: rules` condition matched.

> This is exactly how teams stop pipelines from running on every tiny branch, while still running them for `main` and for merge requests.

---

## Step 6: Parallel Jobs, `needs`, and DAG Pipelines

### 6.1 Parallel jobs

Update the `test` stage:

```yaml
unit-test-job:
  stage: test
  script:
    - echo "Running unit tests"

lint-job:
  stage: test
  script:
    - echo "Running lint checks"

integration-test-job:
  stage: test
  parallel: 3
  script:
    - echo "Running integration test chunk $CI_NODE_INDEX of $CI_NODE_TOTAL"
```

Commit and open the pipeline.

**You should see:** `integration-test-job` appears as **three** boxes (`1/3`, `2/3`, `3/3`) running at the same time, alongside `unit-test-job` and `lint-job` — five jobs total in the `test` stage, all in parallel.

### 6.2 `needs` — skip waiting for the whole previous stage

By default, `test-job`-type jobs wait for **all** of `build-job` to finish, and `deploy-job` waits for **all** of `test`. Let's change that for one job:

```yaml
lint-job:
  stage: test
  needs: []
  script:
    - echo "Running lint checks"
```

`needs: []` means *"don't wait for anything — start immediately."*

Commit and open the pipeline. Switch to the **Visualize** or pipeline graph view.

**You should see:** `lint-job` now has **no arrow** connecting it to `build-job`, and it starts running at the same time as `build-job`, instead of waiting for it. This is a **DAG pipeline** — jobs run based on real dependencies (`needs:`), not just stage order.

### 6.3 Make `deploy-job` depend on a specific job only

```yaml
deploy-job:
  stage: deploy
  environment:
    name: production
  needs:
    - unit-test-job
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - echo "Deploying to $APP_ENV"
```

Commit. **You should see:** in the pipeline graph, `deploy-job` now has an arrow only from `unit-test-job`, not from `lint-job` or `integration-test-job`. If those finish later, `deploy-job` does not need to wait for them.

---

## Checklist

- [ ] Added a plain variable and used it in a job
- [ ] Added a masked variable and confirmed it is hidden in the log
- [ ] Added an environment-scoped variable and saw it change per job
- [ ] Passed a file from one job to another with `artifacts:`
- [ ] Saw `cache:` reuse files on a second pipeline run
- [ ] Used `rules:` to control a single job
- [ ] Used `workflow: rules` to control whether a pipeline runs at all
- [ ] Ran a pipeline for a merge request and for a branch with no pipeline
- [ ] Used `parallel:` to split a job into multiple runs
- [ ] Used `needs:` to build a small DAG pipeline

## Quick Questions

1. What is the difference between a masked variable and a protected variable?
2. Why did `test-job` see `build-report.txt` without creating it itself?
3. What is the difference between artifacts and cache?
4. What is the difference between `rules:` on a job and `workflow: rules`?
5. What does `needs: []` do to a job's start time?

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| "Pipeline syntax is valid" never appears | Click the **Validate**/**Lint** button again after every change; check indentation (spaces, not tabs) |
| Variable value shows in the log instead of `[MASKED]` | The value may be too short/simple to mask; use a longer, more random-looking value |
| `deploy-job` never appears at all | Check your `rules:` condition matches the branch you are actually on |
| No pipeline runs on a branch | Check `workflow: rules` — if none of its conditions match, GitLab creates no pipeline |
| Artifact file not found in the next job | Check the `paths:` under `artifacts:` exactly matches the file/folder path created in the script |
| Cache doesn't seem to speed anything up | Shared runners are shared across many users; cache is best-effort and may not always hit |
| Can't find "Mask variable" checkbox | Newer GitLab versions may show a **Visibility** dropdown (Visible / Masked / Masked and hidden) instead of a checkbox — pick the masked option there |
