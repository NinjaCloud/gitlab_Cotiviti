# Lab 9 (Beginner, GUI-only): Automated Testing & Security Scanning

**Tools:** A web browser and a GitLab.com Free account. **No PowerShell, no Git commands, no downloads.**
**Time:** ~75 minutes
**Goal:** Run automated tests in a pipeline, see test reports and code coverage, and run GitLab's built-in security scanners.

---

## What You Will Learn

- How to run automated tests inside a pipeline
- How to show **test reports** and **code coverage** in GitLab
- How to turn on **SAST** and **Secret Detection** (both work on Free)
- What **dependency scanning**, **DAST**, and **container scanning** are (overview)

## Words to Know

| Word | Simple meaning |
|---|---|
| Unit test | A small automatic check that one function works correctly |
| Test report | A file (JUnit XML) that GitLab reads to show passed/failed tests |
| Code coverage | The percentage of your code that your tests actually run |
| SAST | Static Application Security Testing: scans your **source code** for risky patterns without running it |
| Secret Detection | Looks for passwords, tokens, and keys accidentally saved in the repo |
| Dependency scanning | Checks the libraries your project uses for known vulnerabilities |
| DAST | Dynamic scanning: attacks your **running** app from outside to find weaknesses |
| Container scanning | Checks a Docker image for known vulnerabilities |

---

## Free Tier: What Works and What Does Not

| Feature | On GitLab Free? | In this lab |
|---|---|---|
| Running tests, JUnit test reports | Yes | Hands-on |
| Code coverage (percentage, MR line coverage) | Yes | Hands-on |
| SAST and Secret Detection (scan runs, JSON report as a downloadable file) | Yes | Hands-on |
| Security dashboard, Vulnerability Report, findings shown inside the MR | **No (Ultimate)** | Read only |
| Dependency scanning, DAST, Container scanning (full results in the UI) | **No (Ultimate)** | Overview only |

> On Free, security scanners still run and save a **report file** you can download from the job. The nice dashboards need Ultimate. GitLab changes tier packaging from time to time, so check https://about.gitlab.com/pricing/ if something looks different.

---

## Step 0: Two Tools You Will Use All Lab

1. **Pipeline Editor** (**Build > Pipeline editor**): edit `.gitlab-ci.yml` with a **Validate** button.
2. **New file / Edit** on the repository page (**Code > Repository**): create or change any other file. After every edit, write a short **commit message** and click **Commit changes**. This is the GUI version of `git commit` and `git push`.

---

## Step 1: Create the Project and Some Code

1. **New project > Create blank project**. Name: `testing-security-lab`. **Private**. Tick **Initialize repository with a README**. Click **Create project**.
2. On the project page click **+ > New file**. File name: `app.py`. Paste:

```python
import hashlib
import subprocess


def add(a, b):
    return a + b


def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b


def is_even(n):
    return n % 2 == 0


def run_command(user_input):
    # Insecure on purpose: used later to show what SAST finds
    return subprocess.call(user_input, shell=True)


def hash_password(password):
    # Weak hash on purpose: used later to show what SAST finds
    return hashlib.md5(password.encode()).hexdigest()
```

3. Commit with message `Add app.py`.
4. Create another new file `test_app.py`:

```python
import pytest
from app import add, divide, is_even


def test_add():
    assert add(2, 3) == 5


def test_divide():
    assert divide(10, 2) == 5


def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(1, 0)


def test_is_even():
    assert is_even(4) is True
```

5. Commit with message `Add tests`.

**You should see:** `app.py` and `test_app.py` listed in your repository. Notice the tests do **not** cover `run_command` or `hash_password`. That is why coverage will be below 100%.

---

## Step 2: Run Automated Tests in a Pipeline

1. Go to **Build > Pipeline editor**. Replace the content with:

```yaml
stages:
  - test

unit-tests:
  stage: test
  image: python:3.12-slim
  script:
    - pip install pytest pytest-cov
    - pytest --junitxml=report.xml --cov=app --cov-report=term --cov-report=xml:coverage.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

2. Click **Validate**. It should say the syntax is valid.
3. Commit with message `Add test job` to `main`.
4. Go to **Build > Pipelines** and open the newest pipeline. Click the `unit-tests` job.

**You should see:** the job log shows 4 passed tests and a coverage table ending with a `TOTAL` row and a percentage.

> **What each part does:** `pytest` runs the tests. `--junitxml` saves a test report file. `--cov-report=xml` saves a coverage file. The `coverage:` line is a pattern that tells GitLab where to find the coverage percentage in the log. `artifacts: reports:` hands both files to GitLab so it can display them.

---

## Step 3: See the Test Report

1. Open the pipeline page (**Build > Pipelines**, click the pipeline).
2. Click the **Tests** tab.
3. Click the `unit-tests` suite.

**You should see:** a list of the 4 tests, each with a green check and how long it took.

### Make a test fail on purpose

1. Edit `test_app.py` (open the file, click **Edit**). Change `assert add(2, 3) == 5` to `assert add(2, 3) == 6`.
2. Commit with message `Break a test on purpose`.
3. Open the new pipeline and its **Tests** tab.

**You should see:** the pipeline is **failed**, and the Tests tab shows `test_add` in red with the assertion error. This is how reviewers find out *which* test broke without reading the whole log.

4. Edit the file again, change `6` back to `5`, and commit with message `Fix test`.

---

## Step 4: Code Coverage

### 4.1 See the percentage

Open **Build > Pipelines**. The pipeline row shows a coverage percentage (for example `70%`) next to the run. Click the `unit-tests` job and find the same number in the log.

### 4.2 See coverage inside a merge request

1. Go to **Code > Branches > New branch**. Name: `add-coverage-check`, based on `main`.
2. Open `app.py` while on that branch (use the branch selector at the top left). Click **Edit** and add this at the very bottom:

```python


def is_odd(n):
    return n % 2 != 0
```

3. Commit with message `Add is_odd` to branch `add-coverage-check`.
4. Click **Create merge request**, then **Create merge request** again on the next page.
5. Wait for the MR pipeline to finish. In the MR, click the **Changes** tab and look at `app.py`.

**You should see:** next to the line numbers, a **red** mark on the new, untested `is_odd` lines and a **green** mark on lines that tests ran. That is **coverage visualisation**: reviewers can spot untested new code before merging.

The MR overview page also shows a **Test summary** widget and a coverage figure.

### 4.3 Fix the red lines (optional)

Add a test for `is_odd` to `test_app.py` on the same branch (import it on the first line: `from app import add, divide, is_even, is_odd`, then add `def test_is_odd(): assert is_odd(3) is True`). Commit, and watch the lines turn green.

### 4.4 Coverage badge (optional)

1. Go to **Settings > General > Badges** (expand it).
2. Click **Add badge**. Name: `coverage`.
3. **Link:** `https://gitlab.com/%{project_path}/-/commits/%{default_branch}`
4. **Badge image URL:** `https://gitlab.com/%{project_path}/badges/%{default_branch}/coverage.svg`
5. Click **Add badge**, then look at your project's main page.

**You should see:** a small coverage badge on the project overview, the kind you often see on open-source projects.

---

## Step 5: Security Scanning With SAST and Secret Detection

### 5.1 Turn the scanners on

Merge or switch back to `main` first (you can merge the MR from Step 4). Then edit `.gitlab-ci.yml` in the **Pipeline editor** so it looks like this:

```yaml
stages:
  - test

include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

unit-tests:
  stage: test
  image: python:3.12-slim
  script:
    - pip install pytest pytest-cov
    - pytest --junitxml=report.xml --cov=app --cov-report=term --cov-report=xml:coverage.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

> **Shortcut:** you can also use **Secure > Security configuration** and click **Configure with a merge request** next to SAST. That makes the same change for you.

Commit with message `Enable SAST and Secret Detection` to `main`.

### 5.2 Add a fake secret to find

Create a new file `config.py` with this content:

```python
# This is a FAKE token made up for the lab. Never commit real secrets.
GITLAB_TOKEN = "glpat-abcdefghij1234567890"
```

Commit with message `Add fake token for secret detection demo`.

### 5.3 Look at the results

1. Open the new pipeline in **Build > Pipelines**.

**You should see:** extra jobs such as `semgrep-sast` (SAST) and `secret_detection` next to `unit-tests`. Wait for them to finish.

2. Click the SAST job. On the right side, under **Job artifacts**, click **Download** or **Browse**.
3. Open `gl-sast-report.json` (use **Browse** and click the file to view it).

**You should see:** a JSON file with a `vulnerabilities` list. Look for entries that mention `subprocess` with `shell=True` or `md5`. Those are the two insecure pieces of code we wrote on purpose.

4. Do the same for the `secret_detection` job and open `gl-secret-detection-report.json`.

**You should see:** a finding about a GitLab personal access token in `config.py`.

> **On Free** you read these as files. **On Ultimate** the same findings appear in the **Security** tab of the pipeline, in the merge request, and in a project-wide **Vulnerability Report** with severity, status, and an option to create an issue.

### 5.4 Fix a finding

1. Delete `config.py` (open it, click the three-dot menu, **Delete**) and commit.
2. Edit `app.py` and replace `hashlib.md5` with `hashlib.sha256`. Commit.
3. Run the pipeline and check the SAST report again.

**You should see:** the `md5` finding is gone from the report. (Secrets stay in Git history even after deleting the file, so a real leaked secret must always be **revoked**, not just deleted.)

---

## Step 6: Other Scanners (Overview, Read Only)

| Scanner | What it checks | When it runs | Needs |
|---|---|---|---|
| **Dependency scanning** | Libraries in files like `requirements.txt` or `package.json` against a database of known vulnerabilities | Pipeline, after code is pushed | Results in the UI need **Ultimate** |
| **Container scanning** | The operating system packages and libraries inside a **Docker image** you build | Pipeline, after the image is built | Results in the UI need **Ultimate**; needs a Docker image to scan |
| **DAST** | Your **running** website or API, by sending real attack-like requests | Pipeline, after deploying to a test environment | Needs a reachable running app and **Ultimate** |

How they are turned on (for reference only, you do not need to try these):

```yaml
include:
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: DAST.gitlab-ci.yml
```

> **Remember:** SAST reads code, dependency scanning reads libraries, container scanning reads images, DAST attacks a running app. Together they cover different layers.

---

## Checklist

- [ ] Created a project with code and tests
- [ ] Ran tests in a pipeline and read the log
- [ ] Opened the **Tests** tab and saw a failing test, then fixed it
- [ ] Saw the coverage percentage on the pipeline
- [ ] Saw red/green coverage lines in a merge request
- [ ] Enabled SAST and Secret Detection with `include:`
- [ ] Downloaded and read a SAST report and a Secret Detection report
- [ ] Fixed one finding and re-ran
- [ ] Can explain dependency scanning, container scanning, and DAST

## Quick Questions

1. What is the difference between a test report and code coverage?
2. Why is 100% coverage not a guarantee that code is correct?
3. What does SAST check, and what does DAST check?
4. Why is deleting a leaked secret from a file not enough?
5. Which security features are visible only on Ultimate?

---

## Stuck? Common Fixes

| Problem | Fix |
|---|---|
| Pipeline says YAML is invalid | Check indentation (spaces, not tabs). Use **Validate** in the Pipeline editor |
| No coverage percentage shown | The `coverage:` pattern must match the `TOTAL ... NN%` line in the log; check the job log shows a `TOTAL` row |
| **Tests** tab is empty | Make sure `artifacts: reports: junit: report.xml` is present and `when: always` is set |
| Red/green lines not showing in the MR | Needs a finished MR pipeline with `coverage_report` set; refresh the **Changes** tab after the pipeline ends |
| SAST job does not appear | Check the `include:` lines are spelled exactly and at the left margin of the file; check the pipeline ran on `main` or an MR |
| SAST finds nothing | Semgrep rules can change over time; check the job log says it scanned your `.py` files, and try the secret detection report instead |
| Cannot find the Security tab | It is Ultimate only; use the downloadable JSON reports on Free |
| Menu names look different | GitLab renames menus now and then; use the project search or look for similar wording |
