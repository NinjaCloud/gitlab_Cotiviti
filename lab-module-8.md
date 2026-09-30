# Lab 8 : Templates & Reusability

**Tools:** A web browser and a GitLab.com Free account. **No PowerShell, no Git commands, no downloads.**
**Time:** ~75 minutes
**Goal:** Stop repeating yourself in `.gitlab-ci.yml` by using `extends`, YAML anchors, and `include` — then refactor a messy pipeline into a clean one.

---

## What You Will Learn

- How to reuse job settings with **`extends`** and **YAML anchors**
- How to split one big pipeline file into several smaller files with **`include`** (local, project, and remote)
- What **CI/CD components** and **pipeline templates** are, and how to browse them
- How to **refactor** a repeated pipeline into a clean one, using `includes`, `rules`, and `artifacts` together

## Words to Know

| Word | Simple meaning |
|---|---|
| DRY | "Don't Repeat Yourself" — a general programming habit this module teaches for pipelines |
| `extends` | Lets a job copy settings from a hidden "template" job, then add its own |
| Hidden job | A job whose name starts with a dot (`.`), so GitLab never runs it directly — only used as a template |
| YAML anchor | A native YAML feature (`&name` / `*name`) to reuse a block of settings, without GitLab-specific keywords |
| `include` | Pulls in `.gitlab-ci.yml` content from another file — local, another project, a remote URL, or a built-in template |
| CI/CD component | A ready-made, versioned, reusable piece of pipeline, published so other projects can include it |
| Pipeline template | A ready-made starting `.gitlab-ci.yml` GitLab ships for common cases (Docker, Node, etc.) |

---

## Step 0: Two Tools You Will Use All Lab

1. **Pipeline Editor** (**Build > Pipeline editor**) — for editing `.gitlab-ci.yml` with live validation and a pipeline graph preview.
2. **Web IDE** (click **Edit > Web IDE** from any file, or the pencil/laptop icon on the repository page) — lets you create and edit **several files at once**, which you will need for `include`. It has its own **Commit** button in the left panel.

---

## Step 1: Create the Project With a Repeated (Messy) Pipeline

1. **New project > Create blank project**. Name: `templates-lab`. **Private**. Tick **Initialize repository with a README**.
2. Go to **Build > Pipeline editor** and replace the content with this **deliberately repetitive** pipeline:

```yaml
stages:
  - test
  - build

test-backend:
  stage: test
  image: node:20
  before_script:
    - echo "Installing dependencies..."
    - npm --version
  script:
    - echo "Running backend tests"

test-frontend:
  stage: test
  image: node:20
  before_script:
    - echo "Installing dependencies..."
    - npm --version
  script:
    - echo "Running frontend tests"

build-backend:
  stage: build
  image: node:20
  before_script:
    - echo "Installing dependencies..."
    - npm --version
  script:
    - echo "Building backend"

build-frontend:
  stage: build
  image: node:20
  before_script:
    - echo "Installing dependencies..."
    - npm --version
  script:
    - echo "Building frontend"
```

3. Commit with message `Add repetitive starting pipeline` directly to `main`.

**You should see:** the pipeline runs with 4 jobs. Notice `image: node:20` and the whole `before_script:` block are copy-pasted **four times**. That repetition is what we will remove.

---

## Step 2: Reuse Settings With `extends`

1. In the Pipeline Editor, add a **hidden job** at the top (name starts with a dot) and rewrite the four real jobs to extend it:

```yaml
stages:
  - test
  - build

.node-job:
  image: node:20
  before_script:
    - echo "Installing dependencies..."
    - npm --version

test-backend:
  extends: .node-job
  stage: test
  script:
    - echo "Running backend tests"

test-frontend:
  extends: .node-job
  stage: test
  script:
    - echo "Running frontend tests"

build-backend:
  extends: .node-job
  stage: build
  script:
    - echo "Building backend"

build-frontend:
  extends: .node-job
  stage: build
  script:
    - echo "Building frontend"
```

2. Click **Validate**, then commit with message `Use extends to remove duplication`.

**You should see:** the pipeline graph looks exactly the same as before (still 4 jobs), but the file is shorter, and `.node-job` itself does **not** appear as a job to run — hidden jobs never run directly.

3. Try it out: change `image: node:20` to `image: node:22` in `.node-job` **only**, commit, and open the new pipeline. All four jobs now use Node 22 — you only changed it in one place.

---

## Step 3: Reuse Settings With YAML Anchors

`extends` is a GitLab feature. **YAML anchors** are plain YAML, so they work slightly differently — useful when you want to reuse a list or a small snippet, not a whole job.

Replace the `before_script:` repetition with an anchor:

```yaml
stages:
  - test
  - build

.node-job:
  image: node:22

.install-deps: &install-deps
  - echo "Installing dependencies..."
  - npm --version

test-backend:
  extends: .node-job
  stage: test
  before_script: *install-deps
  script:
    - echo "Running backend tests"

test-frontend:
  extends: .node-job
  stage: test
  before_script: *install-deps
  script:
    - echo "Running frontend tests"

build-backend:
  extends: .node-job
  stage: build
  before_script: *install-deps
  script:
    - echo "Building backend"

build-frontend:
  extends: .node-job
  stage: build
  before_script: *install-deps
  script:
    - echo "Building frontend"
```

Commit with message `Add YAML anchor for shared before_script`.

**You should see:** the pipeline still passes with 4 jobs. `&install-deps` **defines** the anchor; `*install-deps` **reuses** it — like a copy-paste that GitLab's YAML parser does for you before the pipeline even starts.

> **Rule of thumb:** use `extends` to share whole sets of job settings (image, rules, tags, scripts...). Use YAML anchors for small, plain reusable snippets like a list of commands or variables.

---

## Step 4: `include` — Local Includes

Right now everything lives in one `.gitlab-ci.yml`. Let's split it into files, the way larger real projects do.

1. Open the **Web IDE** (from the repository page, click the **Edit** dropdown button and choose **Web IDE**).
2. In the file tree, create a new folder called `ci` (right-click the root, **New folder**).
3. Inside `ci`, create a file **`ci/test.yml`**:

```yaml
.node-job:
  image: node:22

.install-deps: &install-deps
  - echo "Installing dependencies..."
  - npm --version

test-backend:
  extends: .node-job
  stage: test
  before_script: *install-deps
  script:
    - echo "Running backend tests"

test-frontend:
  extends: .node-job
  stage: test
  before_script: *install-deps
  script:
    - echo "Running frontend tests"
```

4. Create **`ci/build.yml`**:

```yaml
build-backend:
  extends: .node-job
  stage: build
  before_script: *install-deps
  script:
    - echo "Building backend"

build-frontend:
  extends: .node-job
  stage: build
  before_script: *install-deps
  script:
    - echo "Building frontend"
```

> Note: `ci/build.yml` reuses `.node-job` and `*install-deps` even though they are **defined in `ci/test.yml`** — this works because GitLab merges all included files together before running the pipeline.

5. Replace the root **`.gitlab-ci.yml`** with:

```yaml
stages:
  - test
  - build

include:
  - local: 'ci/test.yml'
  - local: 'ci/build.yml'
```

6. In the Web IDE's left panel, click **Source control**, write a commit message `Split pipeline into local includes`, and click **Commit** (commit directly to `main`).

**You should see:** in **Build > Pipeline editor**, the **Full configuration** tab (or "View merged YAML") shows all four jobs combined from both files. In **Build > Pipelines**, the pipeline still runs exactly as before — the split is invisible to the pipeline itself, it only makes the files easier to manage.

---

## Step 5: `include` — Project Include

A **project include** pulls a file from a **different GitLab project** — handy for sharing pipeline pieces across many repositories.

### 5.1 Create a small shared-templates project

1. **New project > Create blank project**. Name: `ci-templates`. Visibility: **Public** (so it is simple to include from). Tick **Initialize repository with a README**.
2. Open its **Web IDE**, create a file **`lint.yml`** at the root:

```yaml
lint-job:
  stage: test
  image: node:22
  script:
    - echo "Running a shared lint job from ci-templates"
```

3. Commit directly to `main` with message `Add shared lint job`.

### 5.2 Include it from your `templates-lab` project

Back in **templates-lab > Build > Pipeline editor**, update the `include:` block:

```yaml
stages:
  - test
  - build

include:
  - local: 'ci/test.yml'
  - local: 'ci/build.yml'
  - project: '<your-namespace>/ci-templates'
    ref: main
    file: 'lint.yml'
```

Replace `<your-namespace>` with your actual username or group. Commit with message `Add project include for shared lint job`.

**You should see:** a new `lint-job` appears in the pipeline, defined in a **completely different project**, but running as part of this one. Change something in `ci-templates/lint.yml` later, and every project that includes it picks up the change automatically.

---

## Step 6: `include` — Remote Include

A **remote include** pulls a file from any reachable URL that returns plain YAML — often used for files hosted outside GitLab, or files you want to reference without GitLab project permissions.

We will reuse the file we already made, but reference it by its **raw URL** instead of `project:`.

1. In `ci-templates`, open `lint.yml`, click the **three-dot menu**, and choose **Open raw**. Copy that URL — it looks like:

```
https://gitlab.com/<your-namespace>/ci-templates/-/raw/main/lint.yml
```

2. In `templates-lab`, replace the `project:` include with a `remote:` include:

```yaml
include:
  - local: 'ci/test.yml'
  - local: 'ci/build.yml'
  - remote: 'https://gitlab.com/<your-namespace>/ci-templates/-/raw/main/lint.yml'
```

Commit with message `Switch to remote include`.

**You should see:** the pipeline behaves the same as Step 5 — `lint-job` still appears. The difference is **how** GitLab fetched the file: as a plain HTTPS download, not through project permissions. This is why `remote:` only works for **public** URLs with no authentication required.

> **When to use which:** `project:` include is better inside your own GitLab instance (respects permissions, supports private projects you have access to). `remote:` include works for any public URL, even outside GitLab, but cannot read private content.

---

## Step 7: `include` — Templates and CI/CD Components (Explore)

### 7.1 Browse built-in templates

1. Go to **Code > Repository > Files**.
2. Click **+ > New file**, and set the file name to `.gitlab-ci.yml` (you will not actually commit this — it's just to see the picker).
3. Look for a dropdown near the top labelled something like **"Apply a template"**. Open it and scroll through the list (Docker, Bash, Node.js, and many more).
4. Pick any one to preview its content in the editor, **then leave without committing** (navigate away, or click **Cancel**).

**You should see:** GitLab ships many ready-made `.gitlab-ci.yml` starting points you can browse without writing any YAML yourself.

### 7.2 The `include: template:` keyword

Any file from that list can also be pulled into your own pipeline with `include: template:` instead of copying its content. For example:

```yaml
include:
  - template: 'Jobs/Build.gitlab-ci.yml'
```

> **Try this only as a stretch, on a throwaway branch.** Some built-in templates add several jobs and expect things like a Dockerfile or extra configuration to fully succeed — that is fine, the goal here is to see new jobs **appear** in your pipeline graph from one line of `include:`, not necessarily to get every job green.

### 7.3 CI/CD Components (newer, more flexible reusable pipelines)

1. Go to **Search or go to > Explore > CI/CD Catalog** (or search "CI/CD Catalog" from the top search bar).
2. Browse the list of published **components** — small, versioned, reusable pipeline pieces published by GitLab or the community.
3. Open one that looks simple. Its page shows a ready-to-copy `include:` snippet, for example:

```yaml
include:
  - component: gitlab.org/some-namespace/some-component@1.0.0
```

**You should see:** every component page gives you the exact `include:` line to paste — you do not need to memorise the syntax, only know where to look.

> The GitLab web interface changes over time, so if a menu name above looks slightly different on your screen, look for wording close to it — the underlying feature is stable even when labels move.

---

## Step 8: Hands-on Lab — Refactor a Pipeline Using Includes, Rules, and Artifacts

Now combine everything from this module and the previous one into one clean pipeline.

### 8.1 Starting point

Go back to `templates-lab`'s root **`.gitlab-ci.yml`** in the Pipeline Editor and set it to:

```yaml
stages:
  - test
  - build

include:
  - local: 'ci/test.yml'
  - local: 'ci/build.yml'
```

(Remove the `project:`/`remote:` includes for this final exercise, to keep it self-contained — you already proved they work.)

### 8.2 Add rules so `build` jobs only run on `main`

Open the Web IDE and edit **`ci/build.yml`**:

```yaml
build-backend:
  extends: .node-job
  stage: build
  before_script: *install-deps
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - echo "Building backend"
    - mkdir -p dist
    - echo "backend built" > dist/backend.txt
  artifacts:
    paths:
      - dist/backend.txt
    expire_in: 1 hour

build-frontend:
  extends: .node-job
  stage: build
  before_script: *install-deps
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - echo "Building frontend"
    - mkdir -p dist
    - echo "frontend built" > dist/frontend.txt
  artifacts:
    paths:
      - dist/frontend.txt
    expire_in: 1 hour
```

### 8.3 Add a job that uses both artifacts

Add a new file **`ci/package.yml`**:

```yaml
package-job:
  stage: build
  needs:
    - build-backend
    - build-frontend
  script:
    - echo "Packaging build output..."
    - cat dist/backend.txt
    - cat dist/frontend.txt
```

Include it from the root file:

```yaml
stages:
  - test
  - build

include:
  - local: 'ci/test.yml'
  - local: 'ci/build.yml'
  - local: 'ci/package.yml'
```

### 8.4 Commit everything and run

Commit all changed files with message `Refactor pipeline with includes, rules, and artifacts` (do this from the Web IDE's **Source control** panel so all three files go in one commit).

**You should see:** on `main`, the pipeline runs `test-backend`, `test-frontend` in `test`, then `build-backend`, `build-frontend`, `package-job` in `build`. `package-job`'s log shows the contents of both artifact files, proving it correctly received files from two different jobs, defined in two different included files.

### 8.5 Prove the rule works

1. Create a branch `feature-check`, based on `main`, and make a tiny edit to `README.md` on that branch.
2. Run the pipeline on `feature-check` (**Build > Pipelines > Run pipeline**, pick that branch).

**You should see:** `test-backend` and `test-frontend` still run (no rule limiting them), but `build-backend`, `build-frontend`, and `package-job` are **skipped**, because the `rules:` condition only matches `main`.

---

## Checklist

- [ ] Removed duplication using `extends` and a hidden job
- [ ] Removed duplication using a YAML anchor
- [ ] Split the pipeline into local includes (`ci/test.yml`, `ci/build.yml`)
- [ ] Created a second project and used a **project include**
- [ ] Reused the same file with a **remote include**
- [ ] Browsed built-in templates and the CI/CD Catalog
- [ ] Refactored the final pipeline with includes, `rules:`, `artifacts:`, and `needs:` together
- [ ] Confirmed build jobs are skipped on a non-`main` branch

## Quick Questions

1. What is the difference between `extends` and a YAML anchor?
2. Why did `ci/build.yml` still work even though `.node-job` is defined in `ci/test.yml`?
3. When would you use a project include instead of a remote include?
4. What did `include: template:` let you do that `include: local:` cannot?
5. In the final refactor, why does `package-job` need `needs:` instead of just relying on stage order?

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| "Pipeline syntax is valid" never appears in Pipeline Editor | Check indentation carefully; also confirm every included file itself has valid YAML |
| Job from a local include never shows up | Check the path after `local:` matches the real file path and name exactly, including the `ci/` folder |
| Project include fails with "not found" or permission error | Confirm the namespace/project path is correct and that the other project is **Public**, or that you have access to it |
| Remote include fails | The URL must return plain YAML with no login required; test by opening the raw URL directly in a new browser tab |
| Web IDE won't let you create a folder | Right-click the root of the file tree (not a file) and choose **New folder**, then create the file inside it |
| Anchor (`*install-deps`) shows an error | The anchor (`&install-deps`) must be defined **somewhere GitLab reads before** the job that uses it — check spelling matches exactly, including the leading `&`/`*` |
| Built-in template jobs fail | Expected for some templates without extra setup; this lab only asks you to see the jobs appear, not pass |
