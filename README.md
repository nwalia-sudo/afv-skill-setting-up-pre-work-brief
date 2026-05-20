# Pre-Work Brief skills

Two [Agentforce Vibes](https://github.com/forcedotcom/afv-library) skills for Pre-Work Brief on Field Service Mobile:

1. **`setting-up-pre-work-brief`** — sets up Pre-Work Brief end to end against a Salesforce org with the default managed flow and the standard prompt template.
2. **`customizing-pre-work-brief`** — generates a vertical-specific brief deployable as code. Ships with seed templates for HVAC, banking, telecom, healthcare, and retail merchandising under `templates/`. The admin provides industry, website, and a sentence about what their technicians do; the skill produces custom objects, an autolaunched flow, a prompt template, a permission set, and a test Work Order — all from metadata. Ends by creating a fresh test WO + SA assigned to a Service Resource so the admin can validate on-device immediately.

Both are early candidates for contribution to `forcedotcom/afv-library`. The folder structure mirrors that repo: each skill lives at `skills/<name>/SKILL.md`. When the skills are ready to merge upstream, the folders drop in unchanged.

## What is Pre-Work Brief

Pre-Work Brief is an Einstein-generative-AI feature in Field Service Mobile. It renders a concise, AI-generated summary of an upcoming Work Order in the **Overview tab** of the mobile app, grounded in real Salesforce data (Work Order, Account, Service Appointment, Work Plans, etc.).

Salesforce documents the manual setup at:

- [Set Up Pre-Work Brief for Field Service Mobile Workers](https://help.salesforce.com/s/articleView?id=service.mfs_einstein_pre_work_brief.htm)
- [Pre-Work Brief and Your Data](https://help.salesforce.com/s/articleView?id=service.mfs_einstein_pre_work_brief_data.htm)
- [Set Up Permissions for Einstein for Field Service Mobile](https://help.salesforce.com/s/articleView?id=service.fs_einstein_gen_ai_setup.htm)

The skill automates as many of the documented steps as can be driven from the Salesforce CLI, and surfaces clear deeplinks for the few that still require a click in Setup.

## What `setting-up-pre-work-brief` does

In order:

| Step | What it does | How |
|---|---|---|
| 0 | Detects provisioning state, classifies the org | 5 CLI diagnostic queries |
| 0.5 | Asks the admin which technician to enable Pre-Work Brief for; auto-picks if no preference | SOQL on `ServiceResource` + `User` |
| 1 | Confirms target org and the Einstein for Field Service license | `sf org display`, `sf data query` on `PermissionSetLicense` |
| 2 | Enables Lightning Data Service for the Field Service mobile app | Metadata deploy of `FieldServiceSettings.enableLsdkMode = true` |
| 3 | Points at Salesforce's base Einstein generative AI setup | Help URL (UI step) |
| 4 | Assigns PSLs and permission sets to the admin (3 perm sets) and the chosen technician (2 perm sets) | `sf org assign permsetlicense` / `permset` |
| 5 | Deploys (or detects) the Pre-Work Brief prompt template, then surfaces the activation deeplink | `sf project deploy start` of a `GenAiPromptTemplate` metadata file + `sf org open --url-only` |
| 6 | Verifies the Work Order grounding fields exist | `sf sobject describe` |
| 7 | Adds the `PreWorkBriefPromptTemplate` field to the Work Order layout assigned to the technician | Metadata retrieve + Python edit + redeploy |
| 8 | Wires the prompt template to a Work Order — either an existing record or a fresh auto-created WO + SA assigned to the chosen technician's Service Resource | `sf data update record` or `sf data create record` chain |
| 9 | Documents on-device verification | Manual mobile test |

## What `customizing-pre-work-brief` does

Runs after the base setup is complete. Generator-style: the admin specifies an industry, a company name, and a website; the skill produces a deployable bundle (custom objects, flow, prompt template, permission set, test record) tailored to that vertical.

| Step | What it does | How |
|---|---|---|
| Pre | Confirms `setting-up-pre-work-brief` artifacts exist | `sf org list metadata`, `sf data query`, `sf sobject describe` |
| 1 | Builds business context from `--website` (via WebFetch) and a 1-2 sentence admin description | Tool call + admin prompt |
| 2 | Loads the vertical seed template from `templates/<industry>.md` (or synthesizes one if the seed is missing) | File read |
| 3 | Audits the org's existing schema; cross-references against the seed's recommended custom objects to avoid duplicates | `sf sobject describe`, `sf sobject list` |
| 4 | Generates and deploys the approved custom objects, fields, and any new WorkOrder lookups | Metadata deploy of `CustomObject`, `CustomField` |
| 5 | Generates a from-scratch autolaunched `PromptFlow` flow with `Capability` trigger type — no Save-As click required — and activates it via `FlowDefinition.Metadata.activeVersionNumber` | `sf project deploy start` of `Flow`, then Tooling REST PATCH |
| 6 | Generates the `GenAiPromptTemplate` referencing the new flow, with business-context paragraph + section structure from the seed; surfaces activation deeplink | `sf project deploy start` + `sf org open --url-only` |
| 7 | Generates a permission set granting CRUD + FLS on the new objects/fields; assigns to admin + technician | Metadata deploy + `sf org assign permset` |
| 8 | Creates a fresh test Work Order with sample seed data populating the new customs, plus SA + AssignedResource | `sf data create record` chain |

The two steps that still need a click in Setup are:

- **Step 3** of `setting-up-pre-work-brief` — Salesforce's base Einstein generative AI setup (link given).
- **Step 6 activation** — the prompt template's "Activate" button in Prompt Builder. The skill emits a one-time signed deeplink direct to the template editor so it's a single click. (The flow itself is activated programmatically in Step 5.)

### Seed templates

`skills/customizing-pre-work-brief/templates/` ships with five vertical seeds:

| Seed | Status | Recommended custom objects |
|---|---|---|
| `hvac.md` | **Verified end-to-end against `afvuser` 2026-05-20** | `Maintenance_Contract__c`, `Refrigerant_Log__c` |
| `banking.md` | Scaffold (untested) | `ATM_Cassette__c`, `Compliance_Check__c` |
| `telecom.md` | Scaffold (untested) | `Service_Drop__c`, `Outage_History__c` |
| `healthcare.md` | Scaffold (untested) | `Device_Calibration__c`, `FDA_Compliance_Log__c` |
| `retail-merchandising.md` | Scaffold (untested) | `Store_Visit_Plan__c`, `Planogram_Compliance__c` |

Each seed contributes a section structure for the prompt template, recommended custom objects with field lists, standard fields to query, and a cadence example. For verticals not in the library, pass `--industry other` and the skill synthesizes from the website + admin description.

## How to use it

Both skills are consumed by an AI coding agent (Claude Code, Agentforce Vibes, Cursor, Codex, etc.) that supports the Agent Skills format. Drop the `skills/<name>/SKILL.md` files into your skills directory; the agent picks them up via the `description` frontmatter.

Run the base skill against an org by asking:

> Set up Pre-Work Brief on the `<org-alias>` Salesforce org.

After base setup is in place, layer customization with:

> Customize Pre-Work Brief for `<company>` (industry: `<hvac|banking|telecom|healthcare|retail-merchandising|other>`) on `<org-alias>`. Their website is `<url>`.

The agent walks the steps in order, surfaces the activation deeplinks at the prompt-template steps, and creates a fresh test Work Order so the admin can validate on-device immediately.

## Prerequisites

- Salesforce CLI (`sf`) v2.x
- A Salesforce org with **Field Service** enabled and the **Einstein for Field Service** or **Agentforce for Field Service** add-on
- An admin user with `Customize Application` and `Manage Profiles and Permission Sets`
- For the on-device test: the Field Service Mobile app on iOS or Android, signed in as a technician with the Field Service Mobile license

## Tested against

`afvuser@salesforce.com` trial org, May 2026.

- `setting-up-pre-work-brief` Steps 0 through 8 verified clean. Step 9 (mobile-device verification) requires a working Einstein LLM runtime entitlement.
- `customizing-pre-work-brief` HVAC seed verified end-to-end on 2026-05-20: 2 custom objects + 11 fields deployed, from-scratch flow built and activated programmatically, prompt template deployed, permission set assigned, test WO 00000325 created with full sample data. The other four seeds (banking, telecom, healthcare, retail-merchandising) are scaffolds and need an end-to-end run before customer use.

Step 9 (mobile-device verification) requires a working Einstein LLM runtime entitlement, which isn't provisioned in all trial orgs — track that as a separate org-provisioning question if you hit "We hit a snag" on mobile.

## Limitations

- **Activation isn't fully automatable.** Salesforce does not currently expose a programmatic activation path for `GenAiPromptTemplate` (we tried Tooling REST PATCH on `IsActive`, Connect API `/activate` endpoints across v62-v66, Apex `ConnectApi.EinsteinLLM` methods, and metadata `activeVersionNumber` — none worked). The skill deploys the template with `<status>Published</status>` and surfaces a deeplink for the one-click activation.
- **Field-level security** for `PreWorkBriefPromptTemplate` is documented as a Setup click rather than a Profile metadata deploy, to avoid stomping on unrelated FLS settings.
- Trial orgs may not have the Einstein LLM runtime entitlement provisioned even when Setup → Einstein shows enabled. Symptom: every prompt template (including managed defaults like `einstein_gpt__summarizeContact`) fails with `INTERNAL_ERROR: Failed to generate Einstein LLM generations response`. That's a Salesforce-side provisioning gap, not a skill issue.

## License

[Apache License 2.0](LICENSE)

## Contributing

Open an issue or PR. When the skills are ready for upstream contribution, the entire `skills/setting-up-pre-work-brief/` and `skills/customizing-pre-work-brief/` folders will be copied into `forcedotcom/afv-library` via a fork-based PR.

### Adding a new vertical seed

To contribute a seed for a vertical not in the library:

1. Run `customizing-pre-work-brief` with `--industry other` against a real org for that vertical. The skill writes a draft seed at `templates/<your-industry>.md` at the end of the run.
2. Review the draft. Each seed should contain: business archetype, recommended custom objects (with field lists + rationale), standard objects to query, prompt template section structure, vertical-specific rules, cadence example, and test data sample. See `templates/hvac.md` as the canonical reference.
3. Verify the seed end-to-end against the org by re-running with `--industry <your-industry>`.
4. Open a PR adding the seed.
