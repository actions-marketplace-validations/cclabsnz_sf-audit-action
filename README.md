# sf-audit GitHub Action

Runs a read-only security audit of a live Salesforce org from GitHub Actions and fails the job when its posture slips. It wraps [sf-audit](https://github.com/cclabsnz/sf-audit-plugin), an open-source `sf` CLI plugin: 93 checks across identity, permissions, sharing, guest access, connected apps, secrets, monitoring and code patterns, graded A to F and correlated into attack chains.

Code scanners such as Salesforce Code Analyzer read your repository. This action reads the org itself: who holds which permissions, what guest users can reach, which settings drifted. Run both.

> **Use a private repository.** On a public repository, anyone can read the job summary, and anyone signed in to GitHub can download the artifact. Both describe your org's weaknesses. On a public repository, set `job-summary: 'false'` and `upload-artifact: 'false'`, or better, do not run it there.

## Quick start: audit production every night

```yaml
name: Salesforce security audit

on:
  schedule:
    - cron: '0 17 * * *'   # daily, 17:00 UTC
  workflow_dispatch:

permissions:
  contents: read

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: cclabsnz/sf-audit-action@v1
        with:
          auth-url: ${{ secrets.SF_AUDIT_AUTH_URL }}
          fail-on: HIGH
          plugin-version: '1.15.1'
```

The job fails when any finding is HIGH or CRITICAL. The job summary shows the grade, open findings by severity, the attack chains and the 15 most severe findings. The full HTML and Markdown reports are in the `sf-audit-report` artifact.

## Give it a read-only user

The audit only reads, and a test in the plugin's build fails if any code path could write to an org. Even so, do not hand CI an administrator's credentials. Create a dedicated integration user and assign the read-only permission set the plugin ships: [PERMISSIONS.md](https://github.com/cclabsnz/sf-audit-plugin/blob/main/PERMISSIONS.md) lists what each permission is for, and what the audit deliberately does not need. It never needs Modify All Data, and View All Data only makes one probe (`integration-least-privilege`) conclusive; every check runs without it.

Store its credentials as a repository or environment secret. If your security team will not allow production credentials in GitHub at all, run the plugin from a scheduler they do trust instead. The action is a convenience, not a requirement.

## Logging in

**SFDX auth URL.** Log in once on your machine as the integration user, then read the URL and store it as a secret:

```bash
sf org login web --alias audit-user
sf org display --target-org audit-user --verbose --json | jq -r '.result.sfdxAuthUrl'
```

The URL contains a refresh token. Anyone holding it can act as that user, so treat it like a password.

**JWT.** If you already use a connected app with a certificate for CI:

```yaml
      - uses: cclabsnz/sf-audit-action@v1
        with:
          jwt-client-id: ${{ secrets.SF_CLIENT_ID }}
          jwt-key: ${{ secrets.SF_JWT_KEY }}
          jwt-username: audit.user@example.com
          instance-url: https://login.salesforce.com
```

The action writes the key or URL to a file readable only by the runner user, logs in, and deletes the file before the audit runs.

## Gating a deployment

A pull request gate only makes sense against an org that already holds the change. A fresh scratch org has default configuration, so auditing it proves nothing. Audit the validation sandbox your pipeline deploys to:

```yaml
      - name: Deploy to the validation sandbox
        run: sf project deploy start --target-org validation
      - uses: cclabsnz/sf-audit-action@v1
        with:
          auth-url: ${{ secrets.SF_VALIDATION_AUTH_URL }}
          fail-on: HIGH
```

Do not run this on `pull_request` events from forks with secrets available. GitHub withholds secrets from fork pull requests by default; keep it that way.

## Inputs

| Input | Default | Description |
|---|---|---|
| `auth-url` | | SFDX auth URL. Use this or the JWT inputs. |
| `jwt-client-id` | | Connected app consumer key, for JWT login. |
| `jwt-key` | | PEM private key, for JWT login. |
| `jwt-username` | | Integration user's username, for JWT login. |
| `instance-url` | `https://login.salesforce.com` | Login URL for JWT. Use `https://test.salesforce.com` for sandboxes. |
| `fail-on` | | Fail when a finding is at or above `CRITICAL`, `HIGH`, `MEDIUM` or `LOW`. Empty never fails on findings. |
| `fail-on-inconclusive` | `false` | Fail when a check could not gather evidence. |
| `checks` | | Comma-separated check IDs. Empty runs all 93. `sf audit list` prints them. |
| `format` | `html,md` | Report formats: `html`, `md`, `json`, `executive`. JSON is always added. |
| `output-dir` | `sf-audit-report` | Where reports are written. |
| `job-summary` | `true` | Write a digest to the job summary. |
| `upload-artifact` | `true` | Upload the reports as an artifact. |
| `artifact-name` | `sf-audit-report` | Artifact name. |
| `plugin-version` | `latest` | `@cclabsnz/sf-audit` version. Pin it. |
| `cli-version` | `latest` | `@salesforce/cli` version. |
| `node-version` | `22` | Node.js version. |

## Outputs

| Output | Description |
|---|---|
| `grade` | Letter grade, A to F. |
| `health-score` | Health score, 0 to 100. |
| `json-report` | Path to the JSON report, for your own steps to read. |
| `exit-code` | `0` clean, `1` a finding at or above `fail-on`, `3` inconclusive with `fail-on-inconclusive`. |

## What it does not do

It reads configuration, so it cannot prove what a public site returns to an anonymous visitor; test that from outside with a tool such as [AuraInspector](https://github.com/google/aura-inspector), against your own sites. Its code checks are pattern checks, not data-flow analysis; use Salesforce Code Analyzer for that. And an inconclusive check means the user could not see enough to judge, not that the org passed. `sf audit preflight` tells you which checks a user will be blind to before you spend a run.

## Versioning

Use `@v1` to follow compatible updates, or pin a full commit SHA, which supply-chain scanners such as OpenSSF Scorecard prefer. Releases are listed on the [releases page](https://github.com/cclabsnz/sf-audit-action/releases).

## Licence

Apache-2.0. The plugin it runs is Apache-2.0 too.
