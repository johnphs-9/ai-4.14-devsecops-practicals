# Lesson 4.14: DevSecOps Practicals - OSV-Scanner and OWASP ZAP

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Add** OSV-Scanner to a repository with GitHub Actions, pinned by commit SHA
2. **Distinguish** blocking security gates from non-blocking security checks
3. **Run** an OWASP ZAP baseline (DAST) scan against a deployed application and interpret its alerts

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lesson 4.13 (DevSecOps Foundations), including **Activity 3** (Snyk)
- Your **devops-demo** repository on GitHub (**public**), with the CircleCI pipeline from Lesson 4.13
  (`snyk_scan` failing is expected - the release is blocked until developers fix the findings)
- Your security ticket (GitHub issue) from Lesson 4.13
- Your **APP URL** (e.g. `https://devops-demo-sample.up.railway.app`)
- GitHub Actions basics from Lesson 4.9

---

## Introduction

In Lesson 4.13, Snyk became a **gate** in your CircleCI pipeline: HIGH and CRITICAL issues stopped publish and deploy, and you reported them to the developers in a security ticket.

Today you'll add two more security checks - this time with **GitHub Actions**, in the same `devops-demo` repository:

| Check | Tool | Type (Lesson 4.13) | When it runs | Blocks deploy? |
|---|---|---|---|---|
| Dependency scan | **OSV-Scanner** (Google) | SCA | Every push to `main` + every Monday | **No** |
| Running-app scan | **OWASP ZAP** | DAST | On demand after a deploy + every Monday | **No** |

```
CircleCI (gate):       build ─► test ──┐
                       snyk_scan ──────┴─► publish ─► deploy ─► Railway
                                                                  ▲
GitHub Actions         OSV-Scanner: scans the repository          │
(watchdogs):           ZAP: scans the live app ───────────────────┘
```

Neither new workflow is connected to CircleCI - they never block a release. They **watch** and **report**, and the results land in GitHub.

---

## Part 1 - Gates vs Watchdogs

### Blocking checks (gates)

- Run **inside** the delivery pipeline
- A failure **stops** the release
- Must be **reliable** and use a **threshold** you can live with - otherwise nobody can ship
- Today: `snyk_scan` in CircleCI (fails on HIGH and CRITICAL)

### Non-blocking checks (watchdogs)

- Run **beside** the pipeline, or on a schedule
- A failure **alerts** the team, but doesn't stop the release
- Good for checks that are **noisy**, **slow**, or that find problems in code you **didn't change**
- Today: OSV-Scanner and ZAP

### Why OSV-Scanner is a watchdog here

- OSV-Scanner fails on **any** known vulnerability - LOW, MEDIUM, HIGH or CRITICAL. It has no simple severity threshold
- As a gate, a single LOW issue in a Spring Boot library would stop every deployment
- As a watchdog, it reports everything; the team decides what to fix, and when

### Why scan on a schedule?

New vulnerabilities are published **every day** for libraries that were safe yesterday. A weekly scan catches them even if nobody pushes code.

---

## Part 2 - Add OSV-Scanner with GitHub Actions

**OSV-Scanner** is Google's free, open-source SCA tool. It reads `pom.xml`, works out every dependency (including transitive ones), and checks them against [OSV.dev](https://osv.dev) - an open database that combines the GitHub Advisory Database, NVD and others. **No account or API key needed.**

Google provides a ready-made GitHub Actions workflow for it.

### Step 1: Create the Workflow File

In your **devops-demo** project, create `.github/workflows/osv-scanner.yml`:

```yaml
name: osv-scanner

on:
  push:
    branches: [ main ]
  schedule:
    - cron: "30 1 * * 1"   # every Monday 01:30 UTC (09:30 Singapore)
  workflow_dispatch:       # adds a "Run workflow" button

permissions:
  actions: read
  contents: read
  security-events: write

jobs:
  osv-scan:
    uses: google/osv-scanner-action/.github/workflows/osv-scanner-reusable.yml@b1d267aea5211fa648d9d442fa64b90dbdcbdd36 # v2.7.0
```

### Step 2: Understand the Workflow

| Part | Meaning |
|---|---|
| `push` to `main` | Scan every change |
| `schedule` | Scan every Monday - catches new CVEs in unchanged code |
| `workflow_dispatch` | Adds a **Run workflow** button in the Actions tab |
| `permissions` | Read the code and **write results** to the Security tab (`security-events: write`) - nothing else |
| `uses: ...osv-scanner-reusable.yml@...` | Google's ready-made **reusable workflow** - it checks out the code, runs OSV-Scanner and uploads the results |

**Why the long SHA instead of `@v2.7.0`?**

Remember the Trivy attack (Lesson 4.13): attackers moved **version tags** to malicious code. A full **commit SHA** can't be moved. The comment `# v2.7.0` keeps it readable.

Google does the same inside its workflow - every action it uses is pinned by SHA:

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

**Copy the SHA exactly** - one wrong character and GitHub can't find the workflow.

<!-- INSTRUCTOR: SHAs checked 2026-10-21 with `git ls-remote https://github.com/google/osv-scanner-action.git refs/tags/v2.7.0^{}` (b1d267ae...). To update, run the same command for the new tag and use the peeled (^{}) commit. -->

### Step 3: Commit and Push

```bash
git add .github/workflows/osv-scanner.yml
git commit -m "Add OSV-Scanner GitHub Actions workflow"
git push origin main
```

This push starts **both** CI systems: CircleCI runs your pipeline, and GitHub Actions runs OSV-Scanner.

### Step 4: Watch the Workflow

1. On GitHub, open your repository → **Actions** tab
2. Click the **OSV-Scanner** run → the **osv-scan** job
3. Open the **Run scanner** and **Run osv-scanner-reporter** steps

**Two possible results:**
- ✅ **Green** - no known vulnerabilities in your dependencies today
- ❌ **Red** - at least one known vulnerability, at **any** severity. Expect red: devops-demo's dependencies already have known vulnerabilities (Lesson 4.13). It doesn't affect CircleCI

### Step 5: See the Results in the Security Tab

1. Repository → **Security** tab → **Code scanning**
2. Filter by tool: **osv-scanner**
3. Click an alert to see the package, the vulnerability ID, its severity and the fixed version

**Why the Security tab?** Results stay in one place, can be **dismissed with a reason**, and close automatically when a later scan no longer finds them. On public repositories this is free.

Also notice the commit on your repository's main page: it now shows **two** status checks - CircleCI and GitHub Actions.

---

## Part 3 - DAST: Scan Your Live App with OWASP ZAP

### What Is OWASP ZAP?

**ZAP (Zed Attack Proxy)** is a free, open-source **DAST** tool. Instead of reading code, it talks to your **running** application over HTTP - like a browser, or an attacker - and reports what it finds.

| Scan type | What ZAP does | Safe for a live app? |
|---|---|---|
| **Baseline** | Crawls the app for about a minute and **passively** inspects the responses: headers, cookies, error pages | ✅ Yes - it sends normal requests only |
| **Full scan** | Also **actively attacks** the app: injection, path traversal and more | ❌ Only against test environments you own |

Today we run the **baseline** scan.

> **⚠️ Only scan applications you own or have written permission to test.** Scanning someone else's website can be illegal.

### Step 1: Store Your APP URL as a Repository Variable

The URL isn't a secret, but it's still configuration - keep it out of the workflow file.

1. GitHub repository → **Settings** → **Secrets and variables** → **Actions**
2. Click the **Variables** tab → **New repository variable**
3. **Name:** `APP_URL`
4. **Value:** your APP URL, e.g. `https://devops-demo-sample.up.railway.app` or `https://devops-demo-sample.onrender.com`
5. Click **Add variable**

### Step 2: Create the Workflow File

Create `.github/workflows/zap-baseline.yml`:

```yaml
name: zap-baseline-scan

on:
  workflow_dispatch:       # run it by hand after a deploy
  schedule:
    - cron: "0 2 * * 1"    # every Monday 02:00 UTC (10:00 Singapore)

permissions:
  contents: read

jobs:
  zap-baseline:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - name: ZAP baseline scan of the app
        uses: zaproxy/action-baseline@de8ad967d3548d44ef623df22cf95c3b0baf8b25 # v0.15.0
        with:
          target: ${{ vars.APP_URL }}
          allow_issue_writing: false
          fail_action: false

      - name: Upload ZAP report (viewable in browser)
        if: always()
        uses: actions/upload-artifact@v7
        with:
          path: report_html.html
          archive: false

      - name: Add ZAP report to job summary
        if: always()
        run: cat report_md.md >> "$GITHUB_STEP_SUMMARY"
```

| Part | Meaning |
|---|---|
| `workflow_dispatch` | Run it by hand after a deploy |
| `schedule` | Re-check the live app every Monday |
| `vars.APP_URL` | Your repository variable from Step 1 |
| `allow_issue_writing: false` | Don't open a GitHub issue - we'll read the report instead |
| `fail_action: false` | Don't fail the workflow because of alerts - a **watchdog**, like OSV-Scanner |
| Pinned SHAs | Same rule as Part 2 |

<!-- INSTRUCTOR: zaproxy/action-baseline v0.15.0 = de8ad967..., actions/checkout v7.0.1 = 3d3c42e5... (git ls-remote, 2026-10-21). The action pulls ghcr.io/zaproxy/zaproxy:stable by default (a tag) - an optional extension is setting docker_name to a digest. Run it once against your own Railway app to see which alerts appear before class. -->

```bash
git add .github/workflows/zap-baseline.yml
git commit -m "Add ZAP baseline scan workflow"
git push origin main
```

### Step 3: Run the Scan

1. Repository → **Actions** tab → **ZAP Baseline Scan** (left sidebar)
2. Click **Run workflow** → **Run workflow**
3. Wait 2-3 minutes for it to finish

### Step 4: Read the Report

1. Open the finished run → scroll to **Artifacts** → download **zap_scan**
2. Unzip it and open **`report_html.html`** in your browser

**Each alert shows:**

| Field | Meaning |
|---|---|
| **Risk** | High, Medium, Low or Informational |
| **Alert** | What ZAP found, e.g. "Content Security Policy (CSP) Header Not Set" |
| **URL** | Where it found it |
| **Description** | Why it matters |
| **Solution** | How to fix it |

**What you'll probably see** - devops-demo is a minimal API without Spring Security, so ZAP is likely to report **missing security headers**, for example:

| Alert | What the header does |
|---|---|
| Content Security Policy (CSP) Header Not Set | Controls which scripts and resources a browser may load |
| Missing Anti-clickjacking Header | Stops other sites from embedding your pages in a frame |
| X-Content-Type-Options Header Missing | Stops browsers guessing a response's content type |

**Notice what ZAP found that Snyk and OSV-Scanner can't:** these are problems in how the app **behaves when running** - not in its libraries. That's why DAST complements SCA.

### Step 5: Discuss

1. Why is the baseline scan safe to run against your live app, while a full scan is not?
2. Which of the alerts would you fix first, and why?
3. Snyk, OSV-Scanner and ZAP all reported different things. What does each one see that the others can't?
4. Should the ZAP scan block deployment? What would happen if it did?

<!-- Answers: 1) Baseline only sends normal requests and inspects responses; a full scan sends attack payloads that can change data or take the app down. 2) Usually the highest risk first; missing headers are Low/Medium but cheap to fix (Spring Security adds most of them by default). 3) Snyk/OSV: known CVEs in libraries (SCA). ZAP: behaviour of the running app - headers, cookies, errors (DAST). 4) ZAP needs a running app, so it can only run after deploy - blocking would mean deploying to a test environment first, scanning, then promoting to production. -->

---

## Part 4 - Your Complete DevSecOps Setup

| Where | Tool | Type | Role |
|---|---|---|---|
| Snyk website | Snyk | SCA | Dashboard; re-tests your project and emails you about new issues |
| CircleCI | Snyk (`snyk_scan`) | SCA | **Gate** - blocks publish and deploy on HIGH and CRITICAL |
| GitHub Actions | OSV-Scanner | SCA | **Watchdog** - every push and every Monday; results in the Security tab |
| GitHub Actions | OWASP ZAP | DAST | **Watchdog** - scans the live app |

**Defence in depth:** different tools, different data, different stages. No single scanner catches everything.

---

## Best Practices for DevSecOps

**1. Gate on what must never ship; watch everything else** - gates need reliable thresholds; watchdogs can report everything

**2. Scan on a schedule** - new CVEs appear for code you haven't touched

**3. Pin actions and tools by commit SHA or hash** - not by tag (Lesson 4.13)

**4. Give workflows the least permissions** - `contents: read` plus only what they need

**5. DAST only against apps you own** - baseline for live apps; full scans in test environments

---

## Summary

### What You Accomplished Today

1. ✅ Added OSV-Scanner with Google's reusable workflow, pinned by commit SHA
2. ✅ Saw results in the GitHub **Security** tab
3. ✅ Compared a **watchdog** (OSV-Scanner) with a **gate** (Snyk in CircleCI)
4. ✅ Accepted one finding temporarily with `osv-scanner.toml` (reason + expiry + ticket)
5. ✅ Saw that the other findings stay visible until developers fix them
6. ✅ Ran an OWASP ZAP baseline scan against your live app
7. ✅ Read ZAP's report and compared DAST with SCA

---

## Troubleshooting Guide

### Issue 1: "Unable to resolve action" / workflow not found

**Cause:** The commit SHA is wrong or incomplete

**Solution:** Copy the full 40-character SHA again - one wrong character breaks it.

---

### Issue 2: OSV-Scanner fails with "Resource not accessible by integration" when uploading results

**Cause:** Missing permissions

**Solution:**
1. Check the workflow has `security-events: write`, `actions: read` and `contents: read`
2. Check your repository is **public** (code scanning results are free for public repositories)

---

### Issue 3: OSV-Scanner is red but you didn't add anything

**Cause:** A new vulnerability was published for one of your existing dependencies (often a transitive Spring Boot dependency)

**Solution:**
1. Open the alert in the Security tab - it names the package and the fixed version
2. Fix it (often a newer Spring Boot version), or ignore it temporarily with a reason and an expiry

---

### Issue 4: The ignore doesn't work

**Solution:**
1. `osv-scanner.toml` must be in the **same folder** as `pom.xml`
2. The `id` must match the alert **exactly** (try the GHSA ID if the CVE ID doesn't work, or the other way round)
3. `ignoreUntil` must be in the future and use the format `2026-12-31T00:00:00Z`

---

### Issue 5: ZAP fails with "target not reachable" or an empty URL

**Solution:**
1. Check the repository **variable** (not secret) `APP_URL` exists - Settings → Secrets and variables → Actions → **Variables**
2. Check the URL starts with `https://` and the app is running (open `/hello` in your browser)

---

### Issue 5: There's no "Run workflow" button

**Solution:** The workflow file must be on the **default branch** (`main`) and include `workflow_dispatch:`.

---

## Additional Resources

- [OSV-Scanner Documentation](https://google.github.io/osv-scanner/)
- [OSV-Scanner GitHub Action](https://google.github.io/osv-scanner/github-action/)
- [OSV.dev Vulnerability Database](https://osv.dev/)
- [OWASP ZAP](https://www.zaproxy.org/)
- [ZAP Baseline Scan](https://www.zaproxy.org/docs/docker/baseline-scan/)
- [GitHub: Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [GitHub Code Scanning](https://docs.github.com/en/code-security/code-scanning)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
