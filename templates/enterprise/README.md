# <Example Enterprise> configuration for reactory-workspace

Created by `bin/reactory-repos enterprise init example-enterprise`. This
folder is a git repository of its own: push it to the enterprise's
organisation and clone it back with `enterprise add <url>`. It is gitignored
inside reactory-workspace and must never be committed there.

| Path | Content |
|---|---|
| `profile.json` | Mirror org(s), protection, org-owned paths, mirrored groups, native repos, provisioning |
| `lock.json` | Written by `lock`; consumed by `clone --locked` / `checkout-lock` |
|  `templates/provisioning.yaml` | Rendered by `provision --to <org>`; adapt to your repo-management system |
| `rbac/` | Rendered fragments (`provision --to <org> --out auto`) |
| `docs/` | Your decision records |
