---
name: npm-package-update
description: Safely evaluate and perform npm/yarn package updates. Use when reviewing yarn outdated output, upgrading packages, analyzing release notes, handling breaking changes, or coordinating npm updates with Rails gem dependencies.
---

# NPM Package Update

## When to use this skill

Use this skill when:
- running `yarn outdated`
- upgrading one or more npm packages
- reviewing package release notes
- validating upgrade safety
- coordinating npm updates with Rails gem dependencies
- preparing verification or rollout plans for dependency updates

## Core Principle

All updates are driven strictly by:

    yarn outdated

Do not introduce or evaluate packages outside this output unless explicitly instructed.

## Required Artifacts

Write artifacts to `.claude/npm-updates/` so they accumulate across packages in a session.

### Verification Plan → `.claude/npm-updates/verification-plan.md`

```markdown
# Verification Plan

## Package: <package_name>
- Current Version: <current_version>
- Target Version: <target_version>
- Release Type: Patch / Minor / Major

### Checks
- [ ]
```

---

### New Feature Report → `.claude/npm-updates/new-feature-report.md`

```markdown
# New Feature Report

## Package: <package_name>
- Current Version: <current_version>
- Target Version: <target_version>

### New Features Not Currently Used
-
```

## Update Workflow

### Step 1 — Identify Candidate

For each package in `yarn outdated`:
- name
- current version (installed)
- wanted version (satisfies current range in package.json)
- latest version (newest published)
- classify the gap from current → latest: Patch / Minor / Major
- note whether it is a `dependency` or `devDependency`

Check the version range in `package.json` (e.g., `"^1.0.0"`, `"~2.3"`, `">=4"`). If the range already allows the latest version, `yarn upgrade <package>` is sufficient. If the range must be widened to reach the latest version, note that `package.json` must be updated and include the new range in the recommendation.

### Step 2 — Review Release Notes (MANDATORY)

Fetch upstream release notes using `WebFetch`. Try these sources in order:
1. `https://github.com/<owner>/<repo>/releases` — GitHub releases page
2. `https://github.com/<owner>/<repo>/blob/main/CHANGELOG.md` (also try `master`, `CHANGES.md`, `History.md`)
3. The package's npm page: `https://www.npmjs.com/package/<package_name>?activeTab=versions`

Extract:
- changelog entries between current and target versions
- migration / upgrade guides
- breaking change notices

Do NOT summarize or infer release notes without fetching them. If no release notes can be found, explicitly state that and flag the package for manual review.

### Step 3 — Evaluate

#### Patch Releases
- Accept by default
- Check for any noted breaking changes (some maintainers ship breaking changes in patches)

#### Minor / Major Releases
Evaluate:
- breaking changes
- deprecations
- API or config changes
- peer dependency requirement changes (e.g., requires a newer Node or a paired gem update)

If a changed feature is used in this project:
- add a verification check

If a changed feature is not used:
- add to new feature report

#### Hold Criteria

Recommend **Hold** when any of the following apply:
- Major version with breaking changes that affect APIs used in this project
- Requires a Node version bump beyond what is currently specified in `engines` in `package.json`
- Paired gem must be updated first (e.g., `@hotwired/turbo-rails` is coupled to the `turbo-rails` gem)
- Release notes indicate known regressions or instability (e.g., yanked versions, open critical issues)
- Requires significant code migration that should be planned separately

### Step 4 — Gem Coupling Check

Some npm packages are tightly coupled to a Rails gem and must be updated together:

| npm package | Rails gem |
|---|---|
| `@hotwired/turbo-rails` | `turbo-rails` |
| `@hotwired/stimulus` | `stimulus-rails` |
| `@rails/activestorage` | `activestorage` (part of Rails) |
| `@rails/actioncable` | `actioncable` (part of Rails) |

If a package has a coupled gem:
- check whether the gem version is already compatible with the target npm version
- if both need updating, coordinate them (update gem first, then npm, or vice versa per the package's docs)
- add a verification check

### Step 5 — Execute Update

Once a package is evaluated and recommended as **Accept**:

1. **If the `package.json` range already allows the target version**, upgrade in place:
   ```
   yarn upgrade <package_name>
   ```

2. **If the `package.json` range must be widened**, update the range first, then install:
   ```
   yarn upgrade <package_name>@<new_range>
   ```
   This updates both `package.json` and `yarn.lock`.

3. **Rebuild assets** to catch bundler errors:
   ```
   yarn build
   ```
   If only one entrypoint is affected, run the targeted build for faster feedback:
   ```
   yarn build:ssp
   # or
   yarn build:admin
   ```

4. **Run the JavaScript test suite:**
   ```
   yarn test
   ```

5. **If build and tests pass:** proceed to the next package or finalize.

6. **If build or tests fail:** diagnose the failure.
   - If the fix is straightforward (e.g., a renamed import, changed config key), apply it and re-run.
   - If the failure is complex or unclear, **rollback** and recommend Hold:
     ```
     git checkout package.json yarn.lock
     yarn install
     ```
   - Add the failure details to the verification plan for follow-up.

7. **Do not commit automatically.** Present the update results to the user and let them decide when to commit.

## Output Format

## <package_name>

- Current Version: <x.y.z>
- Target Version: <x.y.z>
- Release Type: Patch / Minor / Major
- Dependency Type: dependency / devDependency
- package.json Range: <current range or "exact">
- Range Change Required: Yes (new range) / No
- Recommendation: Accept / Evaluate / Hold
- Hold Reason: <reason, if applicable>
- Gem Coupling: Yes (<gem_name>) / No
- Tests Pass: Yes / No / Not yet run

### Verification Plan Additions
- [ ]

### New Feature Report Additions
-

## Upgrade Report (REQUIRED)

After every session, write `.claude/npm-updates/upgrades-YYYY-MM-DD.md` (using today's date) with a full table of every package that was evaluated. This file is date-stamped per session — create a new file each session.

```markdown
# NPM Upgrade Report — YYYY-MM-DD

| Package | From | To | Type | Status | Notes |
|---------|------|----|------|--------|-------|
| package-name | x.y.z | x.y.z | patch/minor/major | ✅ Updated / ⏸ Hold / ❌ Blocked | reason if not updated |
```

**Status values:**
- `✅ Updated` — yarn upgrade ran and build/tests pass
- `⏸ Hold` — intentional hold (breaking change, planned separately, blocked dependency)
- `❌ Blocked` — cannot update due to a constraint from another package (name the blocking package)

## Final Deliverable

- per-package evaluations
- verification plan (written to `.claude/npm-updates/verification-plan.md`)
- new feature report (written to `.claude/npm-updates/new-feature-report.md`)
- upgrade report (written to `.claude/npm-updates/upgrades-YYYY-MM-DD.md`)
- gem coupling notes and coordination steps
- build and test results summary
- final recommendations
