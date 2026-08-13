# Phase 2, Step 8 — Build the Parameterized Datasets

Four datasets serve the entire solution: three for Blob, one for SQL MI. Every pipeline references these and nothing else.

**Prerequisite:** Step 7 complete. `LS_Plexis_Blob` and `LS_Azure_SQL_MI` both published and passing **Test connection**. A dataset cannot be created against a linked service that doesn't resolve.

---

## Read this before you start clicking

Two properties on the delimited dataset — `columnDelimiter` and `firstRowAsHeader` — **cannot be parameterized from the ADF Studio canvas.** The delimiter is a dropdown and the header setting is a checkbox; neither offers an *Add dynamic content* link. They are parameterized in the JSON I generated, which works correctly at runtime.

So you have two honest options:

| Option | What you get | When to pick it |
|---|---|---|
| **Deploy the JSON** (recommended) | All five parameters work, including delimiter and header | You're Git-integrated. Same approach as the pipeline in guide 04. |
| **Build in the canvas** | Three path parameters work; delimiter and header are hardcoded | You aren't Git-integrated yet, or you want to learn the UI |

If you take the canvas route, hardcode `|` and *First row as header = checked*, then swap in the JSON later. All three Plexis files share a delimiter, so nothing breaks in the meantime.

The rest of this guide covers the canvas path, because that's where the mistakes happen. If you're deploying JSON, jump to Part E and validate.

---

## Part A — `DS_Plexis_Delimited_File`

The workhorse. One dataset for Member, Provider and Claims.

**Create:** Author → Datasets → **⋯** → New dataset → **Azure Blob Storage** → **DelimitedText** → Continue.

| Field | Value |
|---|---|
| Name | `DS_Plexis_Delimited_File` |
| Linked service | `LS_Plexis_Blob` |
| File path | leave blank for now |
| First row as header | **checked** |
| **Import schema** | **None** |

**Import schema = None is not optional.** A parameterized dataset serving three different files cannot carry one file's schema. An imported schema silently conflicts with the Copy activity's explicit translator and breaks the moment a second file type reuses the dataset.

### A1. Parameters tab — do this before the Connection tab

| Name | Type | Default |
|---|---|---|
| `ContainerName` | String | `plexis` |
| `FolderPath` | String | *(none)* |
| `FileName` | String | *(none)* |
| `Delimiter` | String | `\|` |
| `FirstRowAsHeader` | Bool | `true` |

Parameters must exist before you can reference them, or the dynamic content picker won't list them.

### A2. Connection tab

File path has three boxes — Container, Directory, File. Click each, then *Add dynamic content*:

```
Container   →  @dataset().ContainerName
Directory   →  @dataset().FolderPath
File        →  @dataset().FileName
```

Remaining settings:

| Setting | Value | Why |
|---|---|---|
| Column delimiter | `\|` | Pipe, not comma — `DenialReason` is free text built by `STRING_AGG` and can contain commas |
| Row delimiter | Default (`\r\n`) | |
| Encoding | UTF-8 | |
| Escape character | `"` | |
| Quote character | `"` | Doubled-quote escaping, the CSV standard |
| Null value | *(empty string)* | Blank in the file becomes NULL in staging |
| Compression | None | |

### A3. Schema tab

Must be **empty**. If anything appears, click **Clear**.

---

## Part B — `DS_Plexis_Binary_File`

Used only by the master pipeline's Validation activity, watching `BATCH_READY.txt`.

New dataset → Azure Blob Storage → **Binary** → `LS_Plexis_Blob`.

| Name | Type | Default |
|---|---|---|
| `ContainerName` | String | `plexis` |
| `FolderPath` | String | *(none)* |
| `FileName` | String | *(none)* |

Connection tab: same three `@dataset()` expressions as A2. Binary has no schema, delimiter or encoding — which is exactly why it's used here. The Validation activity only needs to know the blob exists and is non-empty; parsing it would be wasted work.

---

## Part C — `DS_Plexis_Binary_Folder`

Used by `PL_Archive_Batch_Files` for the recursive copy and delete.

New dataset → Azure Blob Storage → **Binary** → `LS_Plexis_Blob`.

| Name | Type | Default |
|---|---|---|
| `ContainerName` | String | `plexis` |
| `FolderPath` | String | *(none)* |

Connection tab: set Container and Directory only. **Leave File blank** — that's what makes this a folder reference. The recursive copy and the wildcard live in the activity, not the dataset.

Separate from B because a dataset with a file name can't be used as a folder source. Trying to reuse one for both is a common early mistake.

---

## Part D — `DS_SQLMI_Table`

New dataset → **Azure SQL Managed Instance** → `LS_Azure_SQL_MI`.

| Name | Type | Default |
|---|---|---|
| `SchemaName` | String | *(none)* |
| `TableName` | String | *(none)* |

Connection tab → **Edit** checkbox under Table, then:

```
Schema  →  @dataset().SchemaName
Table   →  @dataset().TableName
```

**Import schema: None.** Schema tab empty.

This dataset does two different jobs, worth understanding:

- **As a Copy sink** (`stg` / `Plexis_Claim`) the table genuinely matters — that's where 304 columns land.
- **As a Lookup dataset** it's just a connection carrier. Those Lookups run stored procedures, so the procedure defines the result shape and the table reference is effectively ignored. It still has to resolve to a real table or ADF validation complains, which is why the pipelines pass plausible values like `etl` / `Batch_Load`.

---

## Part E — Validate before publishing

Preview data forces you to supply parameter values — that's the point of the test.

### E1. Delimited file

`DS_Plexis_Delimited_File` → **Preview data**:

```
ContainerName    = plexis
FolderPath       = landing/2026/06/04
FileName         = CLAIMS_20260604.psv
Delimiter        = |
FirstRowAsHeader = true
```

You should see a proper grid with `ClaimID`, `ClaimTCN`, `ClaimType`… as column headers.

**If you see one giant column**, the delimiter is wrong — the whole row parsed as a single field. Fix it here; the same symptom in the pipeline is `columnCount = 1` at test D1 and much slower to diagnose.

Scroll right and confirm the column count looks like ~304, and that `DenialReason` values aren't spilling into neighbouring columns.

### E2. Binary file

Preview isn't meaningful for Binary. Instead, browse the file path with the folder icon — if it resolves and you can pick `BATCH_READY.txt`, the linked service and path parameters work.

### E3. Binary folder

Same — browse to `landing/2026/06/04` and confirm the folder resolves.

### E4. SQL MI table

`DS_SQLMI_Table` → **Preview data** with `SchemaName = stg`, `TableName = Plexis_Claim`.

Expect an empty grid with 310 columns. Empty is correct — nothing has loaded yet. What you're testing is that the SHIR reaches SQL MI and the managed identity has `SELECT`.

**If this fails**, the problem is Step 6 or 7, not Step 8:

| Error | Cause |
|---|---|
| Login failed for user `<token-identified principal>` | Contained user not created for the ADF managed identity |
| Cannot connect / timeout | SHIR not running, or can't reach the private endpoint |
| Invalid object name `stg.Plexis_Claim` | Script `01` not deployed, or you're on the wrong database |

---

## Part F — Publish

Validate all → **Publish all** (or commit to your feature branch if Git-integrated).

Organise them: right-click each dataset → Move to folder → `Plexis`. Worth two minutes now; the dataset list gets long.

---

## Sign-off checklist

- [ ] Four datasets created, all in the `Plexis` folder
- [ ] Every one has **Import schema: None** and an empty Schema tab
- [ ] `DS_Plexis_Delimited_File` preview renders a proper column grid, not one column
- [ ] `DS_Plexis_Binary_Folder` has **no** FileName parameter
- [ ] `DS_SQLMI_Table` preview against `stg.Plexis_Claim` returns an empty grid without error
- [ ] No hardcoded container, folder, file or table name anywhere in the four
- [ ] Published or committed

That last one is the real test of the step: if any pipeline later needs to hardcode a path because a dataset can't express it, the parameterization is incomplete.

---

## What this buys you

Three flat files, three SQL schemas, and every table in the solution are served by four objects. A delimiter change is one edit. Adding the Member file in iteration 2 needs **zero** new datasets — just new parameter values on the same `DS_Plexis_Delimited_File`.

The alternative — a dataset per file and per table — would be eleven objects today and eleven places to fix the same problem later.

Next: **Step 9**, build `PL_Load_Plexis_Claims`. See `04_Child_Pipeline_Implementation_Guide.md`.
