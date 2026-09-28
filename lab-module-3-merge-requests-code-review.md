# Lab 3: Merge Requests & Code Review Workflow

**Platform:** GitLab.com Free tier (or self-managed Free/CE)
**Duration:** ~90 minutes
**Level:** Beginner to Intermediate
**Roles needed:** Two GitLab accounts is ideal (Maintainer/Owner + Developer). If you only have one, follow the "Solo mode" notes.

---

## Learning Objectives

By the end of this lab you will be able to:

1. Create and manage a merge request (MR) from a feature branch
2. Use the code review workflow: inline comments, threads, and suggestions
3. Approve a merge request and enforce merge controls on the Free tier
4. Understand which approval-rule features are Premium and how to achieve similar control on Free
5. Create and resolve a merge conflict

---

## Free Tier Notes (Read First)

| Feature | Free tier? | What to do in this lab |
|---|---|---|
| Create / edit / merge MRs | Yes | Fully covered |
| Inline comments, threads, suggestions | Yes | Fully covered |
| Optional approvals (Approve button) | Yes | Covered in Part 4 |
| **Required approvals / approval rules / Code Owners approvals** | **No (Premium+)** | Demonstrated through Free-tier equivalents: protected branches, "all threads must be resolved", "pipelines must succeed" |
| Merge conflict resolution (web editor + CLI) | Yes | Fully covered |
| 400 CI/CD compute minutes/month (GitLab.com Free) | Yes | Used in a small pipeline |

> GitLab changes tier packaging from time to time. Confirm current limits at https://about.gitlab.com/pricing/ and in the docs under **Merge requests > Approvals**.

---

## Prerequisites

- A GitLab.com account (Free) and Git installed locally (`git --version`)
- SSH key or personal access token configured for pushing
- (Optional) A second GitLab account to act as reviewer, added to your project as **Developer**

---

## Part 1: Set Up the Lab Project (10 min)

1. In GitLab, select **New project > Create blank project**.
2. Name: `mr-review-lab`. Visibility: **Private**. Tick **Initialize repository with a README**.
3. Clone it locally:

```bash
git clone git@gitlab.com:<your-namespace>/mr-review-lab.git
cd mr-review-lab
```

4. Add a small app file and a CI file, then push to `main`:

```bash
cat > app.py <<'EOF'
def greet(name):
    return "Hello, " + name

def add(a, b):
    return a + b

if __name__ == "__main__":
    print(greet("GitLab"))
EOF

cat > .gitlab-ci.yml <<'EOF'
stages: [test]

syntax-check:
  stage: test
  image: python:3.12-slim
  script:
    - python -m py_compile app.py
EOF

git add .
git commit -m "Add sample app and CI"
git push origin main
```

5. Go to **Build > Pipelines** and confirm the pipeline passes.

6. *(Optional)* Invite the reviewer: **Manage > Members > Invite members**, role **Developer**.

**Checkpoint:** `main` contains `README.md`, `app.py`, `.gitlab-ci.yml`, and a green pipeline.

---

## Part 2: Create and Manage a Merge Request (15 min)

### 2.1 Create a feature branch and change code

```bash
git checkout -b feature/add-farewell
```

Edit `app.py` to add a function:

```python
def farewell(name):
    return "Goodbye, " + name
```

```bash
git add app.py
git commit -m "Add farewell function"
git push -u origin feature/add-farewell
```

### 2.2 Open the merge request

GitLab prints a link in the terminal after the push. Otherwise: **Code > Merge requests > New merge request**.

Fill in:

- **Title:** `Draft: Add farewell function` (the `Draft:` prefix blocks merging)
- **Description:** what changed, why, and how to test
- **Assignee:** yourself
- **Reviewer:** your second account (or yourself in solo mode)
- **Labels:** optional
- **Squash commits when merge request is accepted:** tick
- **Delete source branch when merge request is accepted:** tick

Select **Create merge request**.

### 2.3 Manage the MR

Explore each tab and note what it shows:

- **Overview:** description, activity, pipeline status, merge box
- **Commits:** the commits on the branch
- **Pipelines:** CI runs for this branch
- **Changes:** the diff (try inline vs side-by-side view)

Practice these management actions:

1. Click **Mark as ready** to remove Draft status.
2. Change the title and description via **Edit**.
3. Use quick actions in a comment: `/assign_reviewer @username`, `/label ~"needs-review"`, `/target_branch main`.
4. Push one more commit to the branch and watch the MR and pipeline update automatically.

**Checkpoint:** MR is open, not in Draft, pipeline is running or green.

---

## Part 3: Code Review Workflow, Comments, Suggestions (20 min)

> **Solo mode:** GitLab lets you comment on your own MR, so you can play both author and reviewer. Switch mental hats between steps.

### 3.1 Seed something worth reviewing

On your feature branch, add a deliberately imperfect function to `app.py`:

```python
def multiply(a,b):
    result = a*b
    return result
```

Commit and push.

### 3.2 Reviewer: leave inline comments

1. Open the MR **Changes** tab.
2. Hover over a line number and click the **comment icon**. Select a line and add:
   *"Please add a docstring and consistent spacing after the comma."*
3. Leave a **general comment** on the Overview tab: *"Overall direction looks good."*
4. Choose **Start a review** (instead of a single comment) so comments are batched, then add a second comment. Finish with **Submit review**.

### 3.3 Reviewer: add a suggestion

1. On the `def multiply(a,b):` line, click the comment icon, then the **Insert suggestion** button (the `±` icon).
2. Edit the suggested block:

````markdown
```suggestion:-0+0
def multiply(a, b):
```
````

3. Submit the review.

### 3.4 Author: respond and apply

1. On the suggestion, click **Apply suggestion** (or **Add suggestion to batch** for several, then **Apply suggestions**). GitLab creates a commit for you.
2. Reply to the docstring thread, fix it locally, push, and reply "Done".

```bash
git pull   # bring down the commit created by "Apply suggestion"
# add docstring in app.py
git add app.py
git commit -m "Add docstring to multiply"
git push
```

### 3.5 Resolve threads

- Click **Resolve thread** on each addressed comment.
- The MR header shows *"X of Y threads resolved"*.

### 3.6 Review history

- Use **Changes > Compare** (the version dropdown) to view what changed between pushes.
- Mark files as **Viewed** to track your review progress.

**Checkpoint:** At least one suggestion applied, all threads resolved.

---

## Part 4: Approvals and Merge Controls on Free Tier (15 min)

### 4.1 What is available on Free

- **Optional approvals:** any eligible user with Developer+ role (other than the author, by default) can click **Approve** on an MR. Approval is recorded but **does not block merging**.

### 4.2 What is Premium (know it, do not expect to configure it)

- **Approval rules** (e.g., "2 approvals from group Security")
- **Required approvers** and **Code Owners as required approvers**
- Preventing author or committer approval, resetting approvals on new commits

You would find these at **Settings > Merge requests > Merge request approvals** on Premium/Ultimate.

### 4.3 Approve the MR

1. As the reviewer account, open the MR and click **Approve** in the merge box.
2. Confirm that the approval appears in the sidebar and activity feed.
3. Click **Revoke approval** and approve again to see both states.

### 4.4 Free-tier equivalents for "required review" control

Configure these to approximate an approval gate:

1. **Protect `main`:** **Settings > Repository > Protected branches**.
   - Allowed to merge: **Maintainers**
   - Allowed to push and merge: **No one**
   - Result: developers must go through an MR, and only Maintainers can merge.
2. **Merge checks:** **Settings > Merge requests > Merge checks**.
   - Tick **Pipelines must succeed**
   - Tick **All threads must be resolved**
3. **Merge method:** choose **Merge commit** or **Fast-forward merge**, and tick **Squash commits** = *Encourage*.

### 4.5 Verify the gate

1. Create a new comment thread on the MR and leave it unresolved.
2. Observe that the **Merge** button is disabled with *"Unresolved discussions must be resolved"*.
3. Resolve the thread, confirm the pipeline is green, and the Merge button activates.

**Checkpoint:** You can explain what is enforced by Free-tier settings and what needs Premium.

---

## Part 5: Merge the MR (5 min)

1. Confirm pipeline green, threads resolved, one approval recorded.
2. Click **Merge**.
3. Verify: MR state is **Merged**, the source branch is deleted, `main` has the new commit.

```bash
git checkout main
git pull
git log --oneline -5
```

---

## Part 6: Resolving Merge Conflicts (20 min)

We will deliberately create two branches that edit the same line.

### 6.1 Create conflicting changes

```bash
git checkout main && git pull

# Branch A
git checkout -b feature/greeting-a
sed -i 's/return "Hello, " + name/return "Hi there, " + name/' app.py
git commit -am "Change greeting to Hi there"
git push -u origin feature/greeting-a

# Branch B (from main, not from A)
git checkout main
git checkout -b feature/greeting-b
sed -i 's/return "Hello, " + name/return "Welcome, " + name/' app.py
git commit -am "Change greeting to Welcome"
git push -u origin feature/greeting-b
```

> On macOS use `sed -i '' 's/.../.../' app.py`, or just edit the file by hand.

### 6.2 Open both MRs and merge the first

1. Create an MR for `feature/greeting-a` into `main` and **merge it**.
2. Create an MR for `feature/greeting-b` into `main`.
3. Observe the message: **"There are merge conflicts."** The Merge button is disabled.

### 6.3 Option 1: Resolve in the web UI

1. Click **Resolve conflicts** in the MR.
2. For the conflicting block, choose **Use ours**, **Use theirs**, or **Edit inline** and write:

```python
    return "Welcome, " + name
```

3. Add a commit message and click **Commit to source branch**.
4. Confirm the conflict warning disappears.

### 6.4 Option 2: Resolve locally (repeat by creating a new conflict if desired)

```bash
git checkout feature/greeting-b
git fetch origin
git merge origin/main
# Git reports: CONFLICT (content): Merge conflict in app.py
```

Open `app.py` and find the markers:

```
<<<<<<< HEAD
    return "Welcome, " + name
=======
    return "Hi there, " + name
>>>>>>> origin/main
```

Edit to the final desired content, remove all markers, then:

```bash
git add app.py
git commit -m "Resolve merge conflict in greeting"
git push
```

Alternative: `git rebase origin/main`, resolve, `git add`, `git rebase --continue`, then `git push --force-with-lease`.

### 6.5 Finish

Merge the MR once the pipeline is green.

**Checkpoint:** Both branches are merged, `main` shows the resolved greeting.

---

## Part 7: Cleanup and Reflection (5 min)

```bash
git checkout main && git pull
git branch -d feature/add-farewell feature/greeting-a feature/greeting-b
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

- [ ] Project created with CI pipeline passing
- [ ] MR created, moved from Draft to ready
- [ ] Inline comment, batched review, and suggestion completed
- [ ] Suggestion applied; threads resolved
- [ ] MR approved (optional approval)
- [ ] `main` protected; merge checks enabled
- [ ] MR merged
- [ ] Merge conflict created and resolved
- [ ] Cleanup done

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Cannot approve own MR | Use the second account; in solo mode, skip Part 4.3 and describe the expected behaviour |
| Pipeline stuck "pending" | Shared runners may need account verification on GitLab.com; check **Settings > CI/CD > Runners** |
| Merge button greyed out | Check Draft status, unresolved threads, failed pipeline, or conflicts |
| "Resolve conflicts" button missing | Conflict is too complex for the UI; resolve locally |
| Push rejected to `main` | Branch is protected; open an MR instead |
