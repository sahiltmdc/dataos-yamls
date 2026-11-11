# yamls

Resource manifests for running a service or a data product on DataOS. `secrets/` holds plaintext credentials and is **git-ignored** (see `.gitignore`). Only the `*.example.*` templates in it are tracked. Never commit, paste or share the real files.

## Layout

| Folder | What | Files |
|---|---|---|
| `depots/` | depot resources (admin-applied) | `postgres-depot.yaml`, `snowflake-depot.yaml` |
| `secrets/` | secret resources and raw secret material (admin-applied). Mode `700`. **Real files are git-ignored.** | `git-sync-secret.yaml`, `postgres-secret.yaml`, `postgres-secrets.txt`, `secret-sf-mine.yaml`, each with a tracked `*.example.*` template beside it |
| `grants/` | `grant` resources (type `grant`, `v2alpha`) | `grant.yaml` (`grant-use-any-compute`, `grant-use-any-secret`), `grant-depot.yaml` (`grant-use-any-depots` in the tenant, `grant-use-any-depots-system` in `system`), `grant.lower.yaml` (`grant-use-specific-compute-qa`: use case `use-specific-compute` for `qa-compute`, `approve: false`) |
| `grants/checks/` | requests for `ds user authorize -f`. **Not resources, do not apply.** | `authorize.yaml` |
| `services/` | `service` resources | `service.yaml` (`mcp-broker`) |
| `vulcan/` | `vulcan` resources | `domain-resource.yaml` (`sahl-demo-v2`) |

## Where each file was

Every file kept its name. Only the folder changed.

| Old path | New path |
|---|---|
| `grant.yaml`, `grant-depot.yaml`, `grant.lower.yaml` | `grants/` |
| `authorize.yaml` | `grants/checks/` |
| `postgres-depot.yaml`, `snowflake-depot.yaml` | `depots/` |
| `git-sync-secret.yaml`, `postgres-secret.yaml`, `postgres-secrets.txt`, `secret-sf-mine.yaml` | `secrets/` |
| `service.yaml` | `services/` |
| `domain-resource.yaml` | `vulcan/` |

To undo: `mv */*.y*ml */*.txt .` and `mv grants/checks/*.yaml .` from this directory, then remove the empty folders.

## Applying

Check the tenant first (`ds tenant list`, the `*` marks the current one), set the variables, lint, then apply:

```bash
export GRANTEE_ID=<your user id>        # ds user get
export DATAOS_TENANT=<tenant id>        # ds health (tenant-id)

ds rs apply -l -f grants/grant.yaml     # lint only
ds rs apply -f  grants/grant.yaml       # apply
```

The grant files use `${GRANTEE_ID}` and `${DATAOS_TENANT}`. `ds rs apply` and `ds user authorize` both interpolate environment variables, and an unset variable becomes an empty string (`authorize` then fails with "collection is required"). Export both first.

Suggested order for a fresh tenant: secrets, depots, grants, then services and vulcan resources. Secrets and depots are admin-applied.

Check what a grant gives (read-only): `ds user authorize -f grants/checks/authorize.yaml`, or the `dataos-grants` skill's `authz_check.sh`.

## Notes

- `grants/grant.lower.yaml` used to be named `grant-use-any-compute`, the same as the grant in `grants/grant.yaml`. Applying it would have replaced the any-compute grant, because `ds rs apply` is an upsert on type and name. It is now `grant-use-specific-compute-qa`. All three grant files lint valid.
- `grant.yaml` and `grant.lower.yaml` each had the `tags` key twice (once empty). Both are fixed.
- `grants/checks/authorize.yaml` had `collection: aug24`. It now uses `${DATAOS_TENANT}`. Tested: the collection makes no difference for `use-mcp-service`, whose atoms are pinned to the `system` collection. It does matter for tenant-scoped use cases.
- `secrets/` real files were never opened. The `*.example.*` templates were derived from the platform's secret schema (`ds develop schema resource manifest -t secret -v v2alpha`: `spec` takes `type`, `data` and `files`), the Vulcan stack definition in `dataos-component-install` (git-sync 4.7.2 reads `GITSYNC_USERNAME` and `GITSYNC_PASSWORD` from the secret), and the secret ids the depots reference. The three YAML examples lint valid. `postgres-secrets.example.txt` is a guess at the format.
- To use an example: copy it to the name without `.example`, fill it in, and apply as an admin. The copy is git-ignored.
