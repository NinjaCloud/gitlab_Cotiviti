# Lab 4 : Issues & Project Management

**Tools:** GitLab.com Free account, Git for Windows, PowerShell
**Time:** ~60 minutes
**Goal:** Plan work with issues, then complete one issue by making a branch and a merge request.

---

## What You Will Learn

- What an **issue** is and how to organise issues with **labels** and **milestones**
- How to use an **issue board**
- How to track time
- How to link an issue to a merge request so it closes by itself

## Words to Know

| Word | Simple meaning |
|---|---|
| Issue | A task, bug, or idea you want to track |
| Label | A coloured tag on an issue, for example `bug` |
| Milestone | A goal or time period, for example `Sprint 1` |
| Board | A Kanban-style view: cards move across columns |
| Merge request (MR) | A request to add your branch's changes into `main` |

## What Is Not on the Free Tier

| Feature | What we do instead |
|---|---|
| Epics and Roadmaps (Premium) | Just learn what they are (Step 8) |
| Issue weight (Premium) | Use labels like `weight-1`, `weight-3` |
| Scoped labels (Premium) | Use simple names like `status-doing` |

---

## Step 0: Check Your Setup

```powershell
git --version
```

If it fails, install Git and reopen PowerShell:

```powershell
winget install --id Git.Git -e
```

Tell Git who you are (one time only):

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

When Git asks for a password, use a **personal access token** (**avatar > Preferences > Access tokens**, scopes `read_repository` and `write_repository`).

---

## Step 1: Create a Project

1. **New project > Create blank project**.
2. Name: `issues-lab`. **Private**. Tick **Initialize repository with a README**.
3. Click **Create project**.

Clone it:

```powershell
cd $HOME
git clone PASTE_THE_HTTPS_ADDRESS_HERE
cd issues-lab
```

Add a starter file:

```powershell
"def add(a, b):`n    return a + b" | Set-Content -Path calculator.py -Encoding ascii
git add calculator.py
git commit -m "Add calculator"
git push origin main
```

**You should see:** `calculator.py` in your project on GitLab.

---

## Step 2: Create Labels

Go to **Manage > Labels > New label**. Create these five:

| Name | Colour |
|---|---|
| `bug` | Red |
| `feature` | Blue |
| `status-todo` | Grey |
| `status-doing` | Yellow |
| `status-done` | Green |

---

## Step 3: Create a Milestone

1. Go to **Plan > Milestones > New milestone**.
2. Title: `Sprint 1`.
3. Pick a start date (today) and a due date (two weeks later).
4. Click **Create milestone**.

---

## Step 4: Create Three Issues

Go to **Plan > Issues > New issue**. Create:

| Title | Labels | Milestone |
|---|---|---|
| Add subtract function | `feature`, `status-todo` | Sprint 1 |
| Add multiply function | `feature`, `status-todo` | Sprint 1 |
| Fix divide by zero | `bug`, `status-todo` | Sprint 1 |

For each issue:

- Write a short **description** (what and why)
- Set **Assignee** to yourself
- Click **Create issue**

**You should see:** three issues numbered #1, #2, #3.

> **Shortcut:** in an issue comment, you can type commands that start with `/`. For example `/label ~bug` adds the bug label.

---

## Step 5: Use the Issue Board

1. Go to **Plan > Issue boards**.
2. Click **Create list** and choose **Label**. Add `status-todo`, then `status-doing`, then `status-done`.
3. **Drag** the "Add subtract function" card from *status-todo* to *status-doing*.
4. Open that issue and look at the labels.

**You should see:** the label changed by itself when you moved the card.

---

## Step 6: Track Time

Open issue #1 and write these comments (one per comment):

```
/estimate 2h
```

```
/spend 30m
```

**You should see:** the **Time tracking** box on the right showing the estimate, the time spent, and a progress bar.

> Time units: `m` minutes, `h` hours, `d` days, `w` weeks.

---

## Step 7: Main Lab, Issue to Branch to Merge Request

We will finish issue #1: **Add subtract function**.

### 7.1 Create the branch and MR from the issue

1. Open issue #1.
2. Click the arrow next to **Create merge request** and choose **Create merge request and branch**.
3. Keep the suggested branch name (for example `1-add-subtract-function`) and click **Create merge request**.

**You should see:** a new MR marked **Draft**, whose description includes `Closes #1`.

### 7.2 Do the work on your computer

```powershell
git fetch origin
git checkout 1-add-subtract-function
```

Add the function:

```powershell
"`ndef subtract(a, b):`n    return a - b" | Add-Content -Path calculator.py -Encoding ascii
Get-Content calculator.py
```

Save and push:

```powershell
git add calculator.py
git commit -m "Add subtract function (#1)"
git push
```

### 7.3 Finish the MR

1. Open the MR page. Click **Mark as ready** (removes Draft).
2. Add a comment: `/label ~status-doing`
3. Check the **Changes** tab. Your new function should be there.

### 7.4 Merge

1. Click **Merge**.
2. Go back to issue #1.

**You should see:**
- Issue #1 is now **Closed**, automatically
- The MR is listed under **Related merge requests** on the issue
- The Sprint 1 milestone shows progress

Update your computer:

```powershell
git checkout main
git pull
```

---

## Step 8: Epics and Roadmaps (Just Read)

These are **Premium** features, so you cannot try them on Free.

- **Epic:** a big goal that groups many issues, for example "Calculator v1.0"
- **Roadmap:** a timeline that shows epics over months

**Free workaround:** make one issue called `[Epic] Calculator v1.0` and list the child issues inside it:

```
- [ ] #1
- [ ] #2
- [ ] #3
```

GitLab turns `#1` into a link, and you can tick the boxes as work finishes.

---

## Step 9: Practice on Your Own (Optional)

Repeat Step 7 for issue #2 (multiply function). This time, write `Closes #2` yourself in the MR description.

Then try `Related to #3` in the MR description. Does issue #3 close when you merge? (It should not. Only words like `Closes`, `Fixes`, and `Resolves` close an issue.)

---

## Checklist

- [ ] Created labels and a milestone
- [ ] Created three issues
- [ ] Moved a card on the board
- [ ] Logged an estimate and time spent
- [ ] Created a branch and MR from an issue
- [ ] Pushed a change and merged the MR
- [ ] Saw the issue close automatically

## Quick Questions

1. What is the difference between a label and a milestone?
2. What does `Closes #1` do?
3. Why link an MR to an issue?
4. Which two features in this module need Premium?

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| `git` not recognized | Close and reopen PowerShell |
| Asked for password repeatedly | Use a personal access token |
| No "Create merge request" button on the issue | You need the Developer role or higher |
| Issue did not close | The MR must be **merged into main**, and its description must contain `Closes #1` |
| `/estimate` did nothing | Put the command alone on its own line in a comment |
| Not sure where you are | Run `git status` and `git branch` |
