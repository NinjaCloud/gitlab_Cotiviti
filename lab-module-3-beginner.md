# Lab 3 (Beginner): Merge Requests & Code Review

**Tools:** GitLab.com Free account, Git for Windows, PowerShell
**Time:** ~60 minutes
**Work with a partner if you can.** One person is the *author*, the other is the *reviewer*. If you are alone, do both roles.

---

## What You Will Learn

- What a **branch** and a **merge request (MR)** are
- How to ask for a review, leave comments, and make suggestions
- How to approve and merge
- How to fix a **merge conflict**

## Words to Know

| Word | Simple meaning |
|---|---|
| Repository (repo) | A project folder tracked by Git |
| Branch | Your own copy of the work where you can make changes safely |
| Commit | A saved snapshot of your changes |
| Push | Send your commits to GitLab |
| Merge request (MR) | A request to add your branch's changes into `main` |
| Conflict | Two people changed the same line and Git cannot pick one |

> **Free tier note:** On GitLab Free you can **approve** an MR, but you cannot *require* approvals. That is a Premium feature. In this lab we use other settings to keep `main` safe.

---

## Step 0: Check Your Setup

Open **PowerShell** and run:

```powershell
git --version
```

You should see a version number. If not, install Git:

```powershell
winget install --id Git.Git -e
```

Close and reopen PowerShell, then tell Git who you are (one time only):

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

**Sign in to GitLab when Git asks.** If it asks for a password, use a *personal access token*: in GitLab go to **your avatar > Preferences > Access tokens**, create a token with `read_repository` and `write_repository`, and paste it as the password.

---

## Step 1: Create a Project

1. In GitLab click **New project > Create blank project**.
2. Name it `mr-lab`. Choose **Private**. Tick **Initialize repository with a README**.
3. Click **Create project**.
4. If you have a partner, add them: **Manage > Members > Invite members**, role **Developer**.

**You should see:** a project page with a `README.md` file.

---

## Step 2: Copy the Project to Your Computer (Clone)

On the project page click the blue **Code** button and copy the **HTTPS** address. Then:

```powershell
cd $HOME
git clone PASTE_THE_ADDRESS_HERE
cd mr-lab
```

**You should see:** a new folder `mr-lab` and PowerShell now inside it.

---

## Step 3: Make a Branch and a Change

Create a branch called `add-hello`:

```powershell
git checkout -b add-hello
```

Create a new file:

```powershell
"print('Hello, GitLab')" | Set-Content -Path hello.py -Encoding ascii
```

Save your change (commit) and send it to GitLab (push):

```powershell
git add hello.py
git commit -m "Add hello.py"
git push -u origin add-hello
```

**You should see:** a message in PowerShell with a link that says *"Create merge request"*.

---

## Step 4: Create the Merge Request

1. Click that link (or go to **Code > Merge requests > New merge request**).
2. **Source branch:** `add-hello`. **Target branch:** `main`.
3. **Title:** `Add hello.py`
4. **Description:** write one sentence about what you changed.
5. **Reviewer:** choose your partner (or yourself).
6. Tick **Delete source branch when merge request is accepted**.
7. Click **Create merge request**.

**You should see:** the MR page with tabs: Overview, Commits, Pipelines, Changes.

Click the **Changes** tab to see exactly what was added (green lines).

---

## Step 5: Review the Code (Reviewer's Job)

1. Open the MR, then the **Changes** tab.
2. Hover over a line number and click the small **speech bubble**.
3. Write a comment, for example: *"Please change the message to 'Hello, World'."*
4. Click **Start review** (not "Add comment now").
5. Click **Submit review** at the bottom of the page.

**You should see:** your comment as a **thread** on the MR.

### Try a Suggestion

1. Click the speech bubble on the `print` line again.
2. Click the **Insert suggestion** button (looks like a ± icon).
3. Replace the text inside the box with:

```
print('Hello, World')
```

4. Submit the review.

**You should see:** a suggestion box with an **Apply suggestion** button.

---

## Step 6: Respond to the Review (Author's Job)

1. On the suggestion, click **Apply suggestion**. GitLab makes a commit for you.
2. Click **Resolve thread** on each comment you have dealt with.

Get that new commit on your computer:

```powershell
git pull
Get-Content hello.py
```

**You should see:** the file now says `Hello, World`.

---

## Step 7: Approve and Merge

**Reviewer:** on the MR page click **Approve**.

> You usually cannot approve your own MR. If you are alone, skip this and just read the button.

**Author:** make sure the pipeline (if any) is green and all threads are resolved. Click **Merge**.

Update your computer:

```powershell
git checkout main
git pull
Get-ChildItem
```

**You should see:** `hello.py` is now in `main`.

---

## Step 8: Keep `main` Safe (Free Settings)

1. Go to **Settings > Repository > Protected branches**.
2. Next to `main`, set **Allowed to push and merge** to **No one**, and **Allowed to merge** to **Maintainers**.
3. Go to **Settings > Merge requests** and tick **All threads must be resolved**.
4. Click **Save changes**.

**Why?** Now nobody can push straight to `main`. Every change must go through a merge request, and an MR with open comments cannot be merged.

---

## Step 9: Make and Fix a Merge Conflict

We will make two branches that change the *same line*.

```powershell
git checkout main
git pull

git checkout -b greeting-a
"print('Hi there')" | Set-Content -Path hello.py -Encoding ascii
git commit -am "Greeting A"
git push -u origin greeting-a

git checkout main
git checkout -b greeting-b
"print('Welcome')" | Set-Content -Path hello.py -Encoding ascii
git commit -am "Greeting B"
git push -u origin greeting-b
```

In GitLab:

1. Create an MR for `greeting-a` into `main`. Click **Merge**.
2. Create an MR for `greeting-b` into `main`.

**You should see:** *"There are merge conflicts"* and a greyed-out Merge button. That is expected!

### Fix It in GitLab

1. Click **Resolve conflicts**.
2. Pick **Use ours** or **Use theirs** (or **Edit inline** to write your own line).
3. Click **Commit to source branch**.
4. Click **Merge**.

**You should see:** both MRs merged and the conflict message gone.

---

## Step 10: Tidy Up

```powershell
git checkout main
git pull
git branch -d add-hello greeting-a greeting-b
```

---

## Checklist

- [ ] Cloned the project
- [ ] Made a branch, committed, and pushed
- [ ] Created a merge request
- [ ] Left a comment and a suggestion
- [ ] Applied the suggestion and resolved threads
- [ ] Approved and merged
- [ ] Protected `main`
- [ ] Created and fixed a conflict

## Quick Questions

1. Why do we make a branch instead of changing `main` directly?
2. What is the difference between a comment and a suggestion?
3. Why did the conflict happen?
4. What can you *not* do on the Free tier? (Hint: required approvals)

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| `git` not recognized | Close and reopen PowerShell |
| Asked for password again and again | Use a personal access token as the password |
| Push rejected on `main` | `main` is protected; use a branch and MR |
| Merge button is grey | Check for Draft, unresolved threads, failing pipeline, or conflicts |
| Cannot approve | You cannot approve your own MR; ask your partner |
| Not sure where you are | Run `git status` and `git branch` |
