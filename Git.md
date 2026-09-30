# Git

## Simple idea
Git = a **time machine + collaboration tool** for code. Every commit = a snapshot with an ID.

```
Working Directory  ──git add──►  Staging Area  ──git commit──►  Local Repo  ──git push──►  Remote (GitHub)
   (your edits)                (next snapshot)                (history)                   (team shares)
                                                              ◄──git pull / fetch────────
```

## Daily flow
```bash
git clone https://github.com/org/shop.git
git checkout -b feature/PROJ-101-place-order     # new branch from main/dev
# ...edit code...
git status                     # what changed
git add src/                   # stage
git commit -m "PROJ-101: add order API"
git push -u origin feature/PROJ-101-place-order
# open Pull Request → review → CI passes → merge
git checkout main && git pull  # sync
git branch -d feature/PROJ-101-place-order
```

## Branching strategy (Git Flow / trunk-based)
```
main     ●─────────────●──────────────●────── (production, tagged v1.0, v1.1)
          \           / \            /
dev        ●───●───●─●   ●──●───●──●            (integration)
            \     /        \    /
feature      ●───●          ●──●                (one per story)
hotfix                ●───────────► merged to main AND dev
```

## Must-know commands & differences
| Question | Answer |
|---|---|
| **merge vs rebase** | merge keeps history + creates merge commit; rebase replays your commits on top of target → linear history. Never rebase **shared/pushed** branches. |
| **fetch vs pull** | fetch downloads only; pull = fetch + merge. |
| **reset vs revert** | `reset` moves branch pointer (rewrites history, local use); `revert` creates a NEW commit that undoes (safe for shared branches). |
| **`--soft / --mixed / --hard`** | keep changes staged / unstaged / discard all |
| **stash** | temporarily shelve uncommitted work: `git stash`, `git stash pop` |
| **cherry-pick** | copy one commit to another branch: `git cherry-pick <sha>` |
| **squash** | combine many commits into one on merge |
| **.gitignore** | files never tracked (target/, node_modules/, .env) |
| **fork vs clone** | fork = your server-side copy of someone's repo |
| **tag** | fixed label on a commit (release) |

## Resolving a merge conflict (flow)
```
git merge dev  →  CONFLICT in OrderService.java
   ▼
Open file, find markers:
<<<<<<< HEAD          (your version)
   discount = 10;
=======
   discount = 15;     (incoming version)
>>>>>>> dev
   ▼
Decide final code, remove markers → git add file → git commit (or git rebase --continue)
```
Useful: `git log --oneline --graph`, `git diff`, `git blame file`, `git reflog` (recover "lost" commits), `git bisect` (binary-search the commit that introduced a bug).
Commit messages: imperative, reference ticket. Small, focused commits. Never commit secrets (if leaked: rotate the secret, then clean history).
