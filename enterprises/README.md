# enterprises/

One nested git repository per enterprise, **gitignored**: this folder is the
workspace's equivalent of the Reactory server's `src/modules/`. The workspace
repository itself names no enterprise; everything specific to one lives in
its own repository, in that enterprise's organisation, and is cloned here.

```
enterprises/
  <name>/                 <- git repo owned by the enterprise (never committed here)
    profile.json          <- the enterprise profile (orgs, protection, owned paths, native repos)
    lock.json             <- SHA pin per repo, written by `lock`
    templates/*.yaml      <- provisioning template(s) for its repo-management system
    rbac/*.fragment       <- rendered repo definitions (`provision --to <org> --out auto`)
    docs/                 <- its decision records
```

```bash
bin/reactory-repos enterprise add git@github.com:<org>/reactory-workspace-<name>.git
bin/reactory-repos enterprise init <name>        # new enterprise from templates/enterprise/
bin/reactory-repos enterprise list
bin/reactory-repos --profile <name> list          # or export REACTORY_PROFILE=<name>
```

With exactly one enterprise present it is selected automatically.
