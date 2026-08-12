---
name: rails-gem-update
description: Safely evaluate and perform Ruby gem updates in a Rails modular monolith. Use when reviewing bundle outdated output, upgrading gems, analyzing release notes, handling migrations, or coordinating gem updates with frontend/npm dependencies.
---

# Rails Gem Update (Modular Monolith)

## When to use this skill

Use this skill when:
- running `bundle outdated`
- upgrading one or more gems
- reviewing gem release notes
- validating upgrade safety
- handling gem-related database migrations
- coordinating gem updates with JavaScript/npm dependencies
- preparing verification or rollout plans for dependency updates

## Core Principle

All updates are driven strictly by:

    bundle outdated

Do not introduce or evaluate gems outside this output unless explicitly instructed.

## Required Artifacts

Write artifacts to `.claude/gem-updates/` so they accumulate across gems in a session.

### Verification Plan → `.claude/gem-updates/verification-plan.md`

```markdown
# Verification Plan

## Gem: <gem_name>
- Current Version: <current_version>
- Target Version: <target_version>
- Release Type: Patch / Minor / Major

### Checks
- [ ]
```

---

### New Feature Report → `.claude/gem-updates/new-feature-report.md`

```markdown
# New Feature Report

## Gem: <gem_name>
- Current Version: <current_version>
- Target Version: <target_version>

### New Features Not Currently Used
-
```

## Update Workflow

### Step 1 — Identify Candidate

For each gem in `bundle outdated`:
- name
- current version
- target version
- classify: Patch / Minor / Major

Check if the gem is pinned in the Gemfile (e.g., `gem "foo", "~> 1.0"`). If the Gemfile constraint prevents reaching the target version, note that the Gemfile must be updated first and include the constraint change in the recommendation.

### Step 2 — Review Release Notes (MANDATORY)

Fetch upstream release notes using `WebFetch`. Try these sources in order:
1. `https://github.com/<owner>/<repo>/releases` — GitHub releases page
2. `https://github.com/<owner>/<repo>/blob/main/CHANGELOG.md` (also try `master`, `CHANGES.md`, `History.md`)
3. The gem's homepage URL from `gem info <gem_name>`

Extract:
- changelog entries between current and target versions
- upgrade guides
- breaking change notices

Do NOT summarize or infer release notes without fetching them. If no release notes can be found, explicitly state that and flag the gem for manual review.

### Step 3 — Evaluate

#### Patch Releases
- Accept by default
- Check for migrations or breaking changes

#### Minor / Major Releases
Evaluate:
- breaking changes
- deprecations
- migrations
- config changes

If feature is used:
- add verification check

If feature is not used:
- add to new feature report

#### Hold Criteria

Recommend **Hold** when any of the following apply:
- Major version with breaking changes that affect features used in this project
- Requires a Rails version bump beyond what is currently pinned
- Depends on another gem that must be updated first (coordinated upgrade)
- Release notes indicate known regressions or instability (e.g., yanked prior versions, open critical issues)
- Requires significant code migration that should be planned separately

### Step 4 — Database Migrations

Check for migrations.

Project uses pack-based paths:
- /packs/<pack_name>/db/migrate

If present:
- determine correct location
- add verification check

### Step 5 — NPM Coupling

Check if gem requires npm updates.

If yes:
- update npm package
- verify frontend behavior

### Step 6 — Execute Update

Once a gem is evaluated and recommended as **Accept**:

1. **Update the gem conservatively:**
   ```
   bundle update <gem_name> --conservative
   ```
   This avoids pulling in unrelated dependency changes.

2. **If the Gemfile constraint needed changing** (identified in Step 1), update the Gemfile first, then run `bundle install`.

3. **Run the test suite:**
   ```
   bundle exec rspec
   ```
   If a specific pack is primarily affected, run its tests first for faster feedback:
   ```
   bundle exec rspec packs/<pack_name>/spec
   ```

4. **If tests pass:** proceed to the next gem or finalize.

5. **If tests fail:** diagnose the failure.
   - If the fix is straightforward (e.g., a renamed method, changed config key), apply it and re-run tests.
   - If the failure is complex or unclear, **rollback** and recommend Hold:
     ```
     git checkout Gemfile Gemfile.lock
     bundle install
     ```
   - Add the failure details to the verification plan for follow-up.

6. **Do not commit automatically.** Present the update results to the user and let them decide when to commit.

## Output Format

## <gem_name>

- Current Version: <x.y.z>
- Target Version: <x.y.z>
- Release Type: Patch / Minor / Major
- Gemfile Constraint: <current constraint or "unpinned">
- Constraint Change Required: Yes (new constraint) / No
- Recommendation: Accept / Evaluate / Hold
- Hold Reason: <reason, if applicable>
- Migrations Required: Yes / No / Unclear
- Requires NPM Update: Yes / No / Unclear
- Tests Pass: Yes / No / Not yet run

### Verification Plan Additions
- [ ]

### New Feature Report Additions
-

## Final Deliverable

- per-gem evaluations
- verification plan (written to `.claude/gem-updates/verification-plan.md`)
- new feature report (written to `.claude/gem-updates/new-feature-report.md`)
- **upgrade report** (written to `.claude/gem-updates/upgrades-YYYY-MM-DD.md` — see below)
- migration steps
- npm alignment
- test results summary
- final recommendations

## Upgrade Report (REQUIRED)

After every session, write `.claude/gem-updates/upgrades-YYYY-MM-DD.md` (using today's date) with a full table of every gem that was evaluated. This file is date-stamped per session — create a new file each session.

```markdown
# Gem Upgrade Report — YYYY-MM-DD

| Gem | From | To | Type | Status | Hold Reason / Notes |
|-----|------|----|------|--------|---------------------|
| gem_name | x.y.z | x.y.z | patch/minor/major | ✅ Updated / ⏸ Hold / ❌ Blocked | reason if not updated |
```

**Status values:**
- `✅ Updated` — bundle update ran and tests pass
- `⏸ Hold` — intentional hold (breaking change, planned separately, Gemfile constraint)
- `❌ Blocked` — cannot update due to a dependency constraint from another gem (name the blocking gem)

## Session Completion Status (REQUIRED)

Always end every session with a clear status block so the user knows whether the task is finished or not:

```
## Update Session Status

- Gems updated this session: <count> (<list>)
- Gems held: <count> (<list with hold reasons>)
- Gems not yet evaluated: <count> (<list>)
- Task complete: Yes / No

<If No>: Run `/rails-gem-update` again to continue with the remaining gems.
```

**Never imply the task is done if gems remain in `bundle outdated` that have not been evaluated.**
If the session ends before all gems are processed, explicitly tell the user how many are left and that another run is needed.
