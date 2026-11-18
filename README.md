# D.A.T.A. - Intake Portal

> [!CAUTION]
> **Disclaimer:** This accelerator is provided **"as is"**, without warranty of any kind. It is a personal
> sample, **not an official Microsoft product**, and is not supported by Microsoft Support. Neither
> Microsoft nor the author is responsible for how it is used or for any impact on your environment.
> There is **no guarantee of updates, bug fixes or ongoing maintenance.** Read the full
> [DISCLAIMER](DISCLAIMER.md) before use.

An intake app for data requests. Requestors submit requests, team members (assignees) work them, managers
track everything on a dashboard, and DMO approvers make approval decisions. Data is stored in SharePoint
lists, and Power Automate handles the saves.

This repository holds every part of the solution: the working proof-of-concept app, the SharePoint list
design with sample data, the Power Automate flow recipes, and a kit for rebuilding the app as a
Power Apps canvas app.

---

## What is in this repo

| Folder | What it is | Use it when |
|---|---|---|
| `sharepoint-and-flows/` | **Current design (v4).** 6 SharePoint lists: column definitions, a PnP PowerShell script that creates and fills them, sample data, the Power Fx snippets, the 6 flow recipes and a reporting guide. | You are building the real solution. **Start here.** |
| `canvas-rebuild/` | Screen-by-screen build prompts (12 screens), a data model, a provisioning script, a flow build sheet and reference screenshots for rebuilding the app in Power Apps. | You are building the canvas app screens. |
| `app/` | Full source code of the proof-of-concept app (React + TypeScript), with all screens and logic. | You want to see exactly how a screen or rule behaves. |
| `docs/` | Plain-English explanation of why the lists are designed the way they are. | You or a reviewer need the reasoning. |
| `archive/` | Older v3 design, where every read and write goes through flows. Superseded by v4. | Only if you later need "users have no SharePoint access at all". |

> Note: `canvas-rebuild/` was written against an earlier list design (separate question and rule lists).
> Build the screens from it, but use the list and column names in `sharepoint-and-flows/`; v4 wins
> wherever the two differ.

---

## Roles

| Role | What they can do | Home screen |
|---|---|---|
| Requestor | Submit requests, answer questions, follow their own requests | My requests |
| Team Member / Assignee | Work assigned requests, answer team questions, manage tasks | Work queue |
| Manager | Everything a team member can do, plus prioritisation and the dashboard | Dashboard |
| DMO Approver | Approve, reject or send back requests that need approval | Approvals |

Roles come from the `IntakeTeams` list (people columns per team). There is no separate role list.

## Screens

| Screen | Who sees it |
|---|---|
| New request (a multi-step form; questions change by team and request type) | Everyone |
| My requests | Requestor |
| Request details (progress tracker, answers, files, activity) | Everyone, scoped to their rights |
| Tasks / work queue | Assignee, Manager |
| Prioritisation and At-risk | Manager |
| Dashboard | Manager |
| Help | Everyone |

---

## Quick start (build the real solution)

Follow `sharepoint-and-flows/00-README.txt`. In short:

1. **Create a SharePoint site** for the lists (one site, no Teams needed).
2. **Run the script**
   ```powershell
   ./sharepoint-and-flows/04-Create-And-Import-Lists.ps1 `
       -SiteUrl "https://<tenant>.sharepoint.com/sites/<site>" `
       -ClientId "<your-entra-app-client-id>" `
       -EmailDomain "contoso.com" `
       -ReadersLogin "<security group or Everyone except external users>" `
       -ServiceAccount "<svc-account@tenant>"
   ```
   It creates the 6 lists, loads setup data and sample data (`-SkipSampleData` leaves the sample out) and
   sets permissions: users **Read**, service account **Contribute**.
   *Option B:* create the lists by hand from `01-Column-Definitions.csv`.
3. **Edit the setup lists** (`IntakeTeams`, `IntakeRequestTypes`, `IntakeAttributes`) with your real
   teams, people and questions.
4. **Build the 6 flows** from `03-Flow-Recipes.txt`, using the service account connection.
5. **Build the canvas app** screen by screen with `canvas-rebuild/02-prompts/`, pasting the formulas from
   `02-PowerFx-Snippets.txt`. The app reads lists directly and saves through the flows.
6. **Test** with the checklist in `00-README.txt` (one person per role).
7. **Reporting:** see `05-Reporting.txt` (Power BI or Excel).

### The 6 lists

| List | Type | Holds |
|---|---|---|
| IntakeTeams | Setup | Teams and the people in each role |
| IntakeRequestTypes | Setup | Request types per team, approval needed or not |
| IntakeAttributes | Setup | The questions ("attributes") and which teams and types they apply to |
| IntakeRequests | Working | One row per request: answers (JSON text), report columns, current approval, attachments |
| IntakeTasks | Working | Tasks per request, with evidence as attachments |
| IntakeActivity | Working | History: stage changes, comments, approval requests and decisions |

### Security model, in one line

Users can **read** the lists (that is what lets the app show data under their own identity), but they
**cannot edit** them directly. Every save goes through a flow that checks who is calling and what they
are allowed to do, then writes as the service account.

---

## Running the proof-of-concept app source (`app/`)

The app was built with Microsoft App Builder (React 19, TypeScript, Vite, Tailwind, shadcn/ui) and uses the
`@microsoft/managed-apps` runtime to reach SharePoint. It is published as a managed app in Power Platform.
The code is included as a **reference implementation**. Running it outside App Builder takes extra work,
because the SharePoint connection is provided by the managed-apps host.

To run it locally for reading or UI work:

```bash
cd app
bun install        # or: npm install
bun run dev        # or: npm run dev
```

Before connecting it to your own tenant:

- Copy `ms.config.example.json` to `ms.config.json` and fill in your app, environment and connection
  IDs. The real file is git-ignored.
- Copy `.env.example` to `.env` and set your values. Nothing tenant-specific is hard-coded; all of it
  is read from `src/config.ts`:

  | Variable | Example | Used for |
  |---|---|---|
  | `VITE_EMAIL_DOMAIN` | `contoso.com` | Domain for sample and default people |
  | `VITE_SHAREPOINT_HOST` | `contoso.sharepoint.com` | SharePoint host |
  | `VITE_SITE_PATH` | `/sites/intake-requests` | Site that holds the lists and documents |
  | `VITE_DOCUMENT_LIBRARY_ID` | GUID | The site's Documents library |
  | `VITE_DEMO_REQUESTOR_ALIAS` | `mauricioa` | Person the POC Requestor view acts as |
  | `VITE_DEFAULT_MANAGER_ALIAS` | `andrel` | Default manager for new requests |
- `generated/` holds typed models for the **POC list design** (IntakeQuestions, IntakeQuestionRules,
  IntakeSubTeams and others). Those are not the v4 lists; regenerate them if you rebind the app.

Key folders: `src/pages/` (one file per screen), `src/components/` (shared pieces such as the app shell,
question panels and tasks), `src/lib/` (data access, roles, question rules, routing).

---

## Email addresses are a variable

No real email domain is stored in this repo. Wherever a person appears:

| Where | How the domain is written | How it is filled in |
|---|---|---|
| Sample and setup CSVs | `mauricioa@{EmailDomain}` | The list script's `-EmailDomain contoso.com` parameter replaces it on import. Creating lists by hand? Find and replace `{EmailDomain}` in the CSVs first. |
| Provisioning script in `canvas-rebuild/` | `"AndreL@$EmailDomain"` | Its `-EmailDomain` parameter |
| App source | `emailFor("alias")` | `VITE_EMAIL_DOMAIN` in `.env` |

For the sample data to show real people, create accounts with the same aliases in your tenant, or
replace the aliases in the CSVs.

No secrets, client IDs or connection strings are included, and `ms.config.json` is replaced by a redacted
example.

## Known limitations

- The PnP script, Power Fx snippets and flow expressions are written to documented syntax but have **not
  been run** against a live tenant. Test in a dev site first.
- The caller-identity header used by the flows (`x-ms-user-email` or `x-ms-user-email-encoded`) must be
  confirmed from a test run's raw output.
- Anyone with Read access can see internal team answers stored on the request row (accepted trade-off in
  v4; the app hides that tab from requestors).
