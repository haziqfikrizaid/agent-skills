---
name: e2na-config
description: 'Add or modify e2na EBL configuration: upload/download flows, routes, scenarios, messages, message groups, support file links, scheduler XML, spec generation (MSE). Use for: "add route", "add scenario", "add schedule", "reload p2c", "EBL config", "e2na", "generate spec", "excel to spec", "run MSE", "excel-spec-converter", "support-files-generator".'
---

# E2na EBL Configuration

Configure e2na EBL integration layers for e2open SSP projects — upload/download flows, routes, scenarios, messages, message groups, support file links, and spec generation.

## Reference Documentation

**Primary Source:** `guide.md` — Read this before proceeding.

## Workflow: Add a New Upload or Download Flow

1. Read existing `messages.ebl`, `groups.ebl`, `scenarios.ebl`, `routes.ebl`, and `start.ebl` to understand current patterns.
2. **Ask the user:** Does this flow need custom data transformation, filtering, or DB lookup logic? If yes, a Groovy handler in `solution/groovy_handlers/` may be needed — clarify what it should do before proceeding.
3. Add the **message** definition to `messages.ebl`.
4. Add the **message group** to `groups.ebl`.
5. Create the **scenario specification XML** under `ebl/scenario/specification/`.
6. Add the **scenario** (with all required `supportFileLink` entries) to `scenarios.ebl`.
7. Register any **new support files** in `start.ebl` (`add supportFile`) and `projects.ebl` (`add supportFileLink`).
8. Add the **route** to `routes.ebl` with correct profile groups, message groups, and scenario reference.

## Workflow: Spec Generation from Excel (MSE)

1. Copy the target `.xls` to `/tmp/mse_single/` and run `excel-spec-converter.sh -fr /tmp/mse_single -to .` from the `specs/` directory.
2. Run `support-files-generator.sh -fs <SPEC_NAME>.spec -all -udn -d -ow -exTplTp \*` — always use `-fs` to target a single spec and avoid regenerating all specs.
3. Review the diff. Do **not** commit `solution/resource/CPCExtResource.properties` — it gets wiped when running single-spec generation.
4. Commit the updated `.xls`, `.spec`, and changed `ebl/support_file/` files.
5. Deploy changed support files with `scripts/reload_p2c_script.sh`.

## Workflow: Add or Modify a Scheduled Download

1. Read the relevant scheduler XML (`support_file/schedules.xml`, `support_file/mtim-schedules.xml`, or `support_file/lts-schedules.xml`).
2. Add or reuse a **timing group** in the `<global>` section if needed.
3. Add the **schedule** entry in the appropriate `<group>` section.
4. For supplier-filtered schedules, update `support_file/DownloadFilters.prop` with the new `<scheduleName>.supplier=ID1^ID2^...` entry.
5. Deploy with `scripts/reload_p2c_script.sh`.

## Constraints

- **4-space indentation** — no tabs.
- **One blank line** between `} //route` and `} //route scenario` blocks.
- **No stray comments** like `//FORDSCPM-XXXX` left in files.
- Route IDs must be **unique** across `routes.ebl`.
- All new scenarios must list every required `supportFileLink`.
- Support file names must match exactly across `start.ebl`, `scenarios.ebl`, and `projects.ebl`.
- Timing group names must be **unique** within the scheduler XML `<global>` section.
- Scheduler files (`schedules.xml`, `mtim-schedules.xml`, `lts-schedules.xml`) are registered in `start.ebl` — do not add new ones without updating `start.ebl`.
- Follow existing naming conventions (e.g. `FooUploadScenario`, `FooManifestFile`, `FooUploadConfigFile1.0`).

## Deploying Changes to the Server

After modifying EBL files, use `scripts/reload_p2c_script.sh` to copy files to the server and reload EBL.

File paths are **relative to the EBL root** (`solution/project/ssp-ext/E2/e2na/ebl/`):

```bash
# Single file
scripts/reload_p2c_script.sh support_file/mtim-schedules.xml

# Multiple files
scripts/reload_p2c_script.sh support_file/mtim-schedules.xml support_file/DownloadFilters.prop

# EBL root files
scripts/reload_p2c_script.sh messages.ebl scenarios.ebl

# Scenario specification
scripts/reload_p2c_script.sh scenario/specification/myScenario.xml
```

The script will:
1. Back up existing server files before overwriting
2. Copy all specified files to their matching server paths under `/e2open/app/projects/ssp-ext/E2/e2na/ebl/`
3. Run `eoadmin p2c.sh` to reload EBL

Check logs after reload:
```bash
tail -100 /e2open/var/log/e2na/e2na.log
tail -100 /e2open/var/log/e2na/ebli.log
```

## Output

After making changes, report:
- Which files were modified and what was added
- The route ID and scenario name used
- Any support files that still need to be created in `ebl/support_file/`
