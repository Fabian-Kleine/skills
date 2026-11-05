---
name: skills-pull
description: Sync skills from the remote GitHub repo Fabian-Kleine/skills down into the local skills folder. Use whenever the user says pull skills, sync skills from remote, update my local skills, download my skills, get my skills onto this machine, or is setting up a new computer and wants their skills back, even if they never name the repo.
---

# skills-pull

Bring skills from the remote repo https://github.com/Fabian-Kleine/skills down to this machine.

The remote stores one folder per skill at `skills/<skill-name>/SKILL.md` (plus any bundled files). If the repo has no `skills/` folder, fall back to top-level folders that contain a `SKILL.md`.

## Where things live

- Local skills folder: use the first one that exists of `~/.agents/skills` then `~/.claude/skills`. If neither exists, ask the user where their skills live before doing anything.
- Disposable cache clone of the remote: `~/.cache/skills-sync/skills`. It only mirrors the remote, so it is safe to reset. Both sync skills share it.
- Backups of anything overwritten: `~/.cache/skills-sync/backup/<timestamp>/<skill-name>`.

Use `git` and plain file copies so the steps work in bash and in PowerShell. On Windows, `~` means the user profile folder.

## Steps

1. **Refresh the cache.**
   - If the cache does not exist: `git clone --depth 1 https://github.com/Fabian-Kleine/skills <cache>`.
   - Otherwise: `git -C <cache> fetch --depth 1 origin` then `git -C <cache> reset --hard origin/HEAD`. Resetting is fine here because the cache holds no user work.

2. **List remote skills**: every folder under `<cache>/skills/` that contains a `SKILL.md`.

3. **Classify each remote skill against the local folder**:
   - `new`: no local folder of that name.
   - `identical`: same contents. Compare with `git diff --no-index --quiet <remote-dir> <local-dir>` (exit code 0 means identical).
   - `different`: both exist but contents differ.

4. **Copy the new ones** straight into the local skills folder. No question needed, since nothing is being replaced.

5. **Handle the different ones carefully.** A local copy may hold edits that were never pushed, so replacing it silently could lose work.
   - Show a short summary per skill (`git diff --no-index --stat`) so the user can judge which side is newer.
   - Ask which of them to overwrite (AskUserQuestion with multiSelect, up to 4 options per question; if there are more, ask in plain text with a numbered list). Do not preselect anything.
   - For each one the user picks, first copy the existing local folder to the backup location, then replace the local folder with the remote version (remove the old folder first so files deleted upstream disappear too).
   - Leave the rest untouched.

6. **Never delete** local skills that are missing from the remote, and never touch `.skill-lock.json`. Third-party skills installed with `npx skills` live in the same folder and must survive a pull.

7. **Report** in a few lines: added, updated, kept local (skipped), unchanged, and where backups went if any. Mention that newly added skills are usually picked up in a fresh session.

## Failure handling

- If the clone or fetch fails (network, auth), say so plainly and stop. Do not try mirrors or other download routes.
- If the remote has no skills yet, say so and stop.
