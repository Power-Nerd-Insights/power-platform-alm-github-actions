# Setup checklist

Everything the workflows need before the first run. Do this in advance of rehearsal, in the order below. Nothing here goes into the repository itself: every tenant-specific value lives in Entra ID, Power Platform or GitHub settings.

Items marked **[confirm]** are untested assumptions. Tick them off during the first dry run.

## 1. Power Platform tenant

| Item | Detail |
|---|---|
| Three environments with Dataverse | dev, test, prod. Developer or sandbox type is fine for a demo. Test and prod must not already contain the solution as unmanaged. |
| Environment URLs recorded | e.g. `https://<org>.crm.dynamics.com`. Needed for GitHub variables in section 4. |
| Sample solution in dev | Publisher prefix `pni`. Contains: table `pni_githubdeployments`, environment variable `pni_ApiBaseUrl` (text), connection reference `pni_DataverseConnection` (Microsoft Dataverse), cloud flow `GHD - Flow` using that connection reference. |
| Settings names confirmed | After the first export, run `pac solution create-settings --solution-zip <zip> --settings-file check.json` and compare the `SchemaName` and `LogicalName` values with `deployment-settings/template.json`. Generated names can differ from the display names. |
| Dataverse connection in test and in prod | Created once in each environment. Record each connection ID (from the URL when you open the connection in make.powerapps.com). |
| Connection usable by the deployment identity | Microsoft Learn states that on import, connections are validated against the owner of the connection reference, so the connection must be owned by or shared with that owner. With a service principal deploying, the owner is the application user. **[confirm]** whether a Dataverse connection can be shared with the application user, or whether the flow needs a different arrangement. This is the item most likely to fail. |

## 2. Entra ID app registration

1. Create one app registration, single tenant. Record the **Application (client) ID** and **Directory (tenant) ID**.
2. Under **Certificates & secrets**, create a client secret. Copy the **Value** immediately: it is shown once. The Secret ID is not the secret.
3. Choose an expiry and put a reminder in your calendar before that date. When the secret expires, every workflow run fails on its first Power Platform step.
4. Access to Dataverse is granted through the application user in section 3. Some Microsoft guides also add the Dynamics CRM API permission to the app registration. **[confirm]** whether it is needed by trying without it first.

The powerplatform-actions take a client secret only. A certificate would need `pac auth create` run directly in a script step instead of the actions, so it is not used here.

## 3. Power Platform application users

In the Power Platform admin center, for **each** of dev, test and prod:

1. Add the app registration as an application user.
2. Assign the System Administrator security role. A narrower custom role is better for real projects, but out of scope for the demo.

## 4. GitHub repository

| Setting | Value |
|---|---|
| Visibility | Public. Required reviewers on GitHub Free, Pro and Team plans are only available for public repositories. |
| Settings > Actions > General | Allow GitHub Actions to create and approve pull requests: **on** (the export workflow opens a PR with `GITHUB_TOKEN`). |
| Environment `test` | No protection rules. |
| Environment `prod` | Required reviewers: yourself. Leave **Prevent self-review off**, otherwise you cannot approve a run you started. Deployment branches: `main` only. |

### Repository variables (Settings > Secrets and variables > Actions > Variables)

| Name | Example | Notes |
|---|---|---|
| `SOLUTION_NAME` | `PowerNerdAlmDemo` | Unique name, not display name. |
| `PP_TENANT_ID` | GUID | Not a secret, but tenant-specific, so kept out of YAML. |
| `PP_APP_ID` | GUID | Application (client) ID. |
| `DEV_ENVIRONMENT_URL` | `https://<org-dev>.crm.dynamics.com` | |
| `CHECKER_GEO` | `UnitedStates` | Optional. Defaults to UnitedStates if unset. Set it to the geo of the environment named in `DEV_ENVIRONMENT_URL`. |

### Environment variables (on `test` and on `prod` separately)

| Name | Notes |
|---|---|
| `ENVIRONMENT_URL` | That environment's URL. |
| `DATAVERSE_CONNECTION_ID` | Connection ID from section 1 for that environment. |
| `API_BASE_URL` | Any value that differs between test and prod, so the demo can show it changed. |

### Repository secret (Settings > Secrets and variables > Actions > Secrets)

| Name | Value |
|---|---|
| `PP_CLIENT_SECRET` | The client secret **Value** from section 2. |

A repository secret is available to every job, including `build` and `export`, which do not use a GitHub environment. GitHub masks it in logs. Because the workflows only run on push to `main` and on manual dispatch, secrets are never passed to workflows triggered from forks.

## 5. Moving to OIDC later

Once the pipeline works, the secret can be replaced with workload identity federation so that nothing long-lived is stored in GitHub. That change, and the subject claim format it depends on, is covered in the README as a next step. It is not needed for the session.

## 6. First dry run order

1. Run `export-solution` manually. Confirm the PR appears with `src/<SOLUTION_NAME>/` populated.
2. Merge the PR. Confirm `deploy-solution` starts.
3. Watch `build`: version stamp, both packs, Solution Checker.
4. Watch `deploy-test`: who-am-i, import. Check the solution in test is managed, the environment variable has the test value, the connection reference is bound, and the flow is on.
5. Approve `deploy-prod`. Repeat the checks in prod.
6. Make a small change in dev and repeat from step 1, to prove the upgrade path as well as the first install.
