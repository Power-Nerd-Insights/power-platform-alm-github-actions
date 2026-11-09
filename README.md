# Power Platform ALM with GitHub Actions

Template GitHub Actions workflows for moving a Power Platform solution from dev to test to prod, with source control, a single managed build, Solution Checker, per-environment configuration and an approval gate before production.

Built for the Power Nerd session on 13 November 2026. The slide deck is in [`deck/`](deck/).

> **Status:** these workflows are being tested in a demo tenant ahead of the session. This line will be updated once they have run end to end.

## Why this exists

Exporting and importing solutions by hand works for a single maker, but it breaks down as soon as more than one person or environment is involved. Production ends up with unmanaged changes, connection references are fixed by hand after every import, and nobody can say what changed or who approved it.

These workflows are a starting point for an ALM process where:

- source control is the record of what the solution contains,
- one managed build is produced and the same file is deployed to every environment,
- environment-specific values are supplied at deployment time, not edited by hand,
- production deployments need a named person to approve them.

If you only need governed promotion between environments and do not need source control or pro-code builds, look at [pipelines in Power Platform](https://learn.microsoft.com/power-platform/alm/pipelines) first. They need far less setup.

## How it works

```
 dev                         GitHub                                   test / prod
 ───                         ──────                                   ───────────
 export-solution.yml  ──►  branch + pull request  ──► review, merge
                                                         │
                           deploy-solution.yml  ◄────────┘
                             build: version, pack, Solution Checker
                             deploy-test ───────────────────────────►  test
                             deploy-prod (waits for approval) ──────►  prod
```

| Workflow | Trigger | What it does |
|---|---|---|
| `export-solution.yml` | Manual | Exports the solution from dev as unmanaged and managed, unpacks both into `src/`, commits to a new branch and opens a pull request. |
| `deploy-solution.yml` | Push to `main` under `src/` or `deployment-settings/`, or manual | Stamps the version, packs the managed solution, runs Solution Checker, then imports it into test and, after approval, into prod. |
| `oidc-check.yml` | Manual | Optional. Checks that the app registration can authenticate with OIDC instead of a client secret. Not used by the other workflows. |

Design decisions worth knowing before you change anything:

- **Build once, deploy many.** The managed zip is built in the `build` job and passed to the deploy jobs as an artefact. Test and prod receive the identical file.
- **Version from the run number.** The workflow keeps `major.minor` from `Solution.xml` and sets the rest to `<run number>.0`, so each release has a higher version than the last and can be traced back to its run. Change `major.minor` in source when you want a new series.
- **Deployment settings are a template.** `deployment-settings/template.json` holds placeholders. Each deploy job fills them from that GitHub environment's variables, so no connection IDs or tenant values are committed.
- **First install and upgrade are handled differently.** Before importing, each deploy job checks whether the solution is already installed in the target. If it is, the import uses stage-and-upgrade, so components removed from source are also removed from the target. If it is not, the import is a plain first install, because an upgrade needs an existing version to upgrade from.
- **Approval is a GitHub feature.** The `prod` job references the `prod` GitHub environment, which has required reviewers. Nothing in Power Platform enforces it.

## Repository structure

```
.github/workflows/      the three workflows
deployment-settings/    template.json, the deployment settings template
src/<SolutionName>/     unpacked solution, written by export-solution.yml
deck/                   session slides
SETUP.md                full setup checklist
```

## Getting started

Follow [SETUP.md](SETUP.md). In outline:

1. Three Dataverse environments and a solution in dev.
2. An Entra ID app registration with a client secret.
3. That app added as an application user in each environment, and the Dataverse connections in test and prod shared with it.
4. A GitHub repository with `test` and `prod` environments, variables and the `PP_CLIENT_SECRET` secret.

Then run `export-solution`, review and merge the pull request, and approve the prod deployment when it asks.

## Configuration

All tenant-specific values live in GitHub settings, not in the workflow files.

| Scope | Name | Purpose |
|---|---|---|
| Repository variable | `SOLUTION_NAME` | Unique name of the solution |
| Repository variable | `PP_TENANT_ID` | Entra tenant ID |
| Repository variable | `PP_APP_ID` | Application (client) ID |
| Repository variable | `DEV_ENVIRONMENT_URL` | Dev environment URL |
| Repository variable | `CHECKER_GEO` | Optional. Solution Checker geo, defaults to UnitedStates |
| Repository secret | `PP_CLIENT_SECRET` | Client secret for the app registration |
| Environment variable (`test`, `prod`) | `ENVIRONMENT_URL` | Target environment URL |
| Environment variable (`test`, `prod`) | `DATAVERSE_CONNECTION_ID` | Connection to bind to the connection reference |
| Environment variable (`test`, `prod`) | `API_BASE_URL` | Value for the `pni_ApiBaseUrl` environment variable |

## Adapting it to your own solution

- **Different connection references or environment variables.** Export once, run `pac solution create-settings --solution-zip <zip> --settings-file check.json`, and copy the names into `template.json` with a `${PLACEHOLDER}` for each value. Add each placeholder to the `envsubst` list and the check loop in both deploy jobs, and create the matching GitHub environment variables.
- **More environments.** Copy a deploy job, give it a new `environment:` name and a `needs:` on the previous job, and create the GitHub environment.
- **Stricter quality gate.** `check-solution` fails the run only when the analysis itself fails. To fail on findings, read the results from the uploaded `solution-checker-logs` artefact in a script step.

Changes that are risky:

- Editing a solution in test or prod directly. Managed solutions can't be edited there, and unmanaged changes on top of them create layers that later imports will not remove.
- Lowering `major.minor`. Imports of a lower version are rejected as upgrades.
- Renaming the GitHub environments. Their variables, secrets and protection rules are tied to the name.

## What can go wrong

| Symptom | Likely cause |
|---|---|
| First Power Platform step fails on authentication | Secret expired or copied wrongly (the Secret ID instead of the Value), or the app user is missing in that environment |
| Import fails with `ConnectionAuthorizationFailed` (403) on a connection | The connection is not shared with the application user. Share it as **Can use** and re-run the failed job. The object ID in the error is the service principal's object ID, not the app ID |
| Import fails on a connection reference with "not found" | The connection ID belongs to another environment, or was copied without its `shared-…` prefix |
| Flow imports but stays off | Connection reference not bound. Check the deployment settings for that environment |
| Import fails on missing dependencies | The solution uses a component that isn't in the target. Add it to the solution or deploy it first |
| `deploy-test` fails with "variable is not set" | A GitHub environment variable is missing for `test` |
| Import fails with "Cannot create a holding solution for missing base" | An upgrade was attempted where the solution is not installed. The install check should prevent this; check its log line |
| Prod job never starts | It is waiting for approval. Check the run page |

## Next steps

These were left out to keep the template readable for a first setup.

- **Move to OIDC.** OIDC (workload identity federation) replaces the stored client secret with short-lived tokens that Entra accepts only from this repository, branch or environment. Add a federated credential to the app registration, add `id-token: write` to each job's permissions, and remove the `client-secret` inputs. Repositories created after 15 July 2026 use a subject in the form `repo:OWNER@OWNER_ID/REPO@REPO_ID:environment:prod`. See [GitHub's OIDC reference](https://docs.github.com/en/actions/reference/security/oidc) and [Microsoft's tutorial](https://learn.microsoft.com/en-us/power-platform/alm/tutorials/github-actions-oidc-fic). `oidc-check.yml` is a minimal test.
- **Separate identity for prod.** Use a second app registration for prod and store its secret as a `prod` environment secret. GitHub only releases environment secrets after approval.
- **Reusable workflow.** The two deploy jobs are deliberately repeated for readability. A reusable workflow called once per environment removes the duplication.

## References

- [microsoft/powerplatform-actions](https://github.com/microsoft/powerplatform-actions) and its [releases](https://github.com/microsoft/powerplatform-actions/releases). The `action.yml` in each action folder is the most accurate list of inputs.
- [GitHub Actions for Microsoft Power Platform](https://learn.microsoft.com/power-platform/alm/devops-github-actions)
- [Pre-populate connection references and environment variables](https://learn.microsoft.com/power-platform/alm/conn-ref-env-variables-build-tools)
- [Managing environments for deployment](https://docs.github.com/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)

## Licence

Code and workflows are released under the [MIT Licence](LICENSE).
