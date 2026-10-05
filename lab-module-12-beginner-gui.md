# Lab 12 (Beginner, GUI-only): GitLab Administration & Best Practices

**Tools:** A web browser and a GitLab.com Free account. A second GitLab account is helpful but optional. You will also use the free website https://webhook.site.
**Time:** ~90 minutes
**Goal:** Organise projects with groups and roles, protect branches and tags, see webhooks and the API in action, fix a broken pipeline, then explore the Container Registry and review a CI/CD best-practices checklist.

---

## What You Will Learn

- How **groups**, **subgroups**, and **permission levels** work
- How to use **protected branches** and **protected tags**
- What **webhooks**, **integrations**, and the **GitLab API** are
- How to find and fix common **pipeline failures**
- How to explore the **Container Registry** and review a **CI/CD best-practices checklist**

## Words to Know

| Word | Simple meaning |
|---|---|
| Group | A folder of projects with shared members and settings |
| Subgroup | A group inside another group (for example `backend` inside `devops-lab`) |
| Role / permission level | What a member is allowed to do: Guest, Reporter, Developer, Maintainer, Owner |
| Protected branch | A branch with rules about who can push or merge |
| Protected tag | A tag that only chosen roles can create or delete |
| Webhook | GitLab sends a message (HTTP request) to a web address when something happens |
| API | A way for programs to read and change GitLab data |
| Access token | A password-like code for the API or for a tool |
| Deploy token | A read-only token for pulling code or images |

---

## Free Tier Notes

| Feature | On GitLab Free? | In this lab |
|---|---|---|
| Groups, subgroups, roles | Yes (GitLab.com Free has a small member limit per top-level group; check the pricing page) | Hands-on |
| Protected branches and tags | Yes | Hands-on |
| Webhooks and API | Yes | Hands-on |
| Many integrations (Slack, Jira, and others) | Mostly yes; some advanced features are Premium | Overview |
| Code Owners approvals, push rules, protected environments | **No (Premium+)** | Mentioned only |

> GitLab changes tier packaging from time to time. Check https://about.gitlab.com/pricing/ if something looks different.

---

## Step 1: Groups, Subgroups, and Permission Levels (15 min)

### 1.1 Create a group and subgroups

1. Click **+** at the top and choose **New group > Create group**. Name: `devops-lab-YOURNAME` (the group name must be unique). Visibility: **Private**. Click **Create group**.
2. On the group page click **New subgroup**. Name: `backend`. Create it.
3. Go back to the parent group and create another subgroup named `frontend`.

**You should see:** the parent group listing two subgroups, `backend` and `frontend`.

### 1.2 Create projects inside a subgroup

1. Open the `backend` subgroup and click **New project > Create blank project**.
2. Name: `api-service`. Tick **Initialize repository with a README**. Click **Create project**.
3. Repeat to create a second project in `backend` named `troubleshoot-lab` (also with a README).

**You should see:** a project path like `devops-lab-yourname/backend/api-service`.

### 1.3 Permission levels

| Role | Can do (simplified) |
|---|---|
| **Guest** | View projects and issues; leave comments (limited on private projects) |
| **Reporter** | Read the code, create issues, download artifacts; cannot push code |
| **Developer** | Push to non-protected branches, create merge requests, run pipelines |
| **Maintainer** | Manage branches, protect branches, edit project settings, merge to protected branches |
| **Owner** | Everything, including deleting the group and managing members |

> A newer **Planner** role (for project managers, issues and boards only) may also appear in your version.

### 1.4 Members and inheritance

1. In the **group**, go to **Manage > Members**. You are listed as **Owner**.
2. If you have a second account, click **Invite members**, enter that username, choose **Developer**, and invite them. Accept the invite from the other account.
3. Open the project `api-service` and go to **Manage > Members**.

**You should see:** the group members listed here too, marked as **inherited** from the group. A person added to a parent group automatically has the same role in every subgroup and project below it. A role can be raised in a lower level, but never lowered below the inherited one.

> **Solo mode:** if you do not have a second account, just look at your own **Owner** entry marked as inherited in the project, and read the table above.

---

## Step 2: Protected Branches and Protected Tags (15 min)

All of this happens in the `api-service` project.

### 2.1 Look at the default protection

1. Go to **Settings > Repository** and expand **Protected branches**.

**You should see:** `main` is already protected. Note who may **push and merge** and who may **merge**.

### 2.2 Protect a wildcard of branches

1. Click **Add protected branch**.
2. **Branch:** type `release-*` (the `*` means "anything").
3. **Allowed to merge:** `Maintainers`. **Allowed to push and merge:** `No one`. Leave **Allowed to force push** off.
4. Click **Protect**.

### 2.3 Prove it works

1. Go to **Code > Branches > New branch**. Name: `release-1.0`, based on `main`.
2. On the `release-1.0` branch (use the branch selector), open `README.md` and click **Edit**.
3. Make a small change and try to commit.

**You should see:** GitLab will not let you commit directly to the protected branch. It offers to commit to a **new branch** and start a merge request instead. That is protection working: changes must go through review.

### 2.4 Protected tags

1. In **Settings > Repository**, expand **Protected tags** and click **Add tag**.
2. **Tag:** `v*`. **Allowed to create:** `Maintainers`. Click **Protect**.
3. Go to **Code > Tags > New tag**. Tag name: `v1.0.0`, create from `main`, and click **Create tag**.

**You should see:** the tag is created, shown with a **protected** badge. A Developer could not create or delete tags that start with `v`.

> **Why protect tags?** Release tags such as `v1.0.0` usually mark production releases. Protecting them stops accidental changes or deletion. Protected **variables** (from Lab 7) are only available on protected branches and tags, so protection also keeps secrets safe.

> **Premium only:** Code Owners approvals, required approvals, and push rules are not on Free.

---

## Step 3: Webhooks, Integrations, and the GitLab API (15 min)

### 3.1 Webhooks

1. Open https://webhook.site in a new browser tab. It gives you a **unique URL**. Copy it. (Anyone with the URL can see what is sent, so only use test data.)
2. In `api-service`, go to **Settings > Webhooks** and click **Add new webhook**.
3. Paste the URL. Under **Trigger** tick **Push events**, **Merge request events**, and **Pipeline events**.
4. Click **Add webhook**.
5. In the webhook list, click **Test > Push events**.
6. Go back to the webhook.site tab.

**You should see:** a request appeared with a JSON body describing the push (project name, commits, user).

7. Now make a real change: edit `README.md` on `main` and commit. Look at webhook.site again.

**You should see:** a new request for your real commit. This is how tools like chat apps, build servers, or custom scripts learn about GitLab events.

8. Delete the webhook when finished (click **Delete** next to it).

### 3.2 Integrations (overview)

Open **Settings > Integrations** and scroll through the list.

**You should see:** many ready-made connections, for example Slack, Discord, Microsoft Teams, Jira, and email notifications. Open one (for example **Slack notifications**) to see the settings form, then leave without saving.

| Webhook | Integration |
|---|---|
| Generic: sends data to **any** URL you choose | Built for a specific tool, with its own settings |
| You write the code that receives it | GitLab and the other tool already understand each other |

### 3.3 The GitLab API (overview)

The API lets programs read and change GitLab. Your browser can read public data without a token.

1. Open this address in a new tab:

```
https://gitlab.com/api/v4/projects/gitlab-org%2Fgitlab-runner
```

**You should see:** a page of JSON data about a public project (name, description, stars).

2. Try a second address that lists its branches (it may show only the first page):

```
https://gitlab.com/api/v4/projects/gitlab-org%2Fgitlab-runner/repository/branches
```

> The `%2F` is how a `/` is written inside a URL.

3. For your **private** projects the API needs a token. Look (do not create anything yet) at **your avatar > Preferences > Access tokens**.

| Token type | Used for |
|---|---|
| Personal access token | Scripts acting as you |
| Project / group access token | Scripts for one project or group (like a bot) |
| Deploy token | Read-only access to code or the registry |

**Token best practices:** give the fewest permissions needed (for example `read_api`), set an expiry date, never put a token in the repository, and revoke it when it is not needed.

For reference, a command-line call with a token looks like this (not needed in this lab):

```
curl --header "PRIVATE-TOKEN: <your_token>" "https://gitlab.com/api/v4/projects/<project_id>"
```

Other APIs: GitLab also has a **GraphQL** explorer at https://gitlab.com/-/graphql-explorer, and the full reference at https://docs.gitlab.com/ee/api/.

---

## Step 4: Troubleshoot a Broken Pipeline (20 min)

Open the project `troubleshoot-lab` (in the `backend` subgroup).

### 4.1 Load the broken pipeline

Go to **Build > Pipeline editor** and paste this exactly:

```yaml
stages:
  - build
  - test

build-job:
  stage: build
  image: python:3.12-slim
  script:
    - pyhton --version
    - mkdir out
    - echo "built" > out/result.txt
  artifacts:
    paths:
      - output/result.txt

test-job:
  stage: tests
  image: python:3.12-slim
  script:
    - cat out/result.txt
    - echo "Deploying to $DEPLOY_ENV"
    - test -n "$DEPLOY_ENV"

deploy-job:
  stage: deploy
  needs:
    - buid-job
  script:
    - echo "deploying"
```

Commit with message `Add broken pipeline` to `main`.

### 4.2 Round 1: errors found before the pipeline runs

Open the **Pipeline editor** and click **Validate** (or look at **Build > Pipelines**: the pipeline may show an error badge).

**You should see:** messages about a stage that does not exist and a job that is needed but does not exist. These are **configuration errors**: GitLab refuses to create the pipeline at all. Read each message, find the line, and fix it. Commit when **Validate** says the syntax is valid.

### 4.3 Round 2: errors found while jobs run

Open the new pipeline and click the red job. Read the **bottom** of the log first, then scroll up.

**You should see:** a command that cannot be found (exit code 127) in `build-job`. Fix it and commit.

Run the pipeline again.

**You should see:** a new failure. Read the log, find the cause, and fix it. Repeat until only one failure remains. Then read the last failure carefully: it is about a **missing variable**. Fix it in the YAML (you can add a `variables:` block) or in **Settings > CI/CD > Variables**.

Keep going until the whole pipeline is green.

### 4.4 Answer key (try first, then check)

| # | What was wrong | Type | Fix |
|---|---|---|---|
| 1 | `test-job` uses stage `tests` | Config | Change to `test` |
| 2 | `deploy-job` uses stage `deploy`, which is not in `stages:` | Config | Add `deploy` to the `stages:` list |
| 3 | `needs: buid-job` is misspelled | Config | Change to `build-job` |
| 4 | `pyhton --version` is a typo | Runtime (exit 127) | Change to `python --version` |
| 5 | Artifact path `output/result.txt` does not match `out/result.txt` | Runtime (file not found) | Change `artifacts: paths` to `out/result.txt` |
| 6 | `DEPLOY_ENV` is never set | Runtime (exit 1) | Add `variables: DEPLOY_ENV: staging` to the job or file |

### 4.5 Where to look when a pipeline fails

| Symptom | Where to look / what to try |
|---|---|
| Pipeline did not start, red badge | **Pipeline editor > Validate**: it is a config error |
| Job red, "command not found" | Job log: typo or tool missing from the `image:` |
| Job red, "No such file" | Artifact or cache paths; was the file created, and is `needs`/stage order right? |
| Job stuck on **pending** | No runner matches the job's `tags:`, or runners are busy or offline |
| Job fails only on some branches | **Protected** variables are not available on unprotected branches |
| "toomanyrequests" when pulling an image | Docker Hub rate limit; retry later or use another registry |
| Job fails randomly | Add `retry:` for system failures; check network or flaky tests |
| Pipeline takes too long | **Analyze > CI/CD analytics** shows duration and success rate over time |

---

## Step 5: Hands-on Lab, Explore the Container Registry and Review a Best-Practices Checklist (25 min)

### 5.1 Push some images to the GitLab Container Registry

If you completed Lab 11 you can use that project instead. Otherwise, use `api-service`:

1. Create `index.html`:

```html
<!DOCTYPE html>
<html>
  <body>
    <h1>api-service</h1>
    <p>Version 1</p>
  </body>
</html>
```

2. Create `Dockerfile`:

```
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
```

3. In **Build > Pipeline editor**, replace the content with:

```yaml
stages:
  - build

build-image:
  stage: build
  image: docker:27
  services:
    - docker:27-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" -t "$CI_REGISTRY_IMAGE:latest" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
    - docker push "$CI_REGISTRY_IMAGE:latest"
```

4. Commit to `main` and wait for the pipeline to pass.
5. Change `Version 1` to `Version 2` in `index.html`, commit, and wait for the pipeline. Do this once more with `Version 3`.

### 5.2 Explore the registry

Go to **Deploy > Container Registry**.

**You should see:** an image repository named after your project. Click it and explore:

1. The **tags** list: one tag per commit plus `latest`.
2. Click a tag to see its **digest**, **size**, and **published time**.
3. Click the copy icon next to a tag to get its `docker pull` command.
4. Click **CLI commands** (or the quick-start box) to see the `docker login`, `docker build`, and `docker push` pattern.
5. Select one old commit tag and click **Delete**.

**You should see:** that tag disappears while `latest` and the others stay.

### 5.3 Cleanup policy

1. Go to **Settings > Packages and registries** and expand **Container registry** (or find **Cleanup policies**).
2. Look at the options: how often to run, how many tags to **keep**, remove tags **older than** a number of days, and which tags to **keep or remove by name pattern**.
3. Tick enable, choose keep **5** tags per image name, older than **7 days**, and click **Save**.

> **Why?** Every pipeline pushes a new image. Without cleanup, the registry fills with old images and uses storage.

### 5.4 Deploy token (read-only access)

1. Go to **Settings > Repository** and expand **Deploy tokens**.
2. Name: `registry-read`. Tick only **read_registry**. Click **Create deploy token**.
3. Note the **username** and **token** shown (the token is only shown once). Then delete the token if you do not need it.

**Why:** a deploy token lets a server or Kubernetes cluster pull images from a private registry without using your personal account.

### 5.5 Docker Hub versus GitLab Container Registry

| | Docker Hub | GitLab Container Registry |
|---|---|---|
| Address | `docker.io/user/image` | `registry.gitlab.com/group/project/image` |
| Linked to your code | No | Yes: tied to the project and its permissions |
| Login in pipelines | Needs your own variables | Automatic with `$CI_REGISTRY_USER` and `$CI_REGISTRY_PASSWORD` |
| Public sharing | Easy and well known | Possible if the project is public |
| Pull limits | Rate limits for anonymous or free users | Different limits, tied to GitLab |

If you did Lab 11, open Docker Hub and compare its **Tags** page with the GitLab registry page.

### 5.6 Review the CI/CD best-practices checklist

Open the `.gitlab-ci.yml` of `api-service` and mark each item **Yes**, **No**, or **N/A**.

| # | Best practice | How to check |
|---|---|---|
| 1 | No secrets or passwords in the repository | Search the repo; use **Settings > CI/CD > Variables** |
| 2 | Secrets stored as masked (and protected) variables | **Settings > CI/CD > Variables** |
| 3 | Docker and tool images use a **specific version** (not `latest`) | Look at each `image:` line |
| 4 | Pipelines only run when useful (`rules:` / `workflow:`) | Look for `rules:` and `workflow:` |
| 5 | Fast jobs run first (fail early) | Stage order |
| 6 | Jobs that can run in parallel do (`needs:` used where helpful) | Pipeline graph |
| 7 | Dependencies are cached | Look for `cache:` |
| 8 | Artifacts have an expiry (`expire_in:`) | Look at each `artifacts:` |
| 9 | Repeated config is reused (`extends`, `include`) | No copy-paste blocks |
| 10 | Old pipelines are cancelled when new ones start (`interruptible: true`) | Look for `interruptible` |
| 11 | Jobs have a sensible `timeout:` | Look for `timeout` or project default |
| 12 | Flaky infrastructure failures are retried (`retry:`) | Look for `retry` |
| 13 | Tests and security scans run on every merge request | Pipeline for the MR |
| 14 | Production deploys are manual or approved | `when: manual`; protected environments (Premium) |
| 15 | `main` and release branches are protected | **Settings > Repository** |
| 16 | Release tags are protected | **Settings > Repository** |
| 17 | Members have the least role they need | **Manage > Members** |
| 18 | Tokens have minimal scope and an expiry date | **Access tokens** |
| 19 | Container images are small and cleaned up regularly | Cleanup policy, `.dockerignore` |
| 20 | Pipeline duration and failure rate are reviewed | **Analyze > CI/CD analytics** |

### 5.7 Fix three gaps

Pick **three** items you marked **No**. Apply at least one in the Pipeline editor. A good starter, which covers items 4, 10, 11 and 12, is to add this to the **top** of your file:

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

default:
  interruptible: true
  timeout: 15 minutes
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
```

Click **Validate**, commit, and confirm the pipeline still passes.

Finally open **Analyze > CI/CD analytics** and look at the pipeline success rate and duration. Write down one more improvement you would make.

---

## Checklist

- [ ] Created a group with two subgroups and a project inside a subgroup
- [ ] Explained the five main roles and how inheritance works
- [ ] Protected a branch pattern and saw direct commits blocked
- [ ] Protected a tag pattern and created a protected tag
- [ ] Sent a webhook to webhook.site and read the payload
- [ ] Read public data through the GitLab API in the browser
- [ ] Fixed the six bugs in the broken pipeline
- [ ] Explored the Container Registry (tags, digest, delete, cleanup policy)
- [ ] Created and understood a deploy token
- [ ] Reviewed the best-practices checklist and fixed at least one gap

## Quick Questions

1. A user is a Developer in the parent group. What role do they have in a subgroup project?
2. What does "Allowed to push: No one" on a protected branch force people to do?
3. What is the difference between a webhook and an integration?
4. What is the difference between a configuration error and a runtime error in a pipeline?
5. Why should container registries have a cleanup policy?
6. Name two best practices that reduce pipeline time and two that improve security.

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| Group name already taken | Add your name or a number to make it unique |
| Cannot invite more members | GitLab.com Free limits members on a private top-level group; remove someone or skip inviting |
| Cannot find **Protected tags** | It is under **Settings > Repository**; expand the section |
| Direct commit to `release-1.0` is allowed | Check **Allowed to push and merge** is set to **No one** and that the branch name starts with `release-` |
| Webhook shows an error | Check the URL is copied exactly and that the webhook.site tab is still open |
| API address shows "404 Not Found" | The `%2F` between group and project must be kept; private projects need a token |
| Pipeline in `troubleshoot-lab` shows no errors in Validate | Check you pasted the file exactly, including the misspellings, and committed to `main` |
| **Deploy > Container Registry** is empty | Check the pipeline passed and the project registry is enabled under **Settings > General > Visibility, project features, permissions** |
| Cleanup policy cannot be saved | Choose values for every field, or look for a **Set cleanup rules** link in the same section |
| Docker job fails with "Cannot connect to the Docker daemon" | Check the `services: docker:27-dind` block and `DOCKER_TLS_CERTDIR` variable and their indentation |
| Menu names look different | GitLab renames menus now and then; look for similar wording |
