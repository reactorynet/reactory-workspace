# Architecture: configuration-driven multi-org workspace

This document explains how `reactory-workspace` turns two JSON files into a
working multi-organisation checkout, and how to reuse it for another
enterprise. Diagrams are Mermaid and render on GitHub.

## 1. Configuration model

```mermaid
flowchart LR
    subgraph ent["enterprises/&lt;name&gt;/ (gitignored: one nested git repo per enterprise)"]
        PROF["profile.json<br/>• mirror org(s) + protection<br/>• include_groups / include_repos / exclude_repos<br/>• org_specific_paths<br/>• extra_repos (enterprise-native)<br/>• provisioning template + vars"]
        TPL["templates/*.yaml<br/>repo-definition template"]
        LOCK["lock.json<br/>pinned SHA per repo"]
        FRAG["rbac/*.fragment<br/>rendered definitions"]
    end

    subgraph config["Configuration (committed, enterprise-neutral)"]
        INV["repos.json<br/>platform inventory<br/>• upstream org<br/>• groups<br/>• 32 repos: path, kind, parent, groups"]
        TPLE["templates/enterprise/<br/>skeleton for `enterprise init`"]
        POL["policies/&lt;name&gt;.json<br/>GitHub security policy per org<br/>• org settings • repo settings<br/>• branch protection • Actions<br/>• baseline files (templates/upstream)"]
    end

    ENV["environment<br/>REACTORY_HOME · REACTORY_PROFILE · REACTORY_PRIMARY_ORG"]
    TPLE -->|enterprise init| PROF

    subgraph runtime["bin/reactory-repos (stdlib Python)"]
        WS["Workspace model<br/>orgs = upstream + profile.orgs<br/>repos = inventory ∩ (include_groups ∪ include_repos)<br/>        + extra_repos"]
        CMD["commands<br/>enterprise · list · clone · remotes · fetch · status<br/>push · sync · bootstrap · init<br/>lock · checkout-lock · check · provision<br/>audit · harden · seed · cleanup"]
    end

    INV --> WS
    PROF --> WS
    ENV --> WS
    TPL --> CMD
    POL --> CMD
    WS --> CMD
    CMD -->|gh api: audit / harden| GH["GitHub org + repo settings"]
    CMD -->|writes| LOCK
    CMD -->|provision --out auto| FRAG
    LOCK -->|clone --locked<br/>checkout-lock| CMD
    CMD -->|git / gh| FS["$REACTORY_HOME/* checkouts"]
```

Rules the model applies:

- An inventory repo is **mirrored** into every org of the active profile when
  one of its `groups` is in `include_groups`, or its name is in
  `include_repos`, and it is not in `exclude_repos`. Otherwise it is
  upstream-only (the reactorynet tenant configs, for example). `include_repos`
  takes a single repo from a group that is not mirrored as a whole; a name
  missing from the inventory in either list is an error.
- A profile's `extra_repos` are **enterprise-native**: they exist only in the
  named org. `no_repo: true` marks a folder that has no remote anywhere yet.
- Per-repo `orgs` and `branches` maps remain available for exceptions
  (different name in the mirror, non-default branch).
- An enterprise is selected with `--profile <name>` or `REACTORY_PROFILE`;
  with exactly one folder under `enterprises/` it is selected automatically.
  `--profile none` loads the inventory alone: one org, no mirroring.
- Paths named in a profile (the provisioning template) resolve against the
  enterprise repo first, then against the workspace.
- An org may name a `policy` file. `audit` compares the live GitHub org and
  repos with it, `harden` applies it, `seed` opens PRs with its baseline
  files. The upstream org uses `policies/public-upstream.json`.

## 2. What a checkout is made of

```mermaid
flowchart TB
    HOME["$REACTORY_HOME"]
    HOME --> WS["reactory-workspace/ (this repo)"]
    HOME --> SRV["reactory-express-server/ (root)"]
    HOME --> PWA["reactory-pwa-client/ (root)"]
    HOME --> CORE["reactory-core/"]
    HOME --> DATA["reactory-data/"]
    HOME --> WF["reactory-workflow-es/"]

    SRV -->|git submodule<br/>relative URL| DEVOPS["config/reactory<br/>reactory-express-server-devops"]
    SRV -->|nested clones<br/>from available.json| MODS["src/modules/*<br/>14 platform modules"]
    SRV -->|nested clones| ZQ["src/modules/&lt;enterprise module&gt;<br/>(enterprise-native)"]
    WS -->|nested clone<br/>gitignored| ENT["enterprises/&lt;name&gt;<br/>(enterprise configuration repo)"]
    SRV -->|nested clones| TEN["src/data/clientConfigs/*<br/>tenant configs"]
    PWA -->|git submodule<br/>relative URL| COREPL["src/components/plugins/reactory-core<br/>reactory-client-core-pwa-plugin"]
    PWA -->|nested clone| TELPL["src/components/plugins/reactory-telemetry"]
    DATA -->|nested clone| CDNPL["plugins/reactory-client-core"]

    classDef sub fill:#e8f1ff,stroke:#4a7bd0;
    classDef nested fill:#fff4e0,stroke:#d08a2b;
    class DEVOPS,COREPL sub;
    class MODS,ZQ,TEN,TELPL,CDNPL,ENT nested;
```

Blue: submodules, pinned by the parent's gitlink. Orange: nested clones in
gitignored folders, **not** pinned by anything in git. The inventory lists
all of them; the lock file pins them.

## 3. Remotes

```mermaid
flowchart LR
    subgraph gh_up["github.com/reactorynet (upstream)"]
        U["&lt;repo&gt; : master"]
    end
    subgraph gh_m["github.com/&lt;enterprise org&gt; (mirror)"]
        M["&lt;repo&gt; : main (protected)"]
    end
    subgraph local["developer checkout of &lt;repo&gt;"]
        O["origin"] ---|same URL as primary org| RN["reactorynet"]
        RN2["acme"]
    end
    RN <-->|fetch / push| U
    RN2 <-->|fetch / push branch, PR| M
    O -.->|plain git pull/push<br/>bin/*.sh scripts<br/>relative .gitmodules URLs| U
```

`origin` is a convenience alias for the primary org. Everything cross-org
goes through the named remotes, so `push --to acme` can refuse the protected
default branch and `sync` always knows which side is which.

Relative submodule URLs (`../reactory-express-server-devops.git`) resolve
against the superproject's `origin`, so a clone from the mirror takes its
submodules from the mirror without changing the committed `.gitmodules`.

## 4. Sync flow

```mermaid
flowchart TD
    A["sync --to T [--from S]"] --> B{"T is mirror?"}
    B -->|yes| S1["S = upstream"]
    B -->|no| S2["S = --from, or the only mirror"]
    S1 --> C
    S2 --> C
    C["fetch S and T remotes"] --> D{"T/branch contains S/branch?"}
    D -->|yes| UP["up to date"]
    D -->|no| W["git worktree add --detach tmp T/branch<br/>checkout -b sync/S-to-T-&lt;stamp&gt;"]
    W --> M["git merge --no-ff --no-commit S/branch"]
    M --> P["org-specific paths:<br/>T-owned: T has file → checkout from T, else git rm<br/>S-owned: never enters T"]
    P --> K{"conflicts left?"}
    K -->|yes| X["abort (or keep worktree with --keep-worktree)"]
    K -->|no| CM["commit merge"]
    CM --> Q{"T protected or --pr?"}
    Q -->|yes| PR["push sync/ branch to T<br/>gh pr create"]
    PR --> AM["gh pr merge --auto --merge<br/>(merge commit; never squash)"]
    AM -.->|review + checks pass| GHM["GitHub merges into T/branch<br/>deletes the sync/ branch"]
    Q -->|no, --push| DP["push to T/branch"]
    Q -->|neither| L["leave local, report"]
    PR --> Z["remove worktree"]
    DP --> Z
    L --> Z
```

Every PR the tool opens (`push --pr`, `sync --pr`, `bootstrap --pr`,
`seed --pr`) is armed with GitHub auto-merge, so the merge happens when the
required review and checks pass, not when someone remembers. The method
depends on the branch's job: sync and import PRs use the org's
`merge_strategy` (a merge commit, because the two default branches must
keep a common ancestor); feature and seed PRs use `pr_merge_method`
(squash, with the PR title and body as the commit message). The head branch
is removed by the repository's `delete_branch_on_merge` setting, which both
the upstream policy and the provisioning template turn on; `cleanup --to
<org>` deletes branches that landed before that setting existed (heads of
merged PRs, and `cleanup_branch_prefixes` that are ancestors of the default
branch).

The org-specific-path rule is symmetric. Every org lists the paths it owns.
When an org is the **target**, its version of those paths is restored (absent
in the target means removed). When it is the **source**, its paths never cross
into the target. So the enterprise keeps its CODEOWNERS and injected workflows and never
sends them upstream; the upstream keeps its seeded CodeQL, Dependabot and
`SECURITY.md` and never sends them into the mirror.

## 5. First import into a new org repo

```mermaid
sequenceDiagram
    participant Dev as developer
    participant RB as enterprise repo-management (e.g. Terraform)
    participant GH as enterprise org repo
    participant T as reactory-repos

    Dev->>T: provision --to acme --out auto
    T-->>Dev: YAML fragments (auto_init: false, merge-commit only)
    Dev->>RB: PR with fragments → atlantis plan → apply
    RB->>GH: create empty repo, teams, branch protection
    Dev->>T: bootstrap --to acme --group platform
    T->>GH: push reactorynet/master as import/reactorynet-master
    alt main does not exist (auto_init false)
        T->>GH: push reactorynet/master as main
    else main exists (auto-initialised, no common ancestor)
        T->>T: graft import/… = main + merge of reactorynet/master (unrelated histories, org paths kept)
        T->>GH: push import/…, gh pr create import/… → main (auto-merge: MERGE COMMIT)
    end
    Dev->>T: status --fetch
    T-->>Dev: reactorynet<>acme in sync
```

## 6. Adding another enterprise

1. `bin/reactory-repos enterprise init <name>` creates `enterprises/<name>/`
   from `templates/enterprise/` as a git repository of its own. Push it to
   the enterprise's organisation (`reactory-workspace-<name>`); other
   machines get it with `enterprise add <git-url>`. It is gitignored here.
2. Set the mirror org: `github_org`, `remote`, `default_branch`,
   `protected_default_branch`, `pr_title_prefix`, and how PRs land:
   `merge_strategy` (sync/import PRs, keep `merge-commit`), `pr_merge_method`
   (feature/seed PRs, `squash`), `auto_merge`, `cleanup_branch_prefixes`.
3. List the files that org injects into repositories under
   `org_specific_paths`. Anything the enterprise's repo-management system
   writes into repos belongs here, plus `CODEOWNERS`.
4. Choose `include_groups` (normally `["platform"]`), any single repos from
   other groups under `include_repos`, and any `exclude_repos`.
5. Add the enterprise's own modules and tenant configs under `extra_repos`.
6. Point `provisioning.template` at a template for the enterprise's repo
   system (adapt `templates/provisioning.yaml` in the enterprise repo;
   variables are `{{name}}`, `{{description}}`, `{{default_branch}}`,
   `{{upstream_org}}`, `{{upstream_name}}`, `{{github_org}}`, `{{letter}}`,
   `{{range}}`, plus anything in `vars`; conditionals are
   `{{#if has_dockerfile}}…{{/if}}` and `{{#if is_mirror}}…{{/if}}`).
7. Optionally point the org at a `policy` file (copy
   `policies/public-upstream.json`) so `audit` / `harden` / `seed` can keep
   its GitHub settings in line; the mirror org normally leaves this to its
   repo-management system.
8. Run `bin/reactory-repos --profile <name> list`, then `remotes`, `provision
   --out auto`, `bootstrap`, `lock`.

Nothing in the CLI names an enterprise or reactorynet; both come from the
files above, and the enterprise's files never enter this repository.
Several mirror orgs in one profile are allowed; `sync --to <upstream>` then
needs `--from <mirror>`.
