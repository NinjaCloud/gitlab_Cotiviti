# Lab 3: Merge Requests & Code Review Workflow (Windows PowerShell)

**Platform:** GitLab.com Free tier (or self-managed Free/CE)
**Shell:** Windows PowerShell 5.1 or PowerShell 7+ on Windows 10/11
**Duration:** ~90 minutes
**Level:** Beginner to Intermediate
**Accounts:** Two GitLab accounts is ideal (Owner/Maintainer + Developer). With one account, follow the "Solo mode" notes.

---

## Learning Objectives

1. Create and manage a merge request (MR) from a feature branch
2. Use the review workflow: inline comments, threads, and suggestions
3. Approve a merge request and enforce merge controls on the Free tier
4. Understand which approval-rule features are Premium and what Free-tier settings partly replace them
5. Create and resolve a merge conflict from PowerShell and the web UI

---

## Free Tier Notes (Read First)

| Feature | Free tier? | What to do in this lab |
|---|---|---|
| Create / edit / merge MRs | Yes | Fully covered |
| Inline comments, threads, suggestions | Yes | Fully covered |
| Optional approvals (Approve button) | Yes | Part 4 |
| **Required approvals / approval rules / Code Owners approvals** | **No (Premium+)** | Free equivalents: protected branches, "all threads must be resolved", "pipelines must succeed" |
| Merge conflict resolution (web editor + CLI) | Yes | Part 6 |
| CI/CD compute minutes (GitLab.com Free) | Limited monthly quota | Small pipeline only |

> GitLab changes tier packaging from time to time. Confirm at https://about.gitlab.com/pricing/ and in the docs.

---

## PowerShell Tips Used Throughout

- Open **Windows Terminal** or **PowerShell** (not Command Prompt).
- Run commands **one line at a time** or as blocks. Avoid `&&` in Windows PowerShell 5.1 (it only works in PowerShell 7+); this lab uses separate lines.
- Multi-line files are written with **single-quoted here-strings** `@' ... '@`. The closing `'@` must be at the **start of a line** with nothing before it.
- Files are written with `-Encoding ascii` to avoid a byte-order mark (BOM) that can break YAML.
- Check your version: `$PSVersionTable.PSVersion`

---

## Prerequisites

1. **Git for Windows.** Check with:

```powershell
git --version
```

If missing:

```powershell
winget install --id Git.Git -e
```

Close and reopen PowerShell afterwards.

2. **Set your identity** (one time):

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

3. **Authentication to GitLab.** Choose one:

**Option A: HTTPS with a personal access token (simplest).** In GitLab: **User avatar > Preferences > Access tokens**, create a token with `read_repository` and `write_repository`. Git Credential Manager (bundled with Git for Windows) prompts you on first push; use the token as the password.

**Option B: SSH key.**

```powershell
ssh-keygen -t ed25519 -C "you@example.com"
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
```

Paste it at **Preferences > SSH Keys**, then test:

```powershell
ssh -T git@gitlab.com
```

4. *(Optional)* A second GitLab account added as **Developer** to act as reviewer.

The examples below use HTTPS URLs. If you use SSH, substitute `git@gitlab.com:<namespace>/<project>.git`.

---

## Part 1: Set Up the Lab Project (10 min)

1. In GitLab, select **New project > Create blank project**.
2. Name: `mr-review-lab`. Visibility: **Private**. Tick **Initialize repository with a README**.
3. Clone it into a working folder:

```powershell
mkdir $HOME\gitlab-labs -Force
cd $HOME\gitlab-labs
git clone https://gitlab.com/<your-namespace>/mr-review-lab.git
cd mr-review-lab
```

4. Create the sample app and CI file:

```powershell
@'
def greet(name):
    return "Hello, " + name

def add(a, b):
    return a + b

if __name__ == "__main__":
    print(greet("GitLab"))
'@ | Set-Content -Path app.py -Encoding ascii

@'
stages: [test]

syntax-check:
  stage: test
  image: python:3.12-slim
  script:
    - python -m py_compile app.py
'@ | Set-Content -Path .gitlab-ci.yml -Encoding ascii
```

5. Commit and push:

```powershell
git add .
git commit -m "Add sample app and CI"
git push origin main
```

> You may see `warning: LF will be replaced by CRLF`. This is normal on Windows and harmless for this lab.

6. In GitLab, open **Build > Pipelines** and confirm the pipeline passes.
7. *(Optional)* Invite the reviewer: **Manage > Members > Invite members**, role **Developer**.

**Checkpoint:** `main` contains `README.md`, `app.py`, `.gitlab-ci.yml`, and a green pipeline.

---

## Part 2: Create and Manage a Merge Request (15 min)

### 2.1 Create a feature branch and change code

```powershell
git checkout -b feature/add-farewell
```

Append a function to `app.py`:

```powershell
@'

def farewell(name):
    return "Goodbye, " + name
'@ | Add-Content -Path app.py -Encoding ascii
```

```powershell
git add app.py
git commit -m "Add farewell function"
git push -u origin feature/add-farewell
```

### 2.2 Open the merge request

GitLab prints a **Create merge request** link in the terminal after the push. Otherwise go to **Code > Merge requests > New merge request**.

Fill in:

- **Title:** `Draft: Add farewell function` (the `Draft:` prefix blocks merging)
- **Description:** what changed, why, and how to test
- **Assignee:** yourself
- **Reviewer:** your second account (or yourself in solo mode)
- **Squash commits when merge request is accepted:** tick
- **Delete source branch when merge request is accepted:** tick

Select **Create merge request**.

### 2.3 Manage the MR

Explore each tab and note what it shows:

- **Overview:** description, activity, pipeline status, merge box
- **Commits**, **Pipelines**, **Changes** (try inline and side-by-side diff)

Practice these actions:

1. Click **Mark as ready** to remove Draft status.
2. Edit the title and description with **Edit**.
3. Use quick actions in a comment:

```
/assign_reviewer @username
/label ~"needs-review"
```

4. Push another commit and watch the MR and pipeline update automatically:

```powershell
"# tweak" | Add-Content -Path app.py -Encoding ascii
git commit -am "Small tweak"
git push
```

**Checkpoint:** MR is open, not in Draft, pipeline running or green.

---

## Part 3: Code Review Workflow, Comments and Suggestions (20 min)

> **Solo mode:** GitLab lets you comment on your own MR, so you can play both author and reviewer.

### 3.1 Seed something worth reviewing

```powershell
@'

def multiply(a,b):
    result = a*b
    return result
'@ | Add-Content -Path app.py -Encoding ascii

git commit -am "Add multiply function"
git push
```

### 3.2 Reviewer: leave inline comments

1. Open the MR **Changes** tab.
2. Hover over a line number, click the **comment icon**, and write:
   *"Please add a docstring and consistent spacing after the comma."*
3. Add a general comment on the Overview tab: *"Overall direction looks good."*
4. Use **Start a review** to batch comments, add a second comment, then **Submit review**.

### 3.3 Reviewer: add a suggestion

1. On the `def multiply(a,b):` line, click the comment icon, then **Insert suggestion**.
2. Edit the suggested block to:

````markdown
```suggestion:-0+0
def multiply(a, b):
```
````

3. Submit the review.

### 3.4 Author: apply the suggestion and respond

1. Click **Apply suggestion** (or **Add suggestion to batch**, then **Apply suggestions**). GitLab commits it for you.
2. Bring that commit down locally, then fix the docstring:

```powershell
git pull
```

Open `app.py` in an editor (for example `notepad app.py` or `code app.py`), add a docstring under `def multiply(a, b):`, and save. Then:

```powershell
git add app.py
git commit -m "Add docstring to multiply"
git push
```

3. Reply "Done" on the thread.

### 3.5 Resolve threads

- Click **Resolve thread** on each addressed comment. The MR header shows *"X of Y threads resolved"*.
- Use **Changes > Compare** (version dropdown) to see what changed between pushes, and mark files as **Viewed**.

**Checkpoint:** At least one suggestion applied, all threads resolved.

---

## Part 4: Approvals and Merge Controls on Free Tier (15 min)

### 4.1 What is available on Free

- **Optional approvals:** an eligible user with Developer+ role (other than the author, by default) can click **Approve**. It is recorded but **does not block merging**.

### 4.2 What is Premium (know it, do not expect to configure it)

- **Approval rules** (for example, "2 approvals from group Security")
- **Required approvers** and **Code Owners as required approvers**
- Preventing author or committer approval, resetting approvals on new commits

These live at **Settings > Merge requests > Merge request approvals** on Premium/Ultimate.

### 4.3 Approve the MR

1. As the reviewer account, open the MR and click **Approve**.
2. Confirm the approval appears in the sidebar and activity feed.
3. Click **Revoke approval**, then approve again.

> Solo mode: you cannot approve your own MR by default. Skip this step and describe the expected behaviour.

### 4.4 Free-tier equivalents for a "required review" gate

1. **Protect `main`:** **Settings > Repository > Protected branches**
   - Allowed to merge: **Maintainers**
   - Allowed to push and merge: **No one**
2. **Merge checks:** **Settings > Merge requests > Merge checks**
   - Tick **Pipelines must succeed**
   - Tick **All threads must be resolved**
3. **Merge method:** choose **Merge commit** or **Fast-forward merge**; set squash to **Encourage**.

### 4.5 Verify the gate

1. Add a new comment thread on the MR and leave it unresolved.
2. Confirm the **Merge** button is disabled with an unresolved-discussions message.
3. Resolve it; with a green pipeline, the Merge button activates.

**Checkpoint:** You can explain what Free-tier settings enforce and what needs Premium.

---

## Part 5: Merge the MR (5 min)

1. Confirm pipeline green, threads resolved, one approval recorded.
2. Click **Merge**.
3. Verify locally:

```powershell
git checkout main
git pull
git log --oneline -5
```

---

## Part 6: Resolving Merge Conflicts (20 min)

We create two branches that edit the same line.

### 6.1 Create conflicting changes

```powershell
git checkout main
git pull

# Branch A
git checkout -b feature/greeting-a
(Get-Content app.py) -replace 'return "Hello, " \+ name', 'return "Hi there, " + name' | Set-Content app.py -Encoding ascii
git commit -am "Change greeting to Hi there"
git push -u origin feature/greeting-a

# Branch B (from main, not from A)
git checkout main
git checkout -b feature/greeting-b
(Get-Content app.py) -replace 'return "Hello, " \+ name', 'return "Welcome, " + name' | Set-Content app.py -Encoding ascii
git commit -am "Change greeting to Welcome"
git push -u origin feature/greeting-b
```

Confirm each branch changed the line:

```powershell
Select-String -Path app.py -Pattern 'return "(Hi there|Welcome|Hello)'
```

> `-replace` uses regex, so the `+` is escaped as `\+`.

### 6.2 Open both MRs and merge the first

1. Create an MR for `feature/greeting-a` into `main` and **merge it**.
2. Create an MR for `feature/greeting-b` into `main`.
3. Observe **"There are merge conflicts."** and a disabled Merge button.

### 6.3 Option 1: Resolve in the web UI

1. Click **Resolve conflicts**.
2. For the conflicting block choose **Use ours**, **Use theirs**, or **Edit inline** and set:

```python
    return "Welcome, " + name
```

3. Add a commit message and click **Commit to source branch**.
4. Confirm the conflict warning disappears.

### 6.4 Option 2: Resolve locally in PowerShell

If you resolved in the UI already, create a fresh conflict by repeating 6.1 with different greeting text (for example `"Greetings, "`), merging one MR, then continue here on the other branch.

```powershell
git checkout feature/greeting-b
git fetch origin
git merge origin/main
```

Git reports: `CONFLICT (content): Merge conflict in app.py`. Check status:

```powershell
git status
```

Open the file:

```powershell
code app.py     # or: notepad app.py
```

You will see markers:

```
<<<<<<< HEAD
    return "Welcome, " + name
=======
    return "Hi there, " + name
>>>>>>> origin/main
```

Edit to the final content, delete all three marker lines, save, then confirm none remain:

```powershell
Select-String -Path app.py -Pattern '^(<<<<<<<|=======|>>>>>>>)'
```

(No output means clean.) Then:

```powershell
git add app.py
git commit -m "Resolve merge conflict in greeting"
git push
```

Alternative with rebase:

```powershell
git rebase origin/main
# resolve, then:
git add app.py
git rebase --continue
git push --force-with-lease
```

To abort at any time: `git merge --abort` or `git rebase --abort`.

### 6.5 Finish

Merge the MR once the pipeline is green.

**Checkpoint:** Both branches merged; `main` shows the resolved greeting.

---

## Part 7: Cleanup and Reflection (5 min)

```powershell
git checkout main
git pull
git branch -d feature/add-farewell
git branch -d feature/greeting-a
git branch -d feature/greeting-b
git fetch --prune
```

### Review questions

1. What is the difference between a single comment and a review with batched comments?
2. When would you use a suggestion versus a normal comment?
3. Which approval features require Premium, and what Free-tier settings partly replace them?
4. Why can a conflict appear only after another MR has merged?
5. What are the pros and cons of resolving conflicts in the UI versus locally?

---

## Lab Completion Checklist

- [ ] Git for Windows installed; identity and authentication configured
- [ ] Project created with CI pipeline passing
- [ ] MR created and moved from Draft to ready
- [ ] Inline comment, batched review, and suggestion completed
- [ ] Suggestion applied; threads resolved
- [ ] MR approved (optional approval)
- [ ] `main` protected; merge checks enabled
- [ ] MR merged
- [ ] Merge conflict created and resolved
- [ ] Cleanup done

---

## Troubleshooting (Windows-specific)

| Symptom | Likely cause / fix |
|---|---|
| `git` is not recognized | Reopen PowerShell after install, or run `winget install --id Git.Git -e` |
| Script blocked / cannot run commands | Not relevant for this lab (no `.ps1` files); if you add scripts: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| `The token '&&' is not a valid statement separator` | You are on Windows PowerShell 5.1; put each command on its own line |
| Here-string error: `Unrecognized token` | The closing `'@` must start at column 1 on its own line |
| CI fails to parse `.gitlab-ci.yml` | File may contain a BOM; rewrite with `-Encoding ascii` |
| `LF will be replaced by CRLF` warning | Harmless; optionally `git config --global core.autocrlf true` |
| HTTPS push asks for password repeatedly | Use a personal access token as the password; clear old entries in **Credential Manager** (Windows) |
| `Permission denied (publickey)` | Key not added in GitLab, or run `Start-Service ssh-agent` (as admin: `Set-Service ssh-agent -StartupType Manual`) then `ssh-add $env:USERPROFILE\.ssh\id_ed25519` |
| Cannot approve own MR | Use the second account; skip Part 4.3 in solo mode |
| Merge button greyed out | Check Draft status, unresolved threads, failed pipeline, or conflicts |
| Push rejected to `main` | Branch is protected; open an MR instead |
