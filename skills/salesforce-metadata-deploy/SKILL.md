---
name: salesforce-metadata-deploy
description: Guidance and automation for deploying SFDX source metadata (Objects, Fields, Flows, Layouts, Permission Sets, etc.) to a Salesforce target org using the Salesforce CLI (`sf project deploy start`), featuring dry-run/validation, multi-pass dependency resolution, and intelligent error diagnostics.
---

# Salesforce Metadata Deploy

Use this skill when tasked with deploying local SFDX source metadata to a target Salesforce org, validating deployments, or diagnosing and fixing metadata deployment failures.

## Core Principles & Constraints
1. **Verify Target Org**: Always inspect and confirm the target org alias/username (`sf config get target-org` or via CLI parameters) before executing a live deployment.
2. **Prioritize Dry-Run**: Run validation with the `--dry-run` flag first to verify dependencies and metadata compatibility before writing changes to the target org.
3. **Automated Multi-Pass for Dependency Ordering**: If a failure is purely caused by deployment sequence (e.g., custom fields must exist before layout or validation rules are deployed), automatically group and deploy components in logical stages.
4. **Approval for Destructive Changes**: If an error requires modifying metadata files (removing references to missing fields, editing XML, or excluding components), summarize the root cause and request user confirmation before modifying files.
5. **No Apex Test Class Auto-Gen**: Do not generate Apex test classes or Jest tests for presales configuration/setup demonstrations.

---

## Deployment Workflow

### Step 1: Environment & Target Org Check
Inspect the current working directory and default org:
```bash
sf config get target-org
```
Present a concise summary to the user:
- **Project path**: Current working directory
- **Target Org**: Target Salesforce username or alias
- **Components**: Source directory (`--source-dir`) or metadata list (`--metadata`) to deploy

Prompt for confirmation if the target org is unconfigured or ambiguous.

---

### Step 2: Validation / Dry-Run
Run deployment validation without applying changes:
```bash
sf project deploy validate --source-dir <path> --target-org <alias/username>
# or
sf project deploy start --dry-run --source-dir <path> --target-org <alias/username>
```

- If **successful** (green): Proceed to ask for confirmation before live deployment.
- If **failed**: Analyze the error output or CLI log (`sf project deploy report`).

---

### Step 3: Error Handling & Problem Classification

When a deployment/validation fails, classify the error into one of two categories:

#### Category A: Dependency Ordering Issues (Automated Multi-Pass)
Common examples:
- Custom fields referenced in Page Layouts, Validation Rules, or Flows before the fields are created.
- Custom Object referenced by child objects or Master-Detail fields before the parent object is deployed.
- Record Types referencing new Picklist Values.

**Automated Action**:
1. Construct a staged multi-pass deployment plan (Pass 1 -> Pass 2 -> Pass 3):
   - **Pass 1 (Core Schema)**: `CustomObject`, `CustomField`, `GlobalPicklist`
   - **Pass 2 (Logic & Rules)**: `ValidationRule`, `RecordType`, `BusinessProcess`
   - **Pass 3 (UI & Security)**: `Layout`, `FlexiPage`, `Flow`, `PermissionSet`, `Profile`
2. Execute each pass sequentially using distinct `--metadata` or `--source-dir` flags.
3. Report the result of each pass.

#### Category B: Missing References / Broken Metadata (Diagnosis & Recommendation)
Common examples:
- Component references a field/object missing both locally and in the target org.
- Profile/PermissionSet references a standard org feature not enabled in the target org (e.g., Chatter, Knowledge).
- Metadata API version unsupported by the target org.

**Action**:
1. Parse the error log and extract:
   - **Component Name**: The affected metadata component
   - **Problem**: Technical error message
   - **File Path**: Relative path to the XML/JSON file
2. Summarize the root cause in 2–3 clear sentences.
3. Present concrete action options:
   - **Option 1**: Strip missing references from the local file/XML (show proposed diff).
   - **Option 2**: Exclude problematic components from the deployment payload.
   - **Option 3**: Retrieve missing dependencies from the target org (`sf project retrieve start`).
4. **Wait for user confirmation** before modifying files or permanently excluding components.

---

### Step 4: Live Deployment Execution
Once validation succeeds or fixes are approved:
```bash
sf project deploy start --source-dir <path> --target-org <alias/username>
```
Useful optional flags:
- `--ignore-conflicts`: Overwrite remote changes in scratch orgs or sandboxes.
- `--concise`: Provide streamlined CLI output.

---

### Step 5: Final Status Reporting
Report deployment results transparently:
- **Deployed Components**: Components successfully deployed
- **Skipped/Excluded Components**: Components omitted (if any)
- **Target Org Status**: Updated org alias/URL
