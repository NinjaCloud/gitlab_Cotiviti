# Lab 12 : GitLab Best Practices

**Tools:** A web browser and a GitLab.com Free account. A second GitLab account is helpful but optional. You will also use the free website https://webhook.site.
**Time:** ~30 minutes
**Goal:** see webhooks and the API in action, CI/CD best-practices checklist.

---

## What You Will Learn


- What **webhooks**, **integrations**, and the **GitLab API** are
- **CI/CD best-practices checklist**

## Words to Know

| Word | Simple meaning |
|---|---|
| Webhook | GitLab sends a message (HTTP request) to a web address when something happens |
| API | A way for programs to read and change GitLab data |


---



## Step 1: Webhooks, Integrations, and the GitLab API (15 min)

### 1.1 Webhooks

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

### 1.2 Integrations (overview)

Open **Settings > Integrations** and scroll through the list.

**You should see:** many ready-made connections, for example Slack, Discord, Microsoft Teams, Jira, and email notifications. Open one (for example **Slack notifications**) to see the settings form, then leave without saving.

| Webhook | Integration |
|---|---|
| Generic: sends data to **any** URL you choose | Built for a specific tool, with its own settings |
| You write the code that receives it | GitLab and the other tool already understand each other |

### 1.3 The GitLab API (overview)

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


### 2.1 Docker Hub versus GitLab Container Registry

| | Docker Hub | GitLab Container Registry |
|---|---|---|
| Address | `docker.io/user/image` | `registry.gitlab.com/group/project/image` |
| Linked to your code | No | Yes: tied to the project and its permissions |
| Login in pipelines | Needs your own variables | Automatic with `$CI_REGISTRY_USER` and `$CI_REGISTRY_PASSWORD` |
| Public sharing | Easy and well known | Possible if the project is public |
| Pull limits | Rate limits for anonymous or free users | Different limits, tied to GitLab |

If you did Lab 11, open Docker Hub and compare its **Tags** page with the GitLab registry page.

### 2.2 Review the CI/CD best-practices checklist

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

