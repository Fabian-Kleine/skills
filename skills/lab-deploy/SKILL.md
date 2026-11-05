---
name: lab-deploy
description: Integrate an existing app repo into Fabian-Kleine/lab-deploy so it builds on every push to main and publishes to https://lab.fabian-kleine.dev/<slug>/. Use whenever the user says add this to the lab, deploy to the lab, set up lab deploy, publish this app on lab.fabian-kleine.dev, or wants a Vite or static project to appear on their lab overview page, even if they never name the lab-deploy repo.
---

# lab-deploy

Wire the app in the current repo to the lab deploy pipeline from https://github.com/Fabian-Kleine/lab-deploy. The pipeline is a GitHub Actions workflow (`.github/workflows/lab.yml`) rendered from a template. It builds the app with a base path of `/<slug>/`, rsyncs it to the lab server, and the app then shows up on the overview page. The workflow is inlined into each repo on purpose, so the app repo does not depend on the (private) lab-deploy repo at runtime.

Static output only: Vite apps, other apps that can build to plain files, or plain HTML repos. Anything that needs a server process does not fit.

## Prerequisites to check first

- The current directory is inside a git repo whose `origin` is on GitHub. Get `owner/repo` from `gh repo view --json nameWithOwner -q .nameWithOwner` (fallback: parse `git remote get-url origin`).
- `gh` is installed and authenticated (`gh auth status`). It is needed for secrets and for fetching the template.
- Python 3 is available (`python3`, or `python` on Windows).

If one of these is missing, say which and stop.

## Steps

1. **Check for an existing setup.** If `.github/workflows/lab.yml` already exists, tell the user and ask whether to re-render it (replace) or leave it. Read its `env` block first so slug, name and description are kept on a re-render.

2. **Get the template.** Keep a shallow cache clone at `~/.cache/lab-deploy`: clone `https://github.com/Fabian-Kleine/lab-deploy` with `gh repo clone Fabian-Kleine/lab-deploy ~/.cache/lab-deploy -- --depth 1`, or when it exists, `git -C ~/.cache/lab-deploy pull --ff-only`. `template/render.py` and `template/lab.yml` must stay together.

3. **Inspect the app** so the pipeline fits it:
   - `package.json` present? Read the `build` script. The default build runs `<pm> run build -- --base=$LAB_BASE`, so it only works when `build` ends in `vite build` (for example `tsc && vite build` is fine). For Next.js, Astro, CRA or anything else, pick a custom build command instead and a matching output folder.
   - Output folder. The default is `dist`. If the build writes elsewhere (check `build.outDir` in the Vite config), it must be set as `LAB_OUT`.
   - No `package.json`: the repo is published as plain static files (tracked files only, excluding `README.md` and `LICENSE`). It needs an `index.html` at the top level.
   - Package manager comes from the lockfile (pnpm, yarn, npm). Nothing to configure, but mention it.

4. **Choose slug, name and description.** Defaults: slug is the lowercased repo name (allowed pattern `^[a-z0-9][a-z0-9._-]{0,62}$`), name is the slug, description is the GitHub repo description. Propose these and ask the user to confirm or change them (AskUserQuestion with the defaults as the first option; the user can type their own). Then check for slug collisions: try `curl -fsS https://lab.fabian-kleine.dev/_dirs/` (JSON listing of existing folders). If the slug already exists and this repo is not the one that owns it, warn that a deploy would replace that folder and ask for another slug. If the lab is unreachable, say the check was skipped.

5. **Render the workflow.**
   ```
   mkdir -p .github/workflows
   python3 ~/.cache/lab-deploy/template/render.py "<slug>" "<name>" "<build command or empty>" > .github/workflows/lab.yml
   ```
   Pass an empty string for the build command to use the default. `render.py` has no flags for the description or output folder, so afterwards edit the `env:` block in the generated file: set `LAB_DESCRIPTION` if the user wants one and `LAB_OUT` if the output folder is not `dist`. Only the `env` block should ever be edited. Keep the rest of the template as rendered so later template updates can be re-applied cleanly.

6. **Make the app work under a sub-path.** The app is served at `/<slug>/`, not at `/`. Vite applies `--base` to `index.html` and bundled assets, but not to everything:
   - Search the source for root-relative URLs in code (for example `"/logo.png"`, `fetch("/data.json")`, CSS `url(/...)`) and for router setup. Use `import.meta.env.BASE_URL` for static files, and a router base such as `<BrowserRouter basename={import.meta.env.BASE_URL}>` (React Router) or the equivalent. Hash routing needs no change. The server already falls back to `/<slug>/index.html` for unknown paths, so client-side routing works.
   - Show the user the findings and the proposed edits, and apply them only after they agree. These touch app source, unlike the workflow file.

7. **Verify the build locally** the way CI will run it. If dependencies are not installed, tell the user and install them with the detected package manager. Run the build with `--base=/<slug>/` (or the custom command with `LAB_BASE=/<slug>/` set), then replay the CI asset check: every `src="/..."` and `href="/..."` in `<out>/index.html` must start with `/<slug>/`. Report failures with the offending lines. Remove the build output afterwards if it is not gitignored.

8. **Secrets.** The workflow needs repo secrets `LAB_SSH_KEY` and `LAB_KNOWN_HOSTS`.
   - Run `gh secret list -R <owner/repo>`. If both exist, skip this step.
   - Otherwise the key files come from the lab server setup (`/root/lab/labdeploy_ed25519` and `/root/lab/lab_known_hosts` on the server, see `server/install.sh`). If they exist on this machine, set them with `gh secret set LAB_SSH_KEY -R <owner/repo> < <path>` and likewise for `LAB_KNOWN_HOSTS`. If not, ask the user where the files are, or give them the two commands to run where the files are. Never print the private key, never copy it into the repo, and never commit it.

9. **Commit and push only with the user's go-ahead.** Pushing to `main` triggers the first deploy, so ask before committing. Stage only `.github/workflows/lab.yml` and any source edits from step 6, commit with a message like `Deploy to lab`, and push. Then offer to follow the run with `gh run watch`. A "Screenshot" step runs after the deploy and is allowed to fail without failing the deploy.

10. **Report** in a few lines: the live URL `https://lab.fabian-kleine.dev/<slug>/`, what was added or changed, whether secrets were set, and anything left for the user to do. The overview card can be refined by adding a `lab.json` (`{"name": "...", "description": "..."}`), but the workflow already writes one on every deploy.

## Notes

- The deploy user can only rsync into a staging folder and run `publish <slug>` or `publish-overview`. There is nothing else to configure on the server for a new app.
- Deployments from the same repo cancel each other (concurrency group `lab`), so the latest push wins.
- Do not hand-edit the template parts of the workflow. To change them, change the template in the lab-deploy repo and re-render.
