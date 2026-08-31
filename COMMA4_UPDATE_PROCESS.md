# Steve’s Approved Comma 4 SunnyPilot Update Process

This is the approved, repeatable process for updating SunnyPilot on Steve’s Comma 4 Hyundai Genesis 3.8 HTRAC. Perform **one step at a time and wait for the output after every step**. Do not continue past an unexpected result.

## Locations

- Mac Studio repository: `/Users/stephennewson/Documents/GitHub/sunnypilot`
- Comma 4 deployment directory: `/data/openpilot`
- Comma 4 access: SSH

## 1. Research before changing anything

Before modifying the checkout, research the current `release-mici` and `release-mici-staging` branches, their changelogs, and the Openpilot base they use. Specifically assess driver-monitoring changes and note whether they could affect this vehicle or the update.

Do not assume a release branch has normal merge ancestry. SunnyPilot release branches are sometimes force-rewritten or represented by root commits. Use **tree-to-tree comparisons** when ancestry is unreliable.

## 2. Preserve the production branch

On the Mac Studio, identify the currently deployed/known-good production branch and leave it unchanged. Create a new versioned branch from the appropriate upstream release branch, using this naming pattern:

`steve/c4-release-mici-vYYYY.NNN.NNN-r1`

Example:

```sh
# Run one command at a time and wait for its output.
cd /Users/stephennewson/Documents/GitHub/sunnypilot
git status --short --branch
git branch --show-current
git fetch upstream --prune
git branch --list
git branch -r
```

Create the versioned branch from `upstream/release-mici` only after confirming that it is the intended release:

```sh
git switch --create steve/c4-release-mici-vYYYY.NNN.NNN-r1 upstream/release-mici
```

If comparing force-rewritten branches, compare commit trees or file contents directly rather than relying on `git merge-base` or a merge diff.

## 3. Read-only verification before changes

Confirm the new branch, working-tree state, upstream refs, recent commits, release/staging relationship, changelog, and Openpilot base. These checks must not modify files or history.

```sh
git status --short --branch
git log --oneline --decorate -n 10
git diff --stat upstream/release-mici upstream/release-mici-staging
git log --oneline --decorate upstream/release-mici..upstream/release-mici-staging --
```

Inspect the relevant changelog and base/version files, then specifically inspect driver-monitoring changes before applying customizations.

## 4. Reapply the two reusable Hyundai Genesis commits

Apply the two known custom changes to the new branch:

1. Lower lateral-control enable speed to **15 mph** (`minSteerSpeed 54 KPH`).
2. Suppress the below-steer-speed visual and audible alert.

Use the known reusable commit IDs when available. If commit IDs differ because of branch rewriting, locate equivalent commits by inspecting patch content and apply the matching changes carefully. Do not blindly apply unrelated commits.

```sh
# Replace these placeholders with the two verified reusable commit IDs.
git cherry-pick <lower-lateral-enable-speed-commit>
git cherry-pick <suppress-below-steer-speed-alert-commit>
```

Resolve conflicts conservatively and stop for review if either patch no longer applies cleanly.

## 5. Verify and publish the update branch

Review the complete diff, confirm both customizations are present, confirm no unintended files changed, and verify the branch is based on the intended upstream release.

```sh
git status --short --branch
git diff --stat upstream/release-mici...HEAD
git diff upstream/release-mici...HEAD
git log --oneline --decorate upstream/release-mici..HEAD
```

After verification, push the new branch to `origin`:

```sh
git push --set-upstream origin steve/c4-release-mici-vYYYY.NNN.NNN-r1
```

Preserve the prior production branch locally and remotely for immediate rollback.

## 6. Deploy to Comma 4

SSH to the Comma 4 and work only in `/data/openpilot`. Before changing branches, perform read-only checks and confirm the current branch is clean and record the current branch and head.

```sh
ssh <comma4-host>
cd /data/openpilot
git status --short --branch
git branch --show-current
git rev-parse HEAD
git remote -v
```

Fetch the newly published branch, switch to it, and verify the tracking branch and head:

```sh
git fetch origin --prune
git switch --track origin/steve/c4-release-mici-vYYYY.NNN.NNN-r1
git status --short --branch
git branch -vv
git rev-parse HEAD
```

Only after verification, reboot:

```sh
sudo reboot -h now
```

## 7. Post-boot verification and test drive

After the Comma 4 returns:

- Verify AGNOS is healthy and the device is online.
- Verify `/data/openpilot` is clean, on the new tracking branch, and at the expected head.
- Verify the two Hyundai Genesis custom settings are present.
- Confirm driver monitoring behaves as expected based on the earlier research.
- Perform a cautious test drive and record lane-centering, alerts, engagement behavior, and any driver-monitoring issues.

Keep the previous known-good branch available for immediate rollback until the test drive is complete and the update is accepted.

## Rollback

If the update is unsuitable, first perform read-only status checks on the Comma 4. Then switch `/data/openpilot` back to the complete previous known-good branch and reboot. Full previous-branch rollback is the preferred strategy; do not attempt to reverse individual commits during an incident.

```sh
ssh <comma4-host>
cd /data/openpilot
git status --short --branch
git branch --show-current
git rev-parse HEAD
git fetch origin --prune
git switch <previous-known-good-branch>
git status --short --branch
git branch -vv
git rev-parse HEAD
sudo reboot -h now
```

After reboot, repeat the post-boot checks and confirm the prior branch, expected head, AGNOS health, and vehicle behavior.
