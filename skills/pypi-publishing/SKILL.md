---
name: pypi-publishing
description: Set up and run PyPI publishing for a Python project with a GitHub Actions release workflow (manual, tag-checked, PyPI trusted publishing through a protected "pypi" environment, no API tokens) plus CI, CHANGELOG and release notes. Use when asked to publish or release a package to PyPI, prepare a project for PyPI, add a release workflow, bump a version and cut a release, or pick a PyPI package name.
---

# PyPI publishing

Publishing is a **manual GitHub Actions run** on a tag, using **PyPI trusted publishing** (OIDC) through a GitHub
environment named `pypi`. No PyPI token is stored anywhere. The workflow refuses to run unless the tag matches the
package version, runs the checks, builds, publishes, then creates the GitHub release with the matching CHANGELOG notes.

Templates are in `templates/` (copy them, replace `{{COMMAND}}` and `{{DATE}}`):

| Template | Goes to |
|---|---|
| `.github/workflows/release.yml` | same path. Manual `workflow_dispatch` with a `tag` input |
| `.github/workflows/ci.yml` | same path. Tests on 3 OSes and the lowest and newest Python, plus a clean-install smoke test of the built wheel |
| `scripts/release_notes.py` | same path. Prints the `CHANGELOG.md` section for a tag |
| `CHANGELOG.md` | same path. `## [x.y.z] - YYYY-MM-DD` sections, newest first |

Action versions are pinned by commit SHA with the version in a comment. Keep it that way when updating them.

## 1. Pick the name

Check it is free **before** writing anything that uses it. PyPI treats `-`, `_` and `.` as the same and ignores case,
and it **rejects names "too similar" to an existing project**, which in practice means the name with all separators
removed also has to be free (`agent-crew` is refused because `agentcrew` exists). Check both forms:

```bash
for n in <name> <name-without-separators>; do curl -s -o /dev/null -w "$n %{http_code}\n" https://pypi.org/pypi/$n/json; done   # 404 = free, 200 = taken
```

Prefer a name that says what the project does and does not imitate a vendor's brand (`claude-*`, `openai-*`).
A free name can still be taken before the first upload: register the trusted publisher (step 3) promptly.

## 2. Prepare the project

- `pyproject.toml`: `name`, `version`, `description`, `readme`, `license = "MIT"` (or the project's), `license-files`,
  `authors`, `requires-python`, `[project.urls]`, `[project.scripts]` for a CLI, a build backend (`hatchling` or
  `uv_build`).
- A `LICENSE` file, and a `.gitignore` (`.venv/`, `dist/`, `__pycache__/`, caches).
- **Everything the program reads at runtime must be inside the wheel.** Files that live outside the package directory
  (templates, agent files, data) are not shipped. With hatchling use
  `[tool.hatch.build.targets.wheel.force-include]` to map them into the package, and have the code look there first.
- Copy the templates, fill the placeholders, and write the first `CHANGELOG.md` entry.
- Prove the wheel works away from the checkout:

```bash
uv build && uvx --from dist/*.whl <command> --help
```

## 3. One-time setup (the user does this; it needs their accounts)

1. **PyPI → Your projects → Publishing → add a (pending) trusted publisher**: owner, repository, workflow file name
   `release.yml`, environment `pypi`. For a name that does not exist yet, use the *pending* publisher form.
2. **GitHub → repository → Settings → Environments → New environment `pypi`**. Add a required reviewer (the owner)
   so every release needs a click.
3. Push to GitHub. Wait for CI to be green on `main`.

Tell the user exactly these steps with their real owner, repo and name filled in. Do not try to do them yourself.

## 4. Cut a release

1. Bump `version` in `pyproject.toml`; add the `## [x.y.z] - date` section to `CHANGELOG.md`; refresh the lockfile
   (`uv lock`). Commit and merge to `main`; CI must be green.
2. Tag and push: `git tag vX.Y.Z && git push origin vX.Y.Z`.
3. **Ask the user** before running the workflow: it publishes to a public index and PyPI versions cannot be reused or
   truly deleted. Then `gh workflow run release.yml -f tag=vX.Y.Z` (or Actions → Release → Run workflow), and approve
   the `pypi` environment prompt.
4. Verify: `uvx <package>==X.Y.Z --help` from a clean directory, and check the GitHub release exists with the notes.

A failed run before the publish step can simply be re-run. If a bad version was published, **yank** it on PyPI and
release a new patch version; never try to re-upload the same version.

## Rules

- Never put a PyPI token, password or API key in the repo, a workflow or a chat. Trusted publishing replaces them.
- Never run the release workflow, push a tag or publish without the user's explicit go-ahead for that version.
- Keep `release.yml` manual. The tag/version check and the environment approval are the safety net.
