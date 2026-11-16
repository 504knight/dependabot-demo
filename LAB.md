# Dependabot lab guide

This guide follows the live session section for section. Use it to follow along, to catch up if you fall behind, or to redo the lab later on your own.

> [!WARNING]
> **Training repo only. Never deploy anything from it.** It pins vulnerable dependency versions on purpose.

Everything happens in **your own copy** of this repo, in the GitHub web UI. You don't need to run anything locally.

The numbers in this guide (alert counts, versions, pull requests) come from a full run on November 16, 2026. New advisories and releases appear all the time, so yours may differ slightly.

## Contents

- [Before the session](#before-the-session)
- [1. Why Dependabot](#1-why-dependabot)
- [2. Dependabot alerts](#2-dependabot-alerts)
- [3. Dependabot security updates](#3-dependabot-security-updates)
- [4. Version updates and dependabot.yml](#4-version-updates-and-dependabotyml)
- [5. Dependabot secrets](#5-dependabot-secrets)
- [6. Going further: auto-merge](#6-going-further-auto-merge)
- [Troubleshooting](#troubleshooting)

## Before the session

1. Open the template repo and click **Use this template** → **Create a new repository**.
2. Choose **your personal account** as the owner, so you're an admin of your copy.
3. Check **Include all branches**. Without it you won't have the checkpoint branches.
4. Create the repository.
5. Open the **Actions** tab of your copy and confirm the first **CI** run passes.

Create your copy close to the session. A copy is a snapshot, so an older one may miss later fixes to this guide and the checkpoints.

### If you fall behind

Each checkpoint branch is the known-good state at the **end** of a section.

| To start section                     | Reset `main` to                    |
| ------------------------------------ | ---------------------------------- |
| 4. Version updates                   | `checkpoint/03-security-updates`   |
| 5. Dependabot secrets                | `checkpoint/04-version-updates`    |
| 6. Going further (the finished repo) | `checkpoint/05-dependabot-secrets` |

```sh
git fetch origin
git checkout main
git reset --hard origin/checkpoint/03-security-updates
git push --force-with-lease origin main
```

Checkpoints only change files. Settings and secrets are separate, so also do the "enable" and "add a secret" steps of any section you skip.

## 1. Why Dependabot

Nothing to do in this section.

Dependencies drift. Every week you don't update, the gap to the current versions grows, and upgrades get harder and riskier. Meanwhile, new vulnerabilities are published against versions you already run. A project that was clean when it shipped can have dozens of known vulnerabilities a year later without a single commit.

Dependabot covers three jobs, and they're separate features:

| Feature              | What it does                                                                       | Configured in            |
| -------------------- | ---------------------------------------------------------------------------------- | ------------------------ |
| **Alerts**           | Tells you which of your dependencies have known vulnerabilities                    | Repository settings      |
| **Security updates** | Opens pull requests that fix those alerts                                          | Repository settings      |
| **Version updates**  | Opens pull requests that keep everything current, vulnerable or not, on a schedule | `.github/dependabot.yml` |

This repo is set up to show all three. It pins six dependencies at vulnerable versions, and leaves others a few releases behind.

## 2. Dependabot alerts

**You'll learn:** how to turn alerts on, how to read one, and why a monorepo shows some alerts twice.

### Steps

1. In your copy, go to **Settings** → **Advanced Security**.
2. Next to **Dependabot alerts**, click **Enable**. If the dependency graph is off, enable it when asked.
3. Open the **Security and quality** tab and choose **Dependabot** in the sidebar. Alerts appear within a minute or two.
4. Open the critical **mysql2** alert and find:
   - the **severity** and the advisory ID
   - the **affected** and **patched** versions
   - the **manifest** the alert belongs to (`yarn.lock` or `portal/package.json`)
5. Open a **fast-xml-parser** alert. Nothing in this repo lists that package in a `package.json`. It's a **transitive** dependency, pulled in by the AWS SDK.
6. Try the filters above the list: by severity, by package, by manifest.

### What you should see

About **34 alerts**, covering **24 distinct advisories**:

| Package                     | Advisories | Why it's here                                 |
| --------------------------- | ---------- | --------------------------------------------- |
| `fast-xml-parser`           | 7          | Transitive, through `@aws-sdk/client-s3`      |
| `mysql2`                    | 6          | Direct, includes the critical ones            |
| `postcss`                   | 4          | Transitive, through `next`                    |
| `zod`                       | 2          | Direct                                        |
| `knex`                      | 1          | Direct                                        |
| `luxon`                     | 1          | Direct                                        |
| `uuid`                      | 1          | Transitive, through the AWS SDK               |
| `minimatch`                 | 1          | Transitive, through tooling and `aws-cdk-lib` |
| `actions/download-artifact` | 1          | A GitHub Actions dependency in `ci.yml`       |

### Why some alerts appear twice

Alerts are tracked per manifest file. This repo is a Yarn workspace, so the portal's direct dependencies are found in two places: `portal/package.json` and the root `yarn.lock`. That's why mysql2, zod, knex and luxon each show a pair of alerts for the same advisory, 10 extra alerts in all.

It's one vulnerability and one fix. In section 3 you'll see a single pull request close both alerts.

### Notes

- Alerts come from the dependency graph. `dependabot.yml` has no effect on them.
- GitHub Actions alerts only fire for actions referenced by a version tag such as `@v4.1.2`, not by a commit SHA.

## 3. Dependabot security updates

**You'll learn:** how Dependabot fixes alerts with pull requests, what it can't fix on its own, and how to judge whether a fix is safe to merge.

### Steps

1. Go to **Settings** → **Advanced Security**.
2. Next to **Dependabot security updates**, click **Enable**. Leave **Grouped security updates** off for now.
3. Open the **Actions** tab. Dependabot's work shows up as runs named like `npm_and_yarn in /. for mysql2`.
4. Open the **Pull requests** tab. Some fixes arrive on their own within a few minutes.
5. Not every alert gets a pull request automatically. For each of these that has none after a few minutes, open its alert and click **Create Dependabot security update**:
   - mysql2
   - zod
   - knex
   - luxon
   - actions/download-artifact
6. Open the **zod** pull request and look at the version. It goes from 3.22.2 to **4.x**, a major upgrade.
7. Wait for **CI** to pass on each pull request, then merge them **one at a time**: mysql2, knex, zod, luxon, then download-artifact.
8. After each merge, give Dependabot a minute or two. The remaining npm pull requests now conflict on `yarn.lock`, and Dependabot rebases them itself. Merge the next one when CI is green again.
9. Go back to the alerts list.

### What you should see

- **Five pull requests**, all with CI passing:

  | Pull request                | From → to      |
  | --------------------------- | -------------- |
  | `mysql2`                    | 3.9.3 → 3.23.1 |
  | `zod`                       | 3.22.2 → 4.6.6 |
  | `knex`                      | 2.3.0 → 2.4.0  |
  | `luxon`                     | 3.2.0 → 3.2.1  |
  | `actions/download-artifact` | 4.1.2 → 4.1.3  |

- **About 13 alerts left** after merging. Each merge closed both the `yarn.lock` and the `portal/package.json` alert.
- **Failed Dependabot runs** in the Actions tab for fast-xml-parser, postcss, uuid and minimatch. That's expected, and explained below.

### What to notice

- **Dependabot picks the lowest version that fixes the alerts.** mysql2 moves to 3.23.1 because that's the first version clear of all six advisories.
- **A security update can be a major upgrade.** zod has two advisories, and the lowest version that fixes both is in zod 4. Dependabot never offers 3.22.3. Your CI is what tells you whether the jump is safe.
- **The compatibility score says "unknown."** That score is the share of CI runs that passed in other public repos making the same version jump. These jumps are too unusual to have data. Your own CI checks are the signal that counts.
- **Dependabot only rebases on conflict.** It won't update a pull request that's merely behind `main`. Avoid GitHub's **Update branch** button and don't push to a Dependabot branch: once you edit a pull request, Dependabot stops maintaining it.

### Why four alerts can't be fixed automatically

The remaining alerts are all transitive, and a parent package pins the vulnerable version:

| Package           | Held back by                                   |
| ----------------- | ---------------------------------------------- |
| `fast-xml-parser` | `@aws-sdk/core` 3.840.0 requires exactly 4.4.1 |
| `uuid`            | The same old AWS SDK                           |
| `postcss`         | `next` 15 requires exactly 8.4.31              |
| `minimatch`       | `aws-cdk-lib` and `typescript-eslint`          |

With Yarn, Dependabot won't upgrade the parent to reach a safe version of the child. It reports **update not possible** and names the conflicting dependency. Open one of the failed runs in the Actions tab to see the message.

The fix is to upgrade the parent yourself, or let version updates do it. You'll see that happen for the AWS SDK in section 4.

## 4. Version updates and dependabot.yml

**You'll learn:** how to keep every dependency current on a schedule, and how to keep the resulting pull requests manageable with groups, cooldown and ignore rules.

Security updates only react to vulnerabilities. Version updates are driven by a config file, `.github/dependabot.yml`, on your default branch.

### Steps

1. In your copy, switch the branch selector to `checkpoint/04-version-updates`.
2. Open `.github/dependabot.yml`, click **Raw** and copy the whole file.
3. Switch back to `main`. Click **Add file** → **Create new file**.
4. Name it `.github/dependabot.yml`, paste the contents, and commit directly to `main`.
5. Go to **Insights** → **Dependency graph** → **Dependabot**. You'll see three entries, one per ecosystem. Dependabot starts a run for each as soon as the file lands.
6. Next to the npm entry, click **Recent update jobs** to watch progress. **Check for updates** on that page starts a run on demand.
7. Open the **Pull requests** tab after a few minutes.
8. Open the **nextjs-major** pull request and look at its checks.
9. Merge one green pull request, such as the **typescript-tooling** one. Leave the **github-actions** pull request open. You need it in section 5.

### What you should see

**Twelve pull requests:** ten for npm, one for GitHub Actions and one for Docker.

| Pull request                     | Contents                                                      | CI             |
| -------------------------------- | ------------------------------------------------------------- | -------------- |
| `aws-sdk` group                  | `@aws-sdk/client-s3` 3.840.0 → 3.1167.0                       | Passes         |
| `aws-cdk` group                  | `aws-cdk-lib` 2.270.0 → 2.276.0                               | Passes         |
| `mui` group                      | 4 updates: MUI and Emotion                                    | Passes         |
| `react-core` group               | 4 updates: React and its types                                | Passes         |
| `testing` group                  | `vitest` 5.0.3 → 5.1.0                                        | Passes         |
| `typescript-tooling` group       | `prettier` 3.8.0 → 3.10.0                                     | Passes         |
| `production-dependencies` group  | 3 updates: luxon, mysql2, zod                                 | Passes         |
| `nextjs-major` group             | `next` and `eslint-config-next` 15 → 16                       | **Lint fails** |
| `typescript-tooling-major` group | `eslint` and `@eslint/js` 9 → 10                              | **Lint fails** |
| `knex`                           | 2.4.0 → 3.3.0, a major with no group                          | Passes         |
| `github-actions` group           | 5 actions, across the workflow and the local composite action | Passes         |
| Docker                           | `node` 22.12-alpine → 22.23-alpine                            | Passes         |

The npm run itself is marked **failed** in **Recent update jobs**, even though it opened its pull requests. That's the TypeScript update, explained below.

### Reading the config

**One entry for the whole workspace**

```yaml
- package-ecosystem: 'npm'
  directory: '/'
```

The repo has two `package.json` files but one root `yarn.lock`, so a single entry covers everything. One package manager and one lockfile means one set of pull requests. Separate lockfiles would need an entry each, and the same library would get separate pull requests.

**Schedule**

```yaml
schedule:
  interval: 'weekly'
  day: 'monday'
```

Pull requests arrive on Monday, so the team has the week to review and merge them.

**Cooldown**

```yaml
cooldown:
  semver-major-days: 14
  semver-minor-days: 5
  semver-patch-days: 3
```

Dependabot waits this long after a release before proposing it. A bad or compromised release is usually caught and pulled within days, so this keeps you off brand-new versions. Cooldown applies to version updates only. Security updates are never delayed.

**Groups**

```yaml
groups:
  mui:
    patterns:
      - '@mui/*'
      - '@emotion/*'
    update-types:
      - 'minor'
      - 'patch'
  mui-major:
    patterns:
      - '@mui/*'
      - '@emotion/*'
    update-types:
      - 'major'
```

A group turns many pull requests into one. Each library family here has two groups: minor and patch updates together, which are usually safe to merge as a batch, and majors separately, which need a proper look.

**How Dependabot picks a group**

When a dependency matches several groups, it joins the one with the **most specific pattern**, not the first one listed. An exact name beats `@scope/*`, which beats `*`.

That's why the catch-all groups at the bottom list `'*'`:

```yaml
production-dependencies:
  dependency-type: 'production'
  patterns:
    - '*'
```

Without that pattern, the catch-alls take every package matched only by a wildcard, and the `mui` and `aws-sdk` groups end up empty. If a group never produces a pull request, look in the job log for "groups where no dependencies match."

**Ignore rules**

```yaml
ignore:
  - dependency-name: '@types/node'
    update-types:
      - 'version-update:semver-major'
```

Node major upgrades touch the Dockerfile, `.nvmrc` and CI together, so they're planned by hand. The Docker entry ignores `node` majors for the same reason, which is why you get 22.12 → 22.23 and not a jump to a newer major.

**The security-patches group**

```yaml
security-patches:
  applies-to: 'security-updates'
  patterns:
    - '*'
  update-types:
    - 'patch'
    - 'minor'
```

This one groups **security** updates. With it, the four small fixes from section 3 would have arrived as one pull request. The zod fix is a major, so it would still be separate.

**GitHub Actions and Docker**

```yaml
- package-ecosystem: 'github-actions'
  directories:
    - '/'
    - '/.github/actions/*'
```

`/` covers the workflows in `.github/workflows`. The second path covers the local composite action, which also uses actions that need updating.

### What to notice

- **The AWS SDK update fixes what section 3 couldn't.** Merging the `aws-sdk` pull request moves `@aws-sdk/client-s3` far enough forward to drop the vulnerable fast-xml-parser and uuid versions. Staying current is what makes transitive vulnerabilities fixable.
- **CI catches the risky majors.** Next 16 and ESLint 10 both break lint. Nothing about the pull requests looks different from the green ones until the checks run.
- **TypeScript can't be updated at all.** The latest TypeScript is a new major, and `typescript-eslint` doesn't support it yet. Dependabot reports "peer dependencies are incorrectly met" and marks the run failed. This is what a blocked upgrade looks like. An `ignore` rule for TypeScript majors would turn the run green.
- **There's a limit.** `open-pull-requests-limit: 10` caps open npm version-update pull requests. This run uses all ten. Further updates wait until you merge or close some.
- **Regrouping takes two runs.** If you change groups while pull requests are open, a package that's already in an open pull request is skipped for one run. Click **Check for updates** again.

## 5. Dependabot secrets

**You'll learn:** why CI that needs a secret fails on Dependabot's pull requests, and how to give Dependabot access to a private registry.

GitHub keeps two separate sets of secrets:

| Secret type            | Available to                                                             |
| ---------------------- | ------------------------------------------------------------------------ |
| **Actions secrets**    | Workflows triggered by people                                            |
| **Dependabot secrets** | Dependabot itself, and workflows triggered by Dependabot's pull requests |

A workflow run triggered by Dependabot does **not** get your Actions secrets. That's deliberate: a dependency update is third-party code, and it shouldn't run with your credentials.

The secret values in this section are placeholders. Type anything, such as `demo-123`.

### Part A: a CI job that needs a secret

1. Go to **Settings** → **Secrets and variables** → **Actions** → **New repository secret**. Name it `INTEGRATION_API_TOKEN`, with any value.
2. On `main`, edit `.github/workflows/ci.yml` and add this job at the end, indented to match the other jobs. Commit directly to `main`.

   ```yaml
   integration-token:
     runs-on: ubuntu-latest
     steps:
       - name: Check that the integration token is available
         env:
           INTEGRATION_API_TOKEN: ${{ secrets.INTEGRATION_API_TOKEN }}
           ACTOR: ${{ github.actor }}
         run: |
           if [ -z "$INTEGRATION_API_TOKEN" ]; then
             echo "::error::INTEGRATION_API_TOKEN is not available to this run (actor: $ACTOR). If this is a Dependabot PR, add the same secret under Settings > Secrets and variables > Dependabot."
             exit 1
           fi
           echo "INTEGRATION_API_TOKEN is available (actor: $ACTOR)."
   ```

   It stands in for any step that needs a credential, such as integration tests or a preview deploy.

3. In the **Actions** tab, the CI run for your commit **passes**, including `integration-token`.
4. Open the **github-actions** pull request. Your edit to `ci.yml` conflicts with it, so Dependabot usually rebases it within a couple of minutes and CI runs again. If it hasn't after five minutes, click **Check for updates** on the GitHub Actions entry under **Insights** → **Dependency graph** → **Dependabot**.
5. This time `integration-token` **fails**: "INTEGRATION_API_TOKEN is not available to this run (actor: dependabot[bot])".

### Part B: fix it with a Dependabot secret

1. Go to **Settings** → **Secrets and variables** → **Dependabot** → **New repository secret**. Use the same name, `INTEGRATION_API_TOKEN`, with any value.
2. Back on the pull request, re-run the failed job.
3. It **passes**. A re-run of a Dependabot-triggered run still uses Dependabot's secrets.

### Part C: a private registry

Dependabot needs credentials to read packages from a private registry, and those credentials are also Dependabot secrets.

> [!IMPORTANT]
> Create the secret **before** you commit the config. If `dependabot.yml` names a secret that doesn't exist, every npm job fails and Dependabot answers comments with "Dependabot could not read a required secret."

1. Go to **Settings** → **Secrets and variables** → **Dependabot** → **New repository secret**. Name it `DEPENDABOT_PACKAGES_TOKEN`, with any value.
2. On `main`, edit `.github/dependabot.yml`. Add a `registries` block between `version: 2` and `updates:`.

   ```yaml
   version: 2

   registries:
     github-packages:
       type: npm-registry
       url: https://npm.pkg.github.com
       token: ${{secrets.DEPENDABOT_PACKAGES_TOKEN}}

   updates:
   ```

3. In the same file, let the npm entry use it. Commit directly to `main`.

   ```yaml
   - package-ecosystem: 'npm'
     directory: '/'
     registries:
       - github-packages
   ```

4. Edit `.yarnrc.yml` and add this at the end, so Yarn knows which packages come from that registry. Commit.

   ```yaml
   npmScopes:
     private-example:
       npmRegistryServer: 'https://npm.pkg.github.com'
   ```

5. Go to **Insights** → **Dependency graph** → **Dependabot** → **Recent update jobs** and click **Check for updates**. The npm jobs run as before.

Nothing in this repo depends on a `@private-example` package, so the placeholder token is never tested. In a real project the token is a credential with read access to your packages.

### What to notice

- **No token is committed anywhere.** `dependabot.yml` refers to the secret by name, and `.yarnrc.yml` only says where the scope lives. Developers keep their own token in `~/.yarnrc.yml`.
- **One secret can be needed in both places.** If CI installs private packages, the same token must exist as an Actions secret for your runs and as a Dependabot secret for Dependabot's.

## 6. Going further: auto-merge

Nothing to do in this section.

What you've built is the recommended baseline: alerts on, security updates on, and a `dependabot.yml` with a weekly schedule, cooldown, groups and a security-patches group.

The next step is to merge the safe updates automatically, starting with **security patches**. Most are small bumps like luxon 3.2.0 → 3.2.1, and the cost of leaving them open is higher than the risk of merging them.

Auto-merge is only as safe as your checks. Section 4 showed why: the Next 16 and ESLint 10 pull requests looked like any other until lint failed. Before turning it on:

- **Require status checks on `main`** with a branch ruleset, so nothing merges unless CI passes.
- **Make the checks meaningful.** Lint, typecheck and unit tests at minimum. End-to-end tests are what catch a library that still compiles but behaves differently.
- **Limit it to patch updates at first**, and keep majors manual.
- **Give Dependabot the secrets CI needs**, as in section 5, or its pull requests can never go green.

## Troubleshooting

**No security pull request appeared for an alert.**
Open the alert and click **Create Dependabot security update**. If the alert shows an error instead, the fix needs a parent upgrade. See [Why four alerts can't be fixed automatically](#why-four-alerts-cant-be-fixed-automatically).

**A Dependabot run is marked failed.**
Open it from the **Actions** tab, or from **Insights** → **Dependency graph** → **Dependabot** → **Recent update jobs** → **view logs**. The table at the end of the log names the dependency and the reason. In this repo, failures for fast-xml-parser, postcss, uuid, minimatch and TypeScript are expected.

**"Dependabot could not read a required secret."**
`dependabot.yml` names a secret that doesn't exist under **Settings** → **Secrets and variables** → **Dependabot**. Add it, then click **Check for updates**.

**`@dependabot rebase` gets no answer.**
Comment commands can take several minutes. **Check for updates** responds in seconds and refreshes open pull requests too.

**A group never opens a pull request.**
Check the job log for "groups where no dependencies match." See [How Dependabot picks a group](#reading-the-config).

**Dependabot comments that a label could not be found.**
Labels listed in `dependabot.yml` must already exist in the repo. Create them, or remove them from the config. Without a `labels` setting, Dependabot applies its default labels.

**The file in my checkpoint branch looks different from this guide.**
Your copy was created before a later fix. Take the file from the template repo's branch instead.

**I want to start over.**
Delete your copy and create a new one from the template. Settings and secrets start fresh.
