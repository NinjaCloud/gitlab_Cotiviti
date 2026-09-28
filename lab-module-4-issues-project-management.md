# Lab 4: Issues & Project Management

**Platform:** GitLab.com Free tier (or self-managed Free/CE)
**Duration:** ~90 minutes
**Level:** Beginner to Intermediate

---

## Learning Objectives

By the end of this lab you will be able to:

1. Track work with issues, labels, milestones, and issue boards
2. Explain epics and roadmaps and what they need (Premium) plus a Free-tier workaround
3. Link issues to merge requests so they close automatically
4. Use time tracking and estimate weight
5. Complete the full flow: **raise an issue, create a branch, open a merge request against it**

---

## Free Tier Notes (Read First)

| Feature | Free tier? | Approach in this lab |
|---|---|---|
| Issues, comments, assignees, due dates | Yes | Fully covered |
| Project labels, group labels | Yes | Fully covered |
| Scoped labels (`key::value`, mutually exclusive) | **Premium+** | Use plain labels with a naming convention |
| Milestones (project and group) | Yes | Fully covered |
| Issue boards | Yes (project-level board; multiple boards per project) | Fully covered |
| Group issue boards, assignee/milestone lists on boards | Partly restricted; check current docs | Label-based lists used |
| **Epics and Roadmaps** | **Premium+** | Overview only, with Free workaround using a parent issue and milestones |
| Issue weight | **Premium+** | Practice the `/weight` quick action if available; otherwise use weight labels |
| Time tracking (`/estimate`, `/spend`) | Yes | Fully covered |
| Issue-MR linking and auto-close | Yes | Fully covered |

> GitLab tier packaging changes. Confirm at https://about.gitlab.com/pricing/ and in the docs before teaching.

---

## Prerequisites

- GitLab.com account (Free) and Git installed locally
- SSH key or personal access token configured
- Optionally, a second user added as Developer (to practice assignment)

---

## Part 1: Set Up the Project (10 min)

1. **New project > Create blank project**. Name: `pm-lab`. Private. Initialize with README.
2. Clone it:

```bash
git clone git@gitlab.com:<your-namespace>/pm-lab.git
cd pm-lab
```

3. Add starter files and push to `main`:

```bash
cat > calculator.py <<'EOF'
def add(a, b):
    return a + b
EOF

cat > .gitlab-ci.yml <<'EOF'
stages: [test]

syntax-check:
  stage: test
  image: python:3.12-slim
  script:
    - python -m py_compile calculator.py
EOF

git add .
git commit -m "Initial calculator app"
git push origin main
```

**Checkpoint:** `main` has README, `calculator.py`, CI file.

---

## Part 2: Labels (10 min)

### 2.1 Create labels

Go to **Manage > Labels > New label**. Create:

| Name | Colour | Purpose |
|---|---|---|
| `type-feature` | Blue | New functionality |
| `type-bug` | Red | Defects |
| `priority-high` | Orange | Urgent work |
| `priority-low` | Grey | Nice to have |
| `status-todo` | Light blue | Not started |
| `status-doing` | Yellow | In progress |
| `status-review` | Purple | Awaiting review |
| `status-done` | Green | Complete |
| `weight-1`, `weight-3`, `weight-5` | Teal | Weight estimate (Free workaround) |

> **Premium note:** With scoped labels you would name these `status::todo`, `status::doing`, and GitLab would allow only one `status::` label at a time. On Free, you enforce that yourself by removing the old label when adding the new one.

### 2.2 Prioritise a label (optional)

In the label list, click the star icon next to `priority-high` to prioritise it. Prioritised labels sort issues on lists.

---

## Part 3: Milestones (10 min)

1. Go to **Plan > Milestones > New milestone**.
2. Create:
   - **Title:** `Sprint 1`
   - **Start date:** today
   - **Due date:** two weeks from today
   - **Description:** Calculator basic operations
3. Create a second milestone `Sprint 2` starting after Sprint 1.
4. Open Sprint 1 and note the tabs: **Issues**, **Merge requests**, **Participants**, **Labels**, and the progress bar (it updates as issues close).

---

## Part 4: Create Issues (15 min)

Go to **Plan > Issues > New issue** and create these four issues:

| # | Title | Type | Labels | Milestone | Priority |
|---|---|---|---|---|---|
| 1 | Add subtract function | Issue | `type-feature`, `status-todo`, `weight-1` | Sprint 1 | `priority-high` |
| 2 | Add multiply function | Issue | `type-feature`, `status-todo`, `weight-3` | Sprint 1 | `priority-low` |
| 3 | Handle divide by zero | Issue | `type-bug`, `status-todo`, `weight-5` | Sprint 1 | `priority-high` |
| 4 | Add unit tests | Issue | `type-feature`, `status-todo`, `weight-3` | Sprint 2 | `priority-low` |

For each issue add:

- **Description** with a template:

```markdown
## Summary
What needs to be done and why.

## Acceptance criteria
- [ ] Function implemented
- [ ] Pipeline passes
- [ ] Reviewed and merged
```

- **Assignee:** yourself
- **Due date:** optional

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

1. Go to **Plan > Issue boards**.
2. Click the board dropdown and select **Create new board**. Name it `Sprint 1 Board`.
3. Add label lists: **Add list > Label > `status-todo`**, then `status-doing`, `status-review`, `status-done`.
4. Use the search bar to filter by Milestone = `Sprint 1`.
5. **Drag** issue 1 from *status-todo* to *status-doing*. Open the issue and confirm the label changed automatically.
6. Try **Open** and **Closed** columns: closing an issue moves it to Closed.
7. Create an issue directly from a list using the **+** icon in a column header.

**Checkpoint:** Board shows your Sprint 1 issues distributed across status lists.

---

## Part 6: Time Tracking and Weight (10 min)

### 6.1 Time tracking (Free)

Open issue #1 and add these as comments (each on its own line in a comment):

```
/estimate 3h
/spend 1h
```

Then log more time with a date:

```
/spend 45m 2026-09-28
```

Observe the sidebar **Time tracking** widget: estimate, spent, and remaining bar. Remove time with `/remove_time_estimate` or a negative `/spend -15m`.

Useful units: `mo`, `w`, `d`, `h`, `m`.

View reports: open the time tracking widget and click **Time tracking report** to see entries.

### 6.2 Weight estimation

- **Premium+:** the sidebar has a **Weight** field, or use `/weight 3`. The board and milestone show total weight.
- **Free workaround (used in this lab):** the `weight-N` labels created in Part 2. Filter issues by label to sum estimates manually.

Question to discuss: what does one weight point represent for your team (story points, ideal days, relative size)?

---

## Part 7: Epics and Roadmaps, Overview (5 min)

> **Premium/Ultimate feature.** Not available on Free, so this section is conceptual.

**Epic:** a container for related issues that spans milestones or teams, for example "Calculator v1.0" containing issues #1 to #4. Epics can be nested (multi-level epics in Premium/Ultimate), have start and due dates, labels, and a progress percentage.

**Roadmap:** a timeline view of epics (and, in newer versions, milestones) by start and due date, so stakeholders can see what is planned across quarters.

**Where they live:** at group level, under **Plan > Epics** and **Plan > Roadmap**.

### Free-tier workaround

1. Create a parent issue titled `[Epic] Calculator v1.0` with label `type-epic`.
2. In its description, add a checklist referencing child issues:

```markdown
- [ ] #1
- [ ] #2
- [ ] #3
- [ ] #4
```

3. Link children with `/relate #1 #2 #3 #4` (related links) on the parent.
4. Use **Milestones** (with start and due dates) as the roadmap: the milestone list sorted by due date is a lightweight timeline.

---

## Part 8: Hands-on Lab, Raise an Issue, Create a Branch, Open an MR (25 min)

We will implement issue #1, "Add subtract function", end to end.

### 8.1 Raise (or reuse) the issue

Open issue #1. Confirm it has description, labels, milestone `Sprint 1`, and an assignee. Move it to `status-doing` on the board (or run `/unlabel ~status-todo` then `/label ~status-doing`).

### 8.2 Create a branch from the issue

On the issue page, use the **Create merge request** dropdown, then choose **Create merge request and branch**.

- Branch name will be suggested like `1-add-subtract-function`
- Source branch: `main`
- Click **Create merge request**

GitLab creates the branch, opens a **Draft** MR, and links it to the issue. The MR description already includes `Closes #1`.

> Alternative: choose **Create branch** only, and open the MR yourself after pushing.

### 8.3 Do the work locally

```bash
git fetch origin
git checkout 1-add-subtract-function
```

Edit `calculator.py`:

```python
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b
```

```bash
git add calculator.py
git commit -m "Add subtract function (#1)"
git push
```

Referencing `#1` in the commit message creates a cross-reference on the issue.

### 8.4 Prepare the MR

1. Open the MR from the issue link, or **Code > Merge requests**.
2. Confirm the description contains `Closes #1`. If not, add it. Other keywords that auto-close: `Fixes #1`, `Resolves #1`.
3. Set **Milestone** = `Sprint 1` and **Labels** = `type-feature`, `status-review`.
4. Use quick actions in a comment:

```
/spend 2h
/label ~status-review
/unlabel ~status-doing
```

5. Click **Mark as ready** to remove Draft status.
6. Watch the pipeline finish green.

### 8.5 Verify the links

- On issue #1, scroll to **Related merge requests**. The MR is listed.
- On the MR, the **Closes #1** link appears in the header area.
- The Milestone page lists both the issue and the MR.

### 8.6 Review and merge

1. Add a review comment (self-review is fine): *"LGTM."*
2. Click **Merge**. Tick **Delete source branch** and **Squash commits** if desired.
3. Confirm:
   - Issue #1 is now **Closed** automatically
   - The board shows it in **Closed**
   - The Sprint 1 milestone progress increased
4. Add `~status-done` label if you use it (`/label ~status-done`).

```bash
git checkout main
git pull
git branch -d 1-add-subtract-function
```

### 8.7 Stretch tasks

- Repeat the flow for issue #3 and this time link with `Related to #2` (which does **not** close the issue) and `Closes #3` in the description.
- Manually link an existing issue to an MR by mentioning `#2` in the MR description without a closing keyword; observe that it stays open.
- Enable **Settings > General > Merge requests > Issues: auto-close** behaviour and observe (default is on).

---

## Part 9: Wrap-up and Reflection (5 min)

### Review questions

1. What is the difference between `Closes #1` and `Related to #1`?
2. When are issues auto-closed: on MR creation, or on merge into the default branch?
3. How do labels and milestones complement each other in planning?
4. What data would an epic and roadmap give you that milestones do not?
5. Why is logging time with `/spend` valuable, and what are its limits?

### Optional: cleanup

```bash
git fetch --prune
```

---

## Lab Completion Checklist

- [ ] Project created; CI green on `main`
- [ ] Labels created (type, priority, status, weight)
- [ ] Milestones `Sprint 1` and `Sprint 2` created
- [ ] Four issues created with labels, milestones, and assignees
- [ ] Sprint board configured with status lists; card moved by drag and drop
- [ ] Estimate and time spent logged with quick actions
- [ ] Epic/roadmap concept understood; workaround parent issue created
- [ ] Branch created from issue #1, MR opened with `Closes #1`
- [ ] MR merged; issue #1 auto-closed; milestone progress updated

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| No **Create merge request** dropdown on the issue | You need Developer+ role; confirm project membership |
| `/weight` or scoped labels not accepted | Premium feature; use `weight-N` labels and plain labels |
| Epics menu missing | Premium/Ultimate and group-level only |
| Issue did not auto-close | MR must merge into the **default branch**, and the closing keyword must be in the MR description or a commit message |
| `/spend` says invalid | Use formats like `1h30m`, `2d`; separate units without spaces |
| Board list missing | Add via **Add list > Label**; the label must exist first |
