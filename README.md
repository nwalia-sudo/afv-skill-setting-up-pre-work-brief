# setting-up-pre-work-brief

An [Agentforce Vibes](https://github.com/forcedotcom/afv-library) skill that walks an admin through setting up Pre-Work Brief on Field Service Mobile in a Salesforce org, end to end, mostly via CLI.

This is an early candidate for contribution to `forcedotcom/afv-library`. The folder structure mirrors that repo: `skills/setting-up-pre-work-brief/SKILL.md`. When the skill is ready to merge upstream, the contents drop in unchanged.

## What is Pre-Work Brief

Pre-Work Brief is an Einstein-generative-AI feature in Field Service Mobile. It renders a concise, AI-generated summary of an upcoming Work Order in the **Overview tab** of the mobile app, grounded in real Salesforce data (Work Order, Account, Service Appointment, Work Plans, etc.).

Salesforce documents the manual setup at:

- [Set Up Pre-Work Brief for Field Service Mobile Workers](https://help.salesforce.com/s/articleView?id=service.mfs_einstein_pre_work_brief.htm)
- [Pre-Work Brief and Your Data](https://help.salesforce.com/s/articleView?id=service.mfs_einstein_pre_work_brief_data.htm)
- [Set Up Permissions for Einstein for Field Service Mobile](https://help.salesforce.com/s/articleView?id=service.fs_einstein_gen_ai_setup.htm)

The skill automates as many of the documented steps as can be driven from the Salesforce CLI, and surfaces clear deeplinks for the few that still require a click in Setup.

## What the skill does

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
| 8 | Sets the prompt template Id on a test Work Order (admin's pick or auto-pick) | `sf data update record` |
| 9 | Documents on-device verification | Manual mobile test |

The two steps that still need a click in Setup are:

- **Step 3** — Salesforce's base Einstein generative AI setup (link given).
- **Step 5 activation** — the prompt template's "Activate" button in Prompt Builder. The skill emits a one-time signed deeplink direct to the template editor so it's a single click.

## How to use it

The skill is consumed by an AI coding agent (Claude Code, Agentforce Vibes, Cursor, Codex, etc.) that supports the Agent Skills format. Drop `skills/setting-up-pre-work-brief/SKILL.md` into your skills directory; the agent picks it up via the `description` frontmatter.

Run the skill against an org by asking the agent something like:

> Set up Pre-Work Brief on the `<org-alias>` Salesforce org.

The agent walks the steps in order, asks for the technician's username (or auto-picks), and surfaces the activation deeplink at step 5.

## Prerequisites

- Salesforce CLI (`sf`) v2.x
- A Salesforce org with **Field Service** enabled and the **Einstein for Field Service** or **Agentforce for Field Service** add-on
- An admin user with `Customize Application` and `Manage Profiles and Permission Sets`
- For the on-device test: the Field Service Mobile app on iOS or Android, signed in as a technician with the Field Service Mobile license

## Tested against

`afvuser@salesforce.com` trial org, May 2026. Steps 0 through 8 ran cleanly. Step 9 (mobile-device verification) requires a working Einstein LLM runtime entitlement, which isn't provisioned in all trial orgs — track that as a separate org-provisioning question if you hit "We hit a snag" on mobile.

## Limitations

- **Activation isn't fully automatable.** Salesforce does not currently expose a programmatic activation path for `GenAiPromptTemplate` (we tried Tooling REST PATCH on `IsActive`, Connect API `/activate` endpoints across v62-v66, Apex `ConnectApi.EinsteinLLM` methods, and metadata `activeVersionNumber` — none worked). The skill deploys the template with `<status>Published</status>` and surfaces a deeplink for the one-click activation.
- **Field-level security** for `PreWorkBriefPromptTemplate` is documented as a Setup click rather than a Profile metadata deploy, to avoid stomping on unrelated FLS settings.
- Trial orgs may not have the Einstein LLM runtime entitlement provisioned even when Setup → Einstein shows enabled. Symptom: every prompt template (including managed defaults like `einstein_gpt__summarizeContact`) fails with `INTERNAL_ERROR: Failed to generate Einstein LLM generations response`. That's a Salesforce-side provisioning gap, not a skill issue.

## License

[Apache License 2.0](LICENSE)

## Contributing

Open an issue or PR. When the skill is ready for upstream contribution, the entire `skills/setting-up-pre-work-brief/` folder will be copied into `forcedotcom/afv-library` via a fork-based PR.
