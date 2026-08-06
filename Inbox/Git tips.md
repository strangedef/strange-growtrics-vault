# Rebase onto vs cherry pick
Here is the side-by-side visual comparison.

In both scenarios, we want to take commits `D` and `E` (currently on the `feature` branch) and apply them to commit `C` (on the `main` branch).

---

## Scenario 1: Using `git cherry-pick`

To use cherry-pick, you must stand on `main`. It copies the data over but leaves the original branch behind, creating duplicate history.

```text
STEP 1: You must be on the destination branch
git checkout main

STEP 2: Run the command
git cherry-pick D E
```

The Visual Result:

```text
                  (Old branch is left untouched)
                  feature ─── D ─── E
                 /
A ─── B ─── C ─── D' ─── E' [main / HEAD]
                         (Copies are created here)
```

- Result: `main` gets the changes (`D'` and `E'`).
- The Catch: The `feature` branch did not move. It still points to the old `D` and `E`. You now have identical changes living in two separate places.

---

## Scenario 2: Using `git rebase --onto`

To use rebase, you stand on `feature`. It cuts the commits, pastes them on `main`, and drags the branch label with it to keep history perfectly clean.

```text
STEP 1: You must be on the branch you want to move
git checkout feature

STEP 2: Run the command (Take everything after C, and put it onto main)
git rebase --onto main C
```

The Visual Result:

```text
              (Original D & E are abandoned/deleted)
               x ─── x
             /
A ─── B ─── C ─── D' ─── E' [feature / HEAD]
            ▲               (The branch label was dragged here)
         [main] 
```

- Result: `feature` is physically moved to the tip of `main`.
- The Benefit: The old, original `D` and `E` commits are discarded by Git. There is no duplicate history left behind.

---

## 🧠 The Core Takeaway

- `cherry-pick` is a copy-paste tool for specific commits. The branch you copy _from_ doesn't change.
- `rebase --onto` is a cut-paste tool for entire branches. The branch you are on gets re-anchored to a new spot.

Does seeing how the `feature` label stays behind vs. moves forward make the difference clear?

# Multiple github profiles
```
Cause: You have two gh accounts (strangenguyen active, strangedef owner of this repo). The gh auth git-credential helper only ever uses the active account, so pushes went out as strangenguyen → 403 denied.
Fix (repo-only, via git config --local):
- credential.https://github.com.username=strangedef — keeps the gh helper silent for this repo
- credential.helper=!~/.local/bin/git-credential-strangedef — a small helper that reads strangedef's token from the macOS keychain on demand (no token stored in plaintext)
Other repos keep using strangenguyen; only this vault pushes as strangedef. All 25 commits are now on origin/main.
```