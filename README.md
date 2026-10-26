# pypi-publishing

Publish a Python package to PyPI the safe way, and have your AI agent set it up for you. The skill adds a manual
GitHub Actions release workflow that uses **PyPI trusted publishing** (no API tokens anywhere), CI, a changelog and
release notes, and walks the agent through preparing the project, the one-time setup and each release.

It is an [Agent Skill](https://agentskills.io) (a `SKILL.md` plus workflow templates), so it works in Claude Code,
Cursor and any other agent that loads skills.

## How it works

| Step | What happens |
|---|---|
| **Name** | Check the PyPI name is free before anything uses it |
| **Prepare** | Complete `pyproject.toml` metadata, license, `.gitignore`; make sure runtime files are bundled in the wheel; add the templates; smoke-test the built wheel |
| **One-time setup** | You add the trusted publisher on PyPI. The agent creates/verifies the protected GitHub `pypi` environment through authenticated `gh`; otherwise it gives manual steps |
| **Release** | Bump the version and changelog, tag, then run the release workflow (after your go-ahead) and approve the environment prompt |

The release workflow (`release.yml`) is manual on purpose. It checks out the tag, **refuses to continue if the tag does
not match the package version**, runs the checks, builds, publishes through OIDC, then creates the GitHub release with
the matching `CHANGELOG.md` section and the built files attached. The `pypi` environment's required reviewer is the
last gate.

What the templates give you:

- `release.yml`: manual release as above
- `ci.yml`: tests on Linux, macOS and Windows with the lowest and newest Python, plus a clean install of the built wheel
- `scripts/release_notes.py`: prints a changelog section for a tag
- `CHANGELOG.md`: the format `release_notes.py` expects

The agent never publishes, tags or pushes without your explicit go-ahead for that version.

## Install

### Any agent: the `skills` CLI

```bash
npx skills add illescasDaniel/pypi-publishing-skill
```

It asks which agents to install for and whether to install into the current project or globally. Non-interactive:
`npx skills add illescasDaniel/pypi-publishing-skill -a claude-code -y`, plus `-g` for global.

### By hand

Copy the `skills/pypi-publishing` folder into your agent's skills directory.

| | Project (shared with your team) | All your projects |
|---|---|---|
| Codex | `.codex/skills/pypi-publishing/` | `~/.codex/skills/pypi-publishing/` |
| Claude Code | `.claude/skills/pypi-publishing/` | `~/.claude/skills/pypi-publishing/` |
| Cursor | `.cursor/skills/pypi-publishing/` | `~/.cursor/skills/pypi-publishing/` |

## Contributing

Issues and pull requests are welcome. The skill is deliberately short: if a change makes `SKILL.md` longer, it should
earn it.

## License

MIT, see [LICENSE](LICENSE).
