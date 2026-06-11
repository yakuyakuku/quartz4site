# Why Git Gets Messy with Obsidian Auto-Backups

This repo (Quartz4) uses the Obsidian Git plugin to auto-commit and auto-push changes. It's convenient, but it has some sharp edges — especially when you're also committing manually or pulling from other machines.

## What Happened Today

Your git was in this state:

```
branch: main
local:  1 commit ahead of origin/main
remote: 10 commits ahead of your fork point
status: diverged, with unmerged paths
conflict: content/.obsidian/workspace.json
```

The Obsidian Git plugin was set to auto-commit and auto-push. Meanwhile, the remote had already received commits from another session (or a previous push). When the plugin tried to pull before pushing, it hit a merge conflict in `.obsidian/workspace.json` — a file that changes **every single time you open or interact with Obsidian**. Then the merge got stuck, and every subsequent auto-backup piled on top of the unresolved mess.

## Why `workspace.json` Is Always the Problem

`.obsidian/workspace.json` tracks:

- Which panes you have open
- Their sizes, positions, and splits
- Your active file
- Any workspace layout state

Every time you open a different file, rearrange a pane, or even click around — **it changes**. So when two sessions both modify it (e.g., your PC and a phone, or two separate Obsidian windows), a merge conflict is guaranteed.

**It should not be tracked by git.** But by default, Obsidian git plugins include everything in the vault unless explicitly ignored.

## How to Prevent This

### Option 1: Ignore `workspace.json` (Recommended)

Add this to `.gitignore` in the repo root:

```
content/.obsidian/workspace.json
content/.obsidian/workspace-mobile.json
content/.obsidian/cache/
content/.obsidian/graph.json
```

Or more broadly, ignore the entire `.obsidian/` config directory **except** `community-plugins.json` and plugin settings you explicitly want to version:

```
content/.obsidian/*
!content/.obsidian/community-plugins.json
```

After adding to `.gitignore`, remove the tracked files from git:

```bash
git rm --cached content/.obsidian/workspace.json
git rm --cached content/.obsidian/workspace-mobile.json
git commit -m "chore: stop tracking obsidian workspace files"
```

### Option 2: Disable Auto-Pull in Obsidian Git

In the Obsidian Git plugin settings, turn off "Pull on startup" or set it to manual only. This way, the plugin will only push your local auto-commits, and you handle the pull/merge yourself when convenient.

### Option 3: Use Rebase Instead of Merge

If you ever need to sync manually, prefer rebasing over merging:

```bash
git pull --rebase origin main
```

This keeps history linear and avoids merge commits. The downside: if `workspace.json` is still tracked, you'll still get conflicts during rebase. So Option 1 first.

## What We Did Today

1. Aborted the stuck merge (`git merge --abort`)
2. Rebased local commit on top of `origin/main` (`git rebase origin/main`)
3. Resolved the `workspace.json` conflict by taking the remote version (newer timestamp = more complete)
4. Force-pushed... wait, you said auto-pushed. Let me check what happened.

Actually, the auto-push happened (you said "it's auto pushed btw"). So your local commit is now on remote with the clean rebased history. Good.

## Dealing with Private Files

Files in `content/Novel/private/` are tracked by git. If you want them **not** tracked (local-only), add the whole `private/` folder to `.gitignore`:

```
content/Novel/private/
```

Then remove from tracking:

```bash
git rm --cached -r content/Novel/private/
git commit -m "chore: stop tracking private folder"
```

But if you *do* want them tracked (e.g., as a personal backup), that's fine — just be aware that auto-committing large files here will inflate your repo size over time.

## Summary

| Problem | Cause | Fix |
|---|---|---|
| Merge conflicts in `workspace.json` | Auto-generated Obsidian file changes constantly | Add to `.gitignore` |
| Diverged branches | Auto-commit + auto-push from multiple sources | Use `pull --rebase` or manual sync |
| Stuck merge state | Plugin tried to merge and failed | `git merge --abort` then rebase |
| Bloated history | Every vault backup = one commit | Set longer auto-commit interval or commit manually |

**Bottom line:** Ignore Obsidian workspace files, and git will mostly just work with auto-backups. If you want more control, reduce the auto-sync frequency or switch to manual sync.
