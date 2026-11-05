---
name: skills-push
description: Sync new or changed local skills up to the remote GitHub repo Fabian-Kleine/skills. Use whenever the user says push skills, publish my skill, upload skills, sync skills to remote, back up my skills, or has just created or edited a skill and wants it in their repo. Always asks which skills to sync before pushing anything.
---

# skills-push

Publish local skills to https://github.com/Fabian-Kleine/skills.

The remote stores one folder per skill at `skills/<skill-name>/SKILL.md` (plus any bundled files). The skills folder is shared with third-party skills installed via `npx skills`, and the repo may be public, so this skill never pushes anything the user has not explicitly picked in this run.

## Where things live

- Local skills folder: first existing of `~/.agents/skills` then `~/.claude/skills`. If neither exists, ask the user.
- Disposable cache clone of the remote: `~/.cache/skills-sync/skills` (shared with skills-pull).
- Install records: `~/.agents/.skill-lock.json`, if present. Its `skills.<name>.source` field names the upstream repo of a third-party skill. Read it only to label skills; never modify it.

Use `git` and plain file copies so the steps work in bash and in PowerShell. On Windows, `~` means the user profile folder.

## Steps

1. **Refresh the cache.** Clone if missing (`git clone --depth 1 https://github.com/Fabian-Kleine/skills <cache>`), otherwise `git -C <cache> fetch --depth 1 origin` then `git -C <cache> reset --hard origin/HEAD`. The cache is rebuilt from local skills on every run, so discarding old state is safe.

2. **Build a status list of local skills.** For every local folder containing a `SKILL.md`, compare it with `<cache>/skills/<name>` using `git diff --no-index --quiet`:
   - `new`: not in the remote.
   - `modified`: in the remote with different contents.
   - `unchanged`: identical. Nothing to sync.
   Ignore `.git`, `node_modules`, `__pycache__` and `.DS_Store` when comparing and copying.
   Label each skill `yours` or `third-party (<source>)` using the lock file.

3. **Always ask which skills to sync.** This is required even when only one skill changed and even when the user already named skills in their request; the named ones just become the recommended first options. Use AskUserQuestion with multiSelect.
   - Offer only `new` and `modified` skills. Mention unchanged ones in the question text so the user knows they were skipped.
   - Put the status and origin in each option description. Do not preselect third-party skills, and note that republishing someone else's skill may have licensing implications.
   - AskUserQuestion allows 4 options per question and 4 questions per call. If there are more candidates, ask in plain text with a numbered list instead and let the user reply with numbers or names.
   - If nothing is new or modified, report that everything is already in sync and stop.

4. **Check the picked skills for secrets** before copying: look for `.env` files, private key blocks, and strings that look like API keys or tokens (`ghp_`, `sk-`, `AKIA`, `xox`, long `password=` or `token=` assignments). If anything matches, show the file and line (not the full secret) and ask whether to continue, skip that skill, or stop.

5. **Copy the picked skills into the cache** at `<cache>/skills/<name>`. Remove the existing folder in the cache first so files deleted locally disappear remotely. This skill only adds and updates; deleting a skill from the remote happens only if the user asks for it explicitly.

6. **Commit.** Run `git -C <cache> add -A` and check `git -C <cache> status --porcelain`. If empty, report nothing to push. Otherwise commit with a clear message such as `Add skill: <name>` or `Update skills: <a>, <b>`. If git has no `user.name` / `user.email` configured, ask the user rather than inventing one.

7. **Push** with `git -C <cache> push origin HEAD`.
   - If rejected as non-fast-forward, run `git -C <cache> pull --rebase origin HEAD` once and push again.
   - If authentication fails, tell the user to sign in (for example `gh auth login`, or a credential manager prompt) and stop. Do not try other transports or workarounds.

8. **Report** in a few lines: which skills were pushed (new vs updated), the commit hash, and the repo link.
