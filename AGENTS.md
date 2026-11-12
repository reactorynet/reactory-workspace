# Working in reactory-workspace (for agents and people)

This repository is the control plane for the Reactory multi-repo platform:
an inventory of every repository (`repos.json`), per-enterprise configuration
repos that mirror the platform into other GitHub organisations (cloned into
the gitignored `enterprises/<name>/`, never committed here), the GitHub
security policy for the public upstream (`policies/`), the baseline files
that policy seeds into each repo (`templates/upstream/`), and one CLI,
`bin/reactory-repos`, that does all of the work. Read `README.md` for the
commands and `docs/` for the reasoning. This file is the short version an
agent needs before touching anything.

## Prerequisites

- Python 3.9+ and git. The CLI uses the standard library only; there is
  nothing to install.
- `gh` authenticated as a member of the organisation(s) you work with
  (`gh auth status`). Repo-level commands need the `repo` scope;
  `harden --org-level` needs `admin:org`.
- The checkout layout the Reactory installer produces, rooted at
  `$REACTORY_HOME` (default: the parent folder of this repo):
  `reactory-express-server/`, `reactory-pwa-client/`, the satellite repos and
  this repo side by side. `bin/reactory-repos check` tells you what is
  missing.
- The enterprise you work for, cloned with `bin/reactory-repos enterprise add
  <git-url>` into `enterprises/<name>/`. Without one, the tool sees the
  upstream organisation only.
- Commit signing configured; both organisations require signed commits.

## Start here

```bash
bin/reactory-repos check                 # remotes, gh auth, org membership, lock
bin/reactory-repos status --fetch        # per repo: branch, dirty, upstream<>mirror divergence
bin/reactory-repos list -v               # what the configuration says exists where
```

Every command accepts `--repo <name>` (repeatable) and `--group <group>`.
Everything that writes to GitHub or to a checkout has a dry run: `-n`.
**Run the dry run first and read it**, then run for real.

## Rules

1. **Never push to a default branch.** `master` in reactorynet and `main` in
   the mirrors are protected by policy. Use a branch and `push --to <org>
   --pr`, `sync --to <org> --pr`, `seed --to <org> --pr`. The tool refuses a
   direct push to a protected default branch; do not work around it with
   plain git.
2. **Merge methods are not interchangeable.** Feature and seed PRs squash.
   Sync and import PRs (`sync/*`, `import/*` branches) must land as a
   **merge commit**, or the two organisations lose their common ancestor and
   every later sync conflicts. The tool arms GitHub auto-merge with the right
   method; if a PR has to be merged by hand, the PR body says which.
3. **Every PR needs a review from someone else.** Branch protection enforces
   one code-owner approval with administrators included. An agent cannot
   approve its own PR. Open the PR, report the URL, stop.
4. **Seeded files belong to the template.** Files marked "Managed by
   reactory-workspace `seed`" in a platform repo are overwritten by the next
   seed. Change `templates/upstream/` here and re-seed; never edit the copy.
   `CODEOWNERS` and `SECURITY.md` are create-only and may be edited in place.
5. **This repository names no enterprise.** Nothing in `bin/reactory-repos`,
   the docs or the templates names a mirror organisation, its teams, tickets
   or people. Everything enterprise-specific belongs in that enterprise's
   configuration repo under `enterprises/<name>/`, which is gitignored here.
   Adding a repo means an entry in `repos.json`; adding an enterprise means
   `enterprise init <name>` and a repo in that enterprise's organisation.
   Before committing, `git grep -i <enterprise name>` must come back empty.
6. **Cleanup and harden change shared state.** `cleanup` deletes remote
   branches, `harden` rewrites repo settings and branch protection for a
   whole organisation. Dry-run, show the user the output, and get an explicit
   go-ahead before dropping `-n`.
7. **Secrets never enter this repo.** No tokens, no `.env` files, no lock
   files from checkouts that contain credentials. `gh` holds the credential.

## Common tasks

| Task | Command |
|---|---|
| Fresh machine | `enterprise add <git-url>`, `clone --group platform`, `remotes` |
| Bring local checkouts up to date | `pull -n`, then `pull` (fast-forward only; uncommitted changes are stashed and restored) |
| Feature branch to the mirror with a PR | `push --to <org> --repo <r> -u --pr` |
| Bring the mirror level with upstream | `sync --to <org> -n`, then `sync --to <org> --pr` |
| Bring upstream level with the mirror | `sync --to reactorynet -n`, then `--pr` |
| Check / apply the GitHub security policy | `audit --to reactorynet`, `harden --to reactorynet -n`, `harden --to reactorynet` |
| Add missing baseline files to repos | `seed --to reactorynet -n`, then `seed --to reactorynet --pr` |
| Remove branches that already landed | `cleanup --to all -n`, then `cleanup --to all` |
| Pin / restore exact SHAs | `lock`, `checkout-lock`, `clone --locked` |
| Render repo definitions for the mirror's repo system | `provision --to <org> --out auto` |

## Changing this repository

- `bin/reactory-repos` is one file; keep it standard-library only and keep
  the `-n` path working for every mutating command.
- Mermaid diagrams in `docs/` must still render; a quick check is a local
  `python3 -m http.server` and the Mermaid CDN page used before.
- Record anything applied to a live organisation (dates, what changed, what
  is still pending) in `docs/SECURITY.md` under "Applied".
- Open a PR; this repository follows the same policy it enforces.
