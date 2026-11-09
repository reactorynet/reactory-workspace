# reactory-workspace

Configuration-driven tooling for running a multi-repository platform as a
**multi-org** codebase: the platform lives in its upstream GitHub organisation
(`github.com/reactorynet`) and is mirrored into one or more enterprise
organisations. Changes can be committed to and pulled from any org, and
brought across in both directions by merge.

This repository is **enterprise-neutral**: nothing in it names a mirror
organisation. Each enterprise keeps its configuration in a repository of its
own, cloned into the gitignored `enterprises/<name>/` folder, the way the
Reactory server keeps modules under `src/modules/`.

| Document | Content |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | How the configuration drives the tool, repo composition, remotes, sync flow, bootstrap, adding another enterprise (Mermaid diagrams) |
| [docs/SECURITY.md](docs/SECURITY.md) | Trust boundaries, risk register, hardening the public upstream (audit findings, control chain), what the tool enforces vs. process controls |
| `enterprises/<name>/docs/` | Each enterprise's own decision records live in its configuration repo, not here |
| [AGENTS.md](AGENTS.md) | The short version for agents and new contributors: prerequisites, the rules (no pushes to default branches, merge methods, reviews, managed files), common tasks |

## Layout

| Path | Purpose |
|---|---|
| `repos.json` | **Platform inventory**: every git repository in a checkout (path, kind root / submodule / nested, parent, groups) and the upstream org. Enterprise-neutral. |
| `enterprises/<name>/` | **Enterprise configuration repos**, gitignored, one git repo each: `profile.json` (mirror org(s), protection, org-owned paths, mirrored groups, enterprise-native repos, provisioning), `lock.json`, provisioning templates, rendered fragments, docs. `enterprise add <git-url>` clones one; `enterprise init <name>` creates one from `templates/enterprise/`. |
| `policies/<name>.json` | **GitHub security policy** for an org: org settings, repo settings, branch protection, Actions permissions, baseline files. `public-upstream.json` is applied to reactorynet by `audit` / `harden` / `seed`. |
| `templates/enterprise/` | Skeleton of an enterprise configuration repo: commented `profile.json`, example provisioning template, README. |
| `templates/upstream/` | Baseline files `seed` adds to upstream repos: `SECURITY.md`, `CODEOWNERS`, `dependabot.yml`, CodeQL, dependency review, Scorecard (SHA-pinned). |
| `templates/github/sync-upstream.yaml` | Optional scheduled Action for the enterprise side that opens upstream-sync PRs. |
| `locks/upstream.lock.json` | SHA pin per repo when no enterprise is selected (with one, the lock is `enterprises/<name>/lock.json`). |
| `bin/reactory-repos` | The CLI. Python 3.9+, standard library only. |

The workspace root is `$REACTORY_HOME` (fallback: the parent of this folder),
the layout the Reactory installer produces:

```
$REACTORY_HOME/
  reactory-workspace/                 <- this repo
  reactory-express-server/            <- root; config/reactory is a submodule, src/modules/* are nested clones
  reactory-pwa-client/                <- root; src/components/plugins/reactory-core is a submodule
  reactory-core/  reactory-data/  reactory-workflow-es/ ...
```

## Selecting an enterprise

```bash
bin/reactory-repos enterprise add git@github.com:<org>/reactory-workspace-<name>.git   # clone into enterprises/<name>/
bin/reactory-repos enterprise list
bin/reactory-repos list                         # the only enterprise present is selected automatically
bin/reactory-repos --profile <name> list
REACTORY_PROFILE=<name> bin/reactory-repos status
bin/reactory-repos --profile none list          # upstream only
```

The examples below use `acme` for the enterprise org key; substitute yours.

## Remotes

Every repo gets one remote per org named by the org key (`reactorynet`,
`acme`, …) plus `origin`, which duplicates the **primary** org
(`primary_org` in the profile, `REACTORY_PRIMARY_ORG` to override). `origin`
keeps plain git, the platform's `bin/*.sh` scripts and relative submodule URLs
working; the named remotes are what every cross-org command uses.

## Everyday commands

```bash
bin/reactory-repos check                      # remotes, relative .gitmodules, gh auth, org membership, lock
bin/reactory-repos remotes                    # (re)create remotes in every present repo
bin/reactory-repos fetch                      # all orgs, all repos, in parallel
bin/reactory-repos status --fetch             # branch, dirty?, upstream<>mirror divergence, local vs origin
bin/reactory-repos exec -- git log --oneline -3
```

Limit any command with `--group platform|tenants|optional|native` or
`--repo <name>` (repeatable).

### Fresh machine

```bash
export REACTORY_HOME=~/Projects/reactory
git clone git@github.com:reactorynet/reactory-workspace.git $REACTORY_HOME/reactory-workspace
cd $REACTORY_HOME/reactory-workspace
bin/reactory-repos clone --group platform             # from the primary org, then wire all remotes
bin/reactory-repos clone --group platform --locked    # same, pinned to enterprises/<name>/lock.json
REACTORY_PRIMARY_ORG=acme bin/reactory-repos clone --group platform --group native   # from the mirror
```

The Reactory installer still works; `REACTORY_GITHUB_ORG=<mirror org>
bin/install.sh` clones from the mirror. Run `remotes` afterwards.

### Working on a change

```bash
bin/reactory-repos push --to acme --repo reactory-express-server -u        # feature branch -> mirror
bin/reactory-repos push --to acme --repo reactory-express-server -u --pr   # ... and open the PR, auto-merge (squash) armed
bin/reactory-repos push --to reactorynet --repo reactory-express-server --pr
```

`push` refuses a protected default branch in **either** org (reactorynet
`master` is protected by policy too); use a branch and a PR. With `--pr` the
tool opens the PR and arms GitHub **auto-merge**: the PR lands as a squash
commit (title and body from the PR) as soon as the required review and
checks pass, and the branch is deleted on merge. `--no-auto-merge` leaves
the merge to a human.

### Bringing the orgs level

```bash
bin/reactory-repos sync --to acme -n                   # dry run
bin/reactory-repos sync --to acme --pr                 # upstream -> sync/ branch -> PR on the mirror
bin/reactory-repos sync --to reactorynet --pr          # mirror -> sync/ branch -> PR on upstream, enterprise paths dropped
bin/reactory-repos sync --to all -n
```

`sync` merges in a temporary worktree, applies the org-specific-path rule
(target-owned paths: target's version wins, absent means removed;
source-owned paths never cross), commits, pushes. Merge commits are signed
with your key (both orgs require signatures; `check` verifies the setup).
Conflicts abort unless `--keep-worktree`. Sync PRs are armed with auto-merge
using a **merge commit**, never squash or rebase: the two histories must keep
a common ancestor. Squash is allowed on the repos for ordinary feature PRs, so
the sync PR body says which method to use if someone merges it by hand.

### First import into a newly created org repo

```bash
bin/reactory-repos provision --to acme --out auto          # repo definitions -> enterprises/<name>/rbac/
bin/reactory-repos bootstrap --to acme --group platform --pr
```

### Hardening an org's GitHub settings

```bash
bin/reactory-repos audit --to reactorynet                 # read-only drift report against policies/public-upstream.json
bin/reactory-repos audit --to reactorynet -v --json out.json
bin/reactory-repos harden --to reactorynet -n             # what would change (repo level)
bin/reactory-repos harden --to reactorynet                # apply: scanning, Dependabot, Actions policy, branch protection
bin/reactory-repos harden --to reactorynet --org-level    # + org settings (needs: gh auth refresh -s admin:org)
bin/reactory-repos seed --to reactorynet -n               # which baseline files each repo lacks
bin/reactory-repos seed --to reactorynet --pr             # push security/baseline-<stamp> branches and open PRs
```

Read [docs/SECURITY.md §3](docs/SECURITY.md#3-hardening-the-public-upstream)
first: the policy removes administrator self-merge and direct pushes.

### Merge hygiene and branch cleanup

Both orgs' repo definitions (the policy for reactorynet, the enterprise's
provisioning template for its mirrors) set `allow_auto_merge`, `delete_branch_on_merge`, squash
merges with the PR title/body as the commit, merge commits kept for sync, and
rebase off. From then on every merge removes its own branch. `cleanup` deals
with what landed before that was on:

```bash
bin/reactory-repos cleanup --to reactorynet -n            # which remote branches would go, and why
bin/reactory-repos cleanup --to all                       # delete: heads of merged PRs + merged sync/ security/ import/ branches
bin/reactory-repos cleanup --to all --all-merged --local  # any branch already in the default branch; matching local branches too
```

Never deleted: the default branch, any branch with an open PR, and a branch
that received commits after its PR was merged. Dry-run on 2026-11-09 listed
over a hundred merged-PR branches still present across the reactorynet repos.

### Pinning

```bash
bin/reactory-repos lock                 # write enterprises/<name>/lock.json from current HEADs
bin/reactory-repos checkout-lock        # detach every repo at its locked SHA
```

### Folders that are not repositories yet

`bin/reactory-repos init --repo <name>` turns a plain folder listed in the
profile into a repo with an initial commit; `bootstrap --to acme` pushes it
once the org repo exists. Scan for secrets first (see SECURITY.md R4).

## Adding another enterprise

`bin/reactory-repos enterprise init <name>` creates `enterprises/<name>/`
from `templates/enterprise/` as a new git repo. Fill in the org, org-specific
paths, included groups, native repos and provisioning template, push it to
the enterprise's organisation, then `bin/reactory-repos --profile <name>
list`. Steps and template variables are
in [docs/ARCHITECTURE.md §6](docs/ARCHITECTURE.md#6-adding-another-enterprise).
