# skills

A personal collection of skills for AI agents (Claude and Codex). These are the skills I use day to day, kept in one place so I can sync them between machines. Feel free to browse or borrow, but expect them to be tailored to my own workflow.

## Skills

| Skill | Description | Source |
| --- | --- | --- |
| [lab-deploy](skills/lab-deploy/SKILL.md) | Integrates an existing app repo into the lab deploy pipeline so it builds on every push to `main` and is published to lab.fabian-kleine.dev. | Original (uses [lab-deploy](https://github.com/Fabian-Kleine/lab-deploy)) |
| [migrate-radix-to-base](skills/migrate-radix-to-base/SKILL.md) | Migrates React projects and components from Radix UI to Base UI, for single components or whole projects. | [shadcn/ui](https://github.com/shadcn/ui) |
| [shadcn](skills/shadcn/SKILL.md) | Manages shadcn/ui components and projects: adding, searching, debugging, styling and composing UI. | [shadcn/ui](https://github.com/shadcn/ui) |
| [skills-pull](skills/skills-pull/SKILL.md) | Syncs skills from this repo down to the local skills folder. Adds new skills, asks before overwriting changed ones, and backs up anything it replaces. | Original |
| [skills-push](skills/skills-push/SKILL.md) | Syncs new or changed local skills up to this repo. Always asks which skills to sync and checks them for secrets before pushing. | Original |

The Source column shows where a skill comes from. "Original" means I wrote it. Skills taken from other authors are listed with a link to their upstream repository.
