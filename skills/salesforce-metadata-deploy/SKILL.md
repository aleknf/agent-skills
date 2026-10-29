---
name: salesforce-metadata-deploy
description: Guided deployment of SFDX source metadata (Objects, Fields, Flows, Layouts, Permission Sets, etc.) to a Salesforce org with the Salesforce CLI. Confirms the target org and its type, validates with a dry-run, deploys atomically in a single pass, and diagnoses failures with options for the user to approve. Handles projects with their own staged deploy scripts and manifests. Stops before deploying Apex to production.
---

# Salesforce Metadata Deploy

Use this skill when deploying local SFDX source metadata to a Salesforce org, validating a deployment, or diagnosing a failed deployment.

## Core Principles
1. **Confirm the target org and its type.** Never deploy before the user has confirmed the org alias/username and whether it is production, sandbox, or scratch/developer.
2. **Dry-run first.** Validate before any live deploy.
3. **One atomic deploy by default.** A single Metadata API deployment is all-or-nothing and already orders dependent components that are in the same package. Splitting into passes removes that guarantee: if a later pass fails, earlier passes stay in the org and cannot be rolled back. Multi-pass is allowed only with explicit user approval (see Step 4).
4. **Never change files or delete anything without approval.** Editing XML, excluding components, destructive changes, and overwriting remote changes all need the user's OK after you show the exact change.
5. **No tests, and no Apex surprises.** Do not write Apex test classes or Jest tests for demo/presales setup. If the package contains Apex and the target is production, stop and explain (Step 2).
6. **Never print secrets.** Do not run `sf org display --verbose`; it prints the `sfdxAuthUrl`.
7. **Send only what changes.** A Profile deployed as a full file can overwrite other settings in the org. For access changes, send only the elements that change (for example `applicationVisibilities`), and never use `--ignore-conflicts` before the conflicts have been shown.
8. **Keep work files out of the project.** Temp copies, corrected manifests, and logs go in a working folder outside the project. Name the path in the report, and delete the files only after the user agrees.
9. **Keep the skill generic.** No project, client, or org names in examples; use placeholders such as `abc__` or `<alias>`.

---

## Workflow

### Step 1: Pre-flight
Run from the SFDX project root (the folder with `sfdx-project.json`); if it is missing, stop and ask for the project path.

```bash
sf config get target-org --json
sf org display --target-org <alias> --json        # no --verbose
sf data query --target-org <alias> --query "SELECT IsSandbox, OrganizationType FROM Organization" --json
```

`IsSandbox = true` means sandbox; `OrganizationType` of `Developer Edition` or a scratch org means a developer environment; otherwise treat it as **production**. A Developer Edition org reports `IsSandbox = false` but is a developer environment: do not treat it as production, and a dry-run there runs without `--test-level`. If the org type is unclear, ask.

Show the user a short summary and wait for confirmation:
- **Project path**
- **Target org**: alias, username, type (production / sandbox / scratch-developer)
- **Scope**: `--source-dir`, `--metadata`, or `--manifest` to deploy
- **Apex in scope?** (see Step 2)

If the target org is unset or ambiguous, ask which org to use. For **production**, the confirmation must name the org explicitly.

**Project-specific deploy setup.** Before choosing the scope, look for `README*`, `scripts/deploy/`, and `manifest/` (staged manifests such as `1-*.xml … 5-*.xml`).
- If the project has its own deploy script, read it. Tell the user what it does (namespace rewrite, tests, `--ignore-conflicts`, temp copies) and ask whether to use it or deploy directly. Scripts often run `RunSpecifiedTests`; for demo orgs the no-tests rule applies, so do not run it as-is without telling the user.
- If the user says earlier stages are done, ask which org they went to, then start at the next stage.
- **Namespace**: compare `namespace` in `sfdx-project.json` and the prefix used in source (for example `abc__`) with `SELECT NamespacePrefix FROM Organization`. If they differ, a direct deploy will fail; offer to deploy from a temp copy of the project with the prefix replaced (the project files stay untouched).
- **Manifest completeness**: compare the components in `force-app` (Apex classes, triggers, LWC, custom fields, WebLinks, flows, tabs, permission sets, flexipages) with the staged manifests and report any that no stage lists. Do not add them without approval (Step 5).
- **`.forceignore`**: note entries that hide components the manifest names (for example `groups/`, `sharingRules/`).

---

### Step 2: Apex check
Look in the scope for `*.cls` and `*.trigger` files (including `-meta.xml`).

- **No Apex**: continue.
- **Apex present and target is production**: stop. Production requires tests to run and 75% coverage for Apex in the package (`RunLocalTests` is the default there, and it runs all local tests in the org). Do not write test classes and do not pick a test level on your own. Tell the user and offer:
  1. Remove Apex from this deployment and deploy the declarative parts only.
  2. Deploy to a sandbox or scratch org instead (no tests by default).
  3. The user decides how to handle coverage (existing tests, `--test-level RunSpecifiedTests --tests <names>`).
- **Apex present and target is sandbox/scratch**: continue; no tests run by default.

For demos, prefer declarative solutions (Flow, LWC with `lightning/uiGraphQLApi` or `lightning/uiRecordApi`) over Apex.

---

### Step 3: Validate (dry-run)
Use `--json` so errors can be parsed, and `--wait` so the command does not return early.

```bash
sf project deploy preview --target-org <alias> --source-dir <path>      # see what deploys and any conflicts
sf project deploy start --dry-run --target-org <alias> --source-dir <path> --wait 30 --json
```

- **Which command**: use `sf project deploy start --dry-run` in every org type. `sf project deploy validate` is intended for production only: it defaults to `RunLocalTests`, so it runs every local test in the org even for a config-only package. Use it only when the user wants a quick deploy later (`sf project deploy quick --job-id <id>`), and tell them it will run local tests.
- `preview` shows conflicts only for orgs with source tracking (scratch/developer sandboxes).
- **Success**: show the summary and ask for confirmation before the live deploy.
- **Failure**: go to Step 5. Failed components are in `result.details.componentFailures[]` (`fullName`, `componentType`, `fileName`, `lineNumber`, `problem`). `componentFailures` is a list for several failures but a single object for one, so handle both, and de-duplicate repeated entries. `sf project deploy report --job-id <id>` re-reads a past run; without an ID use `--use-most-recent`, or query `SELECT Id, Status, NumberComponentsDeployed, NumberComponentErrors FROM DeployRequest ORDER BY CreatedDate DESC LIMIT 2` with `--use-tooling-api`. When parsing JSON with Python, run it with `-I`.

---

### Step 4: Live deploy
After a successful dry-run and confirmation of the org (production: explicit confirmation naming the org):

```bash
sf project deploy start --target-org <alias> --source-dir <path> --wait 30
```

Flags:
- `--ignore-conflicts`: only after `preview` has shown the conflicts and the user approved overwriting them.
- **Do not use** `--ignore-errors` (it disables rollback) or `--purge-on-delete`.
- Deploying never deletes components from the org. Deletions need `--pre-destructive-changes` / `--post-destructive-changes` with a manifest the user approved.

**Multi-pass (only on request or approval).** Offer it when the user asks for it, or when a failure is clearly caused by something that must exist in the org first (for example a feature or setting that must be enabled, or a component deployed by another team). Explain the risk (no rollback across passes), show the plan, and wait for approval. Typical order: schema (`CustomObject`, `CustomField`, `GlobalValueSet`), then logic (`ValidationRule`, `RecordType`, `Flow`), then UI and security (`Layout`, `FlexiPage`, `PermissionSet`, `Profile`). Dry-run each pass before deploying it and stop at the first failure.

**Staged manifests.** When the project deploys in stages, a later stage may be required by an earlier one (for example a FlexiPage that embeds a Dashboard from the next stage). Prefer merging those stages into one deployment, with approval, over multi-pass: it stays atomic.

**Companion scripts with org-wide effect.** Scripts that change many profiles, layout assignments, or sharing (for example one that assigns layouts to every profile) are separate from the deploy. Read the script first, show the scope (how many profiles or objects, whether it uses `--ignore-conflicts`), and ask for confirmation before running it.

**Access after deploy.** A new `CustomApplication`, `CustomTab`, or custom object is not visible to users until something grants it.
- If the project already has permission sets or permission set groups per role, prefer them over editing profiles, and ask the user which to use.
- Otherwise offer a small follow-up deployment that grants System Administrator access: a Profile `Admin` file with only `applicationVisibilities` (`visible` true, `default` false) and `tabVisibilities` where needed. Show the file, dry-run it, and deploy it only after the user agrees.

---

### Step 5: Diagnose failures
Do not retry blindly. For each failed component extract the **component**, **file path**, and **error message**, then explain the root cause in 2–3 plain sentences.

| Symptom | Likely cause | Options to offer |
|---|---|---|
| `no CustomField named X found` / reference to a missing object, field, or record type | The referenced component is not in the package and not in the org | Add it to the scope; retrieve it (`sf project retrieve start --metadata CustomField:Obj.Field__c`); remove the reference (show the diff) |
| `Unknown user permission: X` in a Profile or PermissionSet | A feature or license is not enabled in the target org | Enable the feature in the org; remove that permission from the file (show the diff) |
| API version not supported | `sourceApiVersion` is newer than the org supports | Lower `sourceApiVersion` in `sfdx-project.json` or use `--api-version` (show the change) |
| Conflicts reported on deploy | Org has changes the local source does not | Retrieve first; or `--ignore-conflicts` after approval |
| `Unable to find Apex action class` / `No MODULE named markup://c:X found` in an LWC | The Apex class or LWC is in `force-app` but no manifest lists it | Add the missing members to a temp copy of the manifest (or the project manifest, after showing the diff) |
| `An object 'X' of type Group/SharingRules was named in package.xml, but was not found in zipped directory` | `.forceignore` hides the component the manifest names | Remove those lines from `.forceignore` in a temp copy (or the project, after showing the diff); note the org-wide sharing effect first |
| `Invalid dashboard Unique Name X - no dashboards found` in a FlexiPage | The Dashboard is in a later stage | Merge the stages into one deployment (see Step 4) |
| Apex coverage or test failures | Apex in the package, production target | Back to Step 2 |
| Anything else | | Show the raw error, say what is unknown, and ask |

Present the options, **wait for the user's choice**, apply only that change, then re-run Step 3. If a failure looks like an ordering problem, first check that the dependency really is in the same package; only then propose multi-pass.

---

### Step 6: Report
- **Result**: succeeded or failed, with the deploy ID
- **Org**: alias and type
- **Deployed components** and any **excluded or changed files** (with the reason)
- **Differences from the project's plan**: manifest additions, merged stages, `.forceignore` changes, temp-copy namespace rewrite, and whether the project files were left untouched
- **Access**: whether new apps, tabs, and objects are visible to the intended users, checked read-only (for example `SELECT Id FROM AppMenuItem WHERE Name IN (...)`); say "UI not checked" if you did not open the org
- **Work files**: path of any temp copy, corrected manifest, or log kept outside the project
- **Companion scripts** run or still pending
- **Not verified**: anything you did not check
