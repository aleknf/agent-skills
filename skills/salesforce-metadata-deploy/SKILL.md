---
name: salesforce-metadata-deploy
description: Guided deployment of SFDX source metadata (Objects, Fields, Flows, Layouts, Permission Sets, etc.) to a Salesforce org with the Salesforce CLI. Confirms the target org and its type, validates with a dry-run, deploys atomically in a single pass, and diagnoses failures with options for the user to approve. Stops before deploying Apex to production.
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

---

## Workflow

### Step 1: Pre-flight
Run from the SFDX project root (the folder with `sfdx-project.json`); if it is missing, stop and ask for the project path.

```bash
sf config get target-org --json
sf org display --target-org <alias> --json        # no --verbose
sf data query --target-org <alias> --query "SELECT IsSandbox, OrganizationType FROM Organization" --json
```

`IsSandbox = true` means sandbox; `OrganizationType` of `Developer Edition` or a scratch org means a developer environment; otherwise treat it as **production**. If the org type is unclear, ask.

Show the user a short summary and wait for confirmation:
- **Project path**
- **Target org**: alias, username, type (production / sandbox / scratch-developer)
- **Scope**: `--source-dir`, `--metadata`, or `--manifest` to deploy
- **Apex in scope?** (see Step 2)

If the target org is unset or ambiguous, ask which org to use. For **production**, the confirmation must name the org explicitly.

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
- **Failure**: go to Step 5. Failed components are in `result.details.componentFailures[]` (`fullName`, `componentType`, `fileName`, `lineNumber`, `problem`). `sf project deploy report --job-id <id>` re-reads a past run.

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

---

### Step 5: Diagnose failures
Do not retry blindly. For each failed component extract the **component**, **file path**, and **error message**, then explain the root cause in 2–3 plain sentences.

| Symptom | Likely cause | Options to offer |
|---|---|---|
| `no CustomField named X found` / reference to a missing object, field, or record type | The referenced component is not in the package and not in the org | Add it to the scope; retrieve it (`sf project retrieve start --metadata CustomField:Obj.Field__c`); remove the reference (show the diff) |
| `Unknown user permission: X` in a Profile or PermissionSet | A feature or license is not enabled in the target org | Enable the feature in the org; remove that permission from the file (show the diff) |
| API version not supported | `sourceApiVersion` is newer than the org supports | Lower `sourceApiVersion` in `sfdx-project.json` or use `--api-version` (show the change) |
| Conflicts reported on deploy | Org has changes the local source does not | Retrieve first; or `--ignore-conflicts` after approval |
| Apex coverage or test failures | Apex in the package, production target | Back to Step 2 |
| Anything else | | Show the raw error, say what is unknown, and ask |

Present the options, **wait for the user's choice**, apply only that change, then re-run Step 3. If a failure looks like an ordering problem, first check that the dependency really is in the same package; only then propose multi-pass.

---

### Step 6: Report
- **Result**: succeeded or failed, with the deploy ID
- **Org**: alias and type
- **Deployed components** and any **excluded or changed files** (with the reason)
- **Not verified**: anything you did not check
