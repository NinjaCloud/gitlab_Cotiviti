# Lab 4: Issues & Project Management (Windows PowerShell)

**Platform:** GitLab.com Free tier (or self-managed Free/CE)
**Shell:** Windows PowerShell 5.1 or PowerShell 7+ on Windows 10/11
**Duration:** ~90 minutes
**Level:** Beginner to Intermediate

---

## Learning Objectives

1. Track work with issues, labels, milestones, and issue boards
2. Explain epics and roadmaps and what they need (Premium), plus a Free-tier workaround
3. Link issues to merge requests so they close automatically
4. Use time tracking and estimate weight
5. Complete the full flow: **raise an issue, create a branch, open a merge request against it**

---

## Free Tier Notes (Read First)

| Feature | Free tier? | Approach in this lab |
|---|---|---|
| Issues, comments, assignees, due dates | Yes | Fully covered |
| Project and group labels | Yes | Fully covered |
| Scoped labels (`key::value`) | **Premium+** | Plain labels with a naming convention |
| Milestones | Yes | Fully covered |
| Issue boards | Yes (project boards; multiple boards per project) | Label-based lists |
| **Epics and Roadmaps** | **Premium+** | Overview only, plus a parent-issue workaround |
| Issue weight | **Premium+** | `weight-N` labels as a Free workaround |
| Time tracking (`/estimate`, `/spend`) | Yes | Fully covered |
| Issue-MR linking and auto-close | Yes | Fully covered |

> GitLab changes tier packaging from time to time. Confirm at https://about.gitlab.com/pricing/ and in the docs.

---

## PowerShell Tips Used Throughout

- Use **Windows Terminal** or **PowerShell** (not Command Prompt).
- Put each command on its own line; `&&` only works in PowerShell 7+.
- Multi-line files use **single-quoted here-strings** `@' ... '@`; the closing `'@` must be at the start of a line.
- Use `-Encoding ascii` when writing files so no byte-order mark (BOM) is added.
- Check version: `$PSVersionTable.PSVersion`

---

## Prerequisites

1. **Git for Windows:**

```powershell
git --version
```

If missing:

```powershell
winget install --id Git.Git -e
```

Reopen PowerShell afterwards.

2. **Identity (one time):**

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

3. **Authentication.** Either:
   - **HTTPS + personal access token:** **Preferences > Access tokens**, scopes `read_repository` and `write_repository`; use the token as the password when Git Credential Manager prompts, or
   - **SSH key:**

```powershell
ssh-keygen -t ed25519 -C "you@example.com"
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
ssh -T git@gitlab.com
```

(Paste the key at **Preferences > SSH Keys** before running the last command.)

4. *(Optional)* A second user added as Developer, to practice assignment.

---

## Part 1: Set Up the Project (10 min)

1. **New project > Create blank project**. Name: `pm-lab`. Private. Initialize with README.
2. Clone it:

```powershell
mkdir $HOME\gitlab-labs -Force
cd $HOME\gitlab-labs
git clone https://gitlab.com/<your-namespace>/pm-lab.git
cd pm-lab
```

3. Add starter files:

```powershell
@'
def add(a, b):
    return a + b
'@ | Set-Content -Path calculator.py -Encoding ascii

@'
stages: [test]

syntax-check:
  stage: test
  image: python:3.12-slim
  script:
    - python -m py_compile calculator.py
'@ | Set-Content -Path .gitlab-ci.yml -Encoding ascii
```

4. Commit and push:

```powershell
git add .
git commit -m "Initial calculator app"
git push origin main
```

> `warning: LF will be replaced by CRLF` is normal on Windows and harmless here.

**Checkpoint:** `main` has README, `calculator.py`, CI file.

---

## Part 2: Labels (10 min)

### 2.1 Create labels

Go to **Manage > Labels > New label** and create:

| Name | Colour | Purpose |
|---|---|---|
| `type-feature` | Blue | New functionality |
| `type-bug` | Red | Defects |
| `type-epic` | Dark blue | Parent tracking issue (Free workaround) |
| `priority-high` | Orange | Urgent work |
| `priority-low` | Grey | Nice to have |
| `status-todo` | Light blue | Not started |
| `status-doing` | Yellow | In progress |
| `status-review` | Purple | Awaiting review |
| `status-done` | Green | Complete |
| `weight-1`, `weight-3`, `weight-5` | Teal | Weight estimate (Free workaround) |

> **Premium note:** With scoped labels you would name these `status::todo`, `status::doing`, and GitLab would allow only one `status::` label per issue. On Free you enforce that yourself by removing the old label when adding the new one.

### 2.2 Prioritise a label (optional)

Click the star next to `priority-high` in the label list to prioritise it.

### 2.3 Optional: create labels with the API from PowerShell

Create a personal access token with the `api` scope, find your project ID (**Settings > General**, shown under the project name), then:

```powershell
$token   = Read-Host "GitLab token" -AsSecureString
$plain   = [System.Net.NetworkCredential]::new("", $token).Password
$project = "<PROJECT_ID>"
$headers = @{ "PRIVATE-TOKEN" = $plain }

Invoke-RestMethod -Method Post -Headers $headers `
  -Uri "https://gitlab.com/api/v4/projects/$project/labels" `
  -Body @{ name = "type-feature"; color = "#428BCA" }
```

Note the backtick `` ` `` at the end of the first line: it is PowerShell's line-continuation character, and there must be nothing after it. Do not paste your token into files or commits.

---

## Part 3: Milestones (10 min)

1. **Plan > Milestones > New milestone**.
2. Create:
   - **Title:** `Sprint 1`, **Start date:** today, **Due date:** two weeks from today, **Description:** Calculator basic operations
3. Create `Sprint 2` starting after Sprint 1.
4. Open Sprint 1 and note the **Issues**, **Merge requests**, **Participants**, and **Labels** tabs and the progress bar that updates as issues close.

To calculate dates in PowerShell:

```powershell
(Get-Date).ToString("yyyy-MM-dd")
(Get-Date).AddDays(14).ToString("yyyy-MM-dd")
```

---

## Part 4: Create Issues (15 min)

Go to **Plan > Issues > New issue** and create four issues:

| # | Title | Labels | Milestone |
|---|---|---|---|
| 1 | Add subtract function | `type-feature`, `status-todo`, `weight-1`, `priority-high` | Sprint 1 |
| 2 | Add multiply function | `type-feature`, `status-todo`, `weight-3`, `priority-low` | Sprint 1 |
| 3 | Handle divide by zero | `type-bug`, `status-todo`, `weight-5`, `priority-high` | Sprint 1 |
| 4 | Add unit tests | `type-feature`, `status-todo`, `weight-3`, `priority-low` | Sprint 2 |

Use this description template on each:

```markdown
## Summary
What needs to be done and why.

## Acceptance criteria
- [ ] Function implemented
- [ ] Pipeline passes
- [ ] Reviewed and merged
```

Set **Assignee** to yourself and optionally a due date.

### Useful quick actions (type in a comment or description)

```
/label ~type-feature ~status-todo
/milestone %"Sprint 1"
/assign me
/due in 7 days
/relate #2
```

---

## Part 5: Issue Boards (10 min)

1. **Plan > Issue boards**.
2. From the board dropdown choose **Create new board**. Name: `Sprint 1 Board`.
3. Add label lists: **Add list > Label** for `status-todo`, `status-doing`, `status-review`, `status-done`.
4. Filter the board by Milestone = `Sprint 1`.
5. **Drag** issue 1 from *status-todo* to *status-doing*. Open the issue and confirm the label changed.
6. Close an issue and see it move to the **Closed** column, then reopen it.
7. Create an issue from the **+** icon in a list header.

**Checkpoint:** The board shows Sprint 1 issues across status lists.

---

## Part 6: Time Tracking and Weight (10 min)

### 6.1 Time tracking (Free)

Open issue #1 and add each of these as a comment:

```
/estimate 3h
/spend 1h
```

Log more time with a date (use today's date in `YYYY-MM-DD`):

```
/spend 45m 2026-09-28
```

Watch the **Time tracking** sidebar widget: estimate, spent, remaining. Reduce logged time with a negative value such as `/spend -15m`, and clear the estimate with `/remove_estimate`.

Units: `mo`, `w`, `d`, `h`, `m`. Open **Time tracking report** in the widget to see entries.

### 6.2 Weight estimation

- **Premium+:** the sidebar has a **Weight** field, or use `/weight 3`. Boards and milestones show total weight.
- **Free workaround (used here):** `weight-N` labels. Filter by label to sum estimates manually.

Discussion: what does one weight point mean for your team (story points, ideal days, relative size)?

---

## Part 7: Epics and Roadmaps, Overview (5 min)

> **Premium/Ultimate feature.** Not available on Free, so this section is conceptual.

**Epic:** a container for related issues across milestones or teams, for example "Calculator v1.0" containing issues #1 to #4. It can have start and due dates, labels, and a progress percentage, and can be nested in higher tiers.

**Roadmap:** a timeline view of epics by start and due date, giving stakeholders a quarter-level view.

**Where they live:** at group level, under **Plan > Epics** and **Plan > Roadmap**.

### Free-tier workaround

1. Create an issue titled `[Epic] Calculator v1.0` with label `type-epic`.
2. In its description add a checklist of children:

```markdown
- [ ] #1
- [ ] #2
- [ ] #3
- [ ] #4
```

3. Comment `/relate #1 #2 #3 #4` on the parent to add related links.
4. Use the **Milestones** list, sorted by due date, as a lightweight roadmap.

---

## Part 8: Hands-on Lab, Raise an Issue, Create a Branch, Open an MR (25 min)

We implement issue #1, "Add subtract function", end to end.

### 8.1 Raise (or reuse) the issue

Open issue #1. Confirm it has a description, labels, milestone `Sprint 1`, and an assignee. Move it to `status-doing` on the board, or comment:

```
/unlabel ~status-todo
/label ~status-doing
```

### 8.2 Create a branch and MR from the issue

On the issue page, open the **Create merge request** dropdown and choose **Create merge request and branch**.

- The suggested branch name looks like `1-add-subtract-function`
- Source branch: `main`
- Click **Create merge request**

GitLab creates the branch, opens a **Draft** MR, and links it to the issue. The MR description contains `Closes #1`.

> Alternative: choose **Create branch** only, then open the MR yourself after pushing.

### 8.3 Do the work locally in PowerShell

```powershell
git fetch origin
git checkout 1-add-subtract-function
```

Append the function:

```powershell
@'

def subtract(a, b):
    return a - b
'@ | Add-Content -Path calculator.py -Encoding ascii
```

Verify, commit, and push:

```powershell
Get-Content calculator.py
git add calculator.py
git commit -m "Add subtract function (#1)"
git push
```

Referencing `#1` in the commit message adds a cross-reference on the issue.

### 8.4 Prepare the MR

1. Open the MR (link from the issue, or **Code > Merge requests**).
2. Confirm the description has `Closes #1`. Other closing keywords: `Fixes #1`, `Resolves #1`.
3. Set **Milestone** = `Sprint 1`, **Labels** = `type-feature`, `status-review`.
4. Add a comment with quick actions:

```
/spend 2h
/label ~status-review
/unlabel ~status-doing
```

5. Click **Mark as ready** to remove Draft status.
6. Wait for the pipeline to turn green.

### 8.5 Verify the links

- Issue #1 shows the MR under **Related merge requests**.
- The MR shows the closing-issue link.
- The Sprint 1 milestone lists both the issue and the MR.

### 8.6 Review and merge

1. Add a review comment (self-review is fine): *"LGTM."*
2. Click **Merge** (tick **Delete source branch**).
3. Confirm:
   - Issue #1 is **Closed** automatically
   - The board shows it under **Closed**
   - Sprint 1 milestone progress increased
4. Add `/label ~status-done` on the issue if you use that list.

Clean up locally:

```powershell
git checkout main
git pull
git branch -d 1-add-subtract-function
git fetch --prune
```

### 8.7 Stretch tasks

- Repeat for issue #3, using `Related to #2` and `Closes #3` in the MR description. Observe that `Related to` does **not** close #2.
- Mention `#2` in an MR description without a closing keyword and confirm the issue stays open.
- Optional: list issues from PowerShell with the API:

```powershell
$token   = Read-Host "GitLab token" -AsSecureString
$plain   = [System.Net.NetworkCredential]::new("", $token).Password
$project = "<PROJECT_ID>"

Invoke-RestMethod -Headers @{ "PRIVATE-TOKEN" = $plain } `
  -Uri "https://gitlab.com/api/v4/projects/$project/issues?state=all" |
  Select-Object iid, title, state, milestone |
  Format-Table -AutoSize
```

---

## Part 9: Wrap-up and Reflection (5 min)

### Review questions

1. What is the difference between `Closes #1` and `Related to #1`?
2. When is an issue auto-closed: on MR creation or on merge into the default branch?
3. How do labels and milestones complement each other in planning?
4. What would an epic and roadmap give you that milestones do not?
5. Why is logging time with `/spend` valuable, and what are its limits?

---

## Lab Completion Checklist

- [ ] Git for Windows installed; identity and authentication configured
- [ ] Project created; CI green on `main`
- [ ] Labels created (type, priority, status, weight)
- [ ] Milestones `Sprint 1` and `Sprint 2` created
- [ ] Four issues created with labels, milestones, and assignees
- [ ] Sprint board configured; card moved by drag and drop
- [ ] Estimate and time spent logged with quick actions
- [ ] Epic/roadmap concept understood; workaround parent issue created
- [ ] Branch created from issue #1; MR opened with `Closes #1`
- [ ] MR merged; issue #1 auto-closed; milestone progress updated

---

## Troubleshooting (Windows-specific)

| Symptom | Likely cause / fix |
|---|---|
| `git` is not recognized | Reopen PowerShell after install, or `winget install --id Git.Git -e` |
| `The token '&&' is not a valid statement separator` | Windows PowerShell 5.1; use one command per line |
| Here-string error: `Unrecognized token` | The closing `'@` must be at column 1 on its own line |
| CI cannot parse `.gitlab-ci.yml` | Possible BOM; rewrite with `-Encoding ascii` |
| `LF will be replaced by CRLF` | Harmless; optionally `git config --global core.autocrlf true` |
| HTTPS asks for a password repeatedly | Use a personal access token; clear stale entries in Windows **Credential Manager** |
| `Permission denied (publickey)` | Add the public key in GitLab; start the agent with `Start-Service ssh-agent` (admin PowerShell) and run `ssh-add $env:USERPROFILE\.ssh\id_ed25519` |
| API call returns 401 | Token missing the `api` scope or pasted incorrectly |
| No **Create merge request** dropdown on the issue | Needs Developer+ role |
| `/weight` or scoped labels not accepted | Premium feature; use `weight-N` and plain labels |
| Epics menu missing | Premium/Ultimate and group-level only |
| Issue did not auto-close | The MR must merge into the **default branch**, with a closing keyword in the MR description or a commit message |
| `/spend` invalid | Use formats like `1h30m` or `2d` |
