# pp-iam-canvas-starter

> Power Apps Canvas starter for **multi-service IAM request management** — approval workflows, Planner integration, SharePoint backend, and PDF/notes generation. Built from a real enterprise deployment in a large telecom organization.

![Power Apps](https://img.shields.io/badge/Power_Apps-742774?style=flat&logo=powerapps&logoColor=white)
![Power Automate](https://img.shields.io/badge/Power_Automate-0066FF?style=flat&logo=powerautomate&logoColor=white)
![SharePoint](https://img.shields.io/badge/SharePoint-0078D4?style=flat&logo=microsoftsharepoint&logoColor=white)
![Planner](https://img.shields.io/badge/Microsoft_Planner-31752F?style=flat&logo=microsoft&logoColor=white)

---

## Overview

A production-tested Canvas App pattern for organizations that need to manage **identity and access requests** across multiple service types — each with its own form, validation logic, approval flow, and audit trail.

The app supports 4 service request types with independent branching logic, conditional field visibility, and a unified Power Automate flow that routes each request to Microsoft Planner with structured notes.

**Deployed at:** Enterprise telecom organization (~500 users)  
**Sprint duration:** 4 weeks (Phases 1–3)  
**Key integrations:** SharePoint Lists · Microsoft Planner · Power Automate · Power Apps Canvas

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Canvas App (Power Apps)                 │
│                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ Gestores │ │ Clientes │ │   PAM    │ │  M2M/B2B │  │
│  │  Form    │ │  Form    │ │  Form    │ │   Form   │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │
│       └─────────────┴─────────────┴─────────────┘       │
│                          │                              │
│              SubmitForm + .Run(ParametrosFlujo)          │
└──────────────────────────┬──────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │    Power Automate Flow   │
              │  IAM-PowerApps_Planner   │
              │                         │
              │  Trigger: PowerApps V2  │
              │  text_16 → DGP Summary  │
              │  text_17 → SG Summary   │
              │  text_18 → SC Summary   │
              │  text_19 → PAM Summary  │
              │  text_20 → B2B Summary  │
              └──────┬──────────┬───────┘
                     │          │
          ┌──────────▼──┐  ┌────▼──────────┐
          │  SharePoint  │  │    Planner    │
          │     List     │  │  Task + Notes │
          │  (Audit DB)  │  │  per service  │
          └─────────────┘  └───────────────┘
```

---

## Service Types

| Service | Prefix | Description |
|---|---|---|
| **Gestores** | `SG_` | Manager-level identity services — portal web, SSO, APIGW, SM |
| **Clientes** | `SC_` | Client-facing identity services — portal access, API consumption |
| **Accesos Privilegiados (PAM)** | `SAP_` | Privileged access management — Identity Platform, PIM |
| **Consumos M2M/B2B** | `SCMB_` | Machine-to-machine and B2B integration flows |

Each service type has:
- Independent form section with conditional field visibility
- Dedicated SharePoint columns (prefixed)
- Separate Planner task with structured notes
- PDF preview screen (optional, can be disabled)

---

## Key Features

### Multi-branch request form
The app uses a single `Form3` with conditional `DataCard.Visible` and `DataCard.Required` driven by RadioGroup selections. Each service branch is fully independent.

### Planner integration without Premium connectors
All request data is consolidated into **Planner task Notes** using the standard Planner connector (no Graph API / HTTP Premium required). Format:

```
====================================================== DESCRIPCIÓN GENERAL =======================================================
Nombre App: [value]
------...
[fields]
====================================================== RESPUESTAS DEL SERVICIO GESTORES ==========================================
[service-specific fields]
```

### Responsive PDF preview
Each service type has a PDF preview screen built entirely in Power Fx using dynamic height cascading:

```powerfx
// Dynamic label height for long text
Label12_xy.Height = Max(25; RoundUp(Len(Self.Text) / 45; 0) * 18)

// Responsive separator position
rtgSeparator.Y = Label12_xy.Y + Label12_xy.Height + 4
```

### SM / APIGW branch separation
The Gestores service splits into two independent branches based on protocol selection:

| Branch | Protocols | Extra fields |
|---|---|---|
| **SM** | FE-BE architecture, SAML | Volume, schedule, flow description, profiles |
| **APIGW** | API-ficada, OIDC | SSO=Sí: Issuer, JWKS, Provenience, Scopes / SSO=No: URLCallback, ScopesAPIGW |

---

## SharePoint Schema

### Core list: `IAM Service Requests`

| Column | Type | Prefix | Notes |
|---|---|---|---|
| `Title` | Single line | — | Auto-generated request ID |
| `DGP_NombreApp` | Single line | DGP_ | Application name |
| `DGP_AnagramaIMASIMAD` | Single line | DGP_ | Supports "No existe" via RadioGroup |
| `DGP_Descripcion` | **Multi-line** | DGP_ | Long text — must be multi-line |
| `DGP_FechaInicioDeseada` | Date | DGP_ | Optional production date |
| `SG_IDplanner` | **Single line** | SG_ | ⚠️ Must be single-line — used in OData filter |
| `SC_IDPlanner` | **Single line** | SC_ | ⚠️ Same constraint |
| `SAP_IDPlanner` | **Single line** | SAP_ | ⚠️ Same constraint |
| `SCMB_IDPlanner` | **Single line** | SCMB_ | ⚠️ Same constraint |
| `SG_ProtocolosSoportados` | Choice (radio) | SG_ | FE-BE / API-ficada / SAML / OIDC |
| `SG_RBA` | Choice (radio) | SG_ | Sí / No |
| `SC_FlujoAutorizacion` | Choice (radio) | SC_ | 4 options + free text "Otro" |
| `SC_RBA` | Choice (radio) | SC_ | Sí / No |

> **Critical:** All columns used in multi-line TextInput controls must be **Multi-line text** in SharePoint. Single-line columns (255 chars) cause silent `Form3.Valid = false` failures when the user types more than 255 characters.

---

## Power Fx Patterns

### Form validation with RadioGroup (Choice columns)

```powerfx
// WRONG — causes Form3.Valid = false with Choice columns
DataCard.Update = RadioGroup.Selected.Value

// CORRECT
DataCard.Update = {Value: RadioGroup.Selected.Value}

// With "Otro" free text option
DataCard.Update = If(
    Radio2.Selected.Value = "Otro";
    {Value: TextInputOtro.Text};
    {Value: Radio2.Selected.Value}
)
```

### RadioGroup Default (mandatory pattern)

```powerfx
// Always set a Default — without it, Selected.Value = Blank() on load
// and OnChange never fires on first interaction
RadioGroup.Default = {Value: "Option A"}  // for Choices() items
RadioGroup.Default = "Option A"           // for static Items array
```

### Invisible Required fields (critical anti-pattern)

```powerfx
// RULE: Visible = false → Required = false (always)
// Invisible required fields silently block Form3.Valid

// Pattern for conditional required
DataCard.Required = If(
    ParentCondition && OtherCard.Required;
    true;
    false
)

// NEVER check fields unconditionally in submit button
// WRONG:
|| IsBlank(DataCardValue68.Value)

// CORRECT:
|| (IsBlank(DataCardValue68.Value) && DGP_NumeroDRS_DataCard2.Required)
```

### Debug pattern for Form3.Valid

```powerfx
// Add a temporary label with this formula to diagnose which field blocks submission
Set(
    varDebugForm;
    "Valid=" & Text(Form3.Valid) &
    " | Campo1_Req=" & Text(DataCard1.Required) &
    " | Campo1_Blank=" & Text(IsBlank(DataCardValue1.Value)) &
    " | RadioVal=" & RadioGroup.Selected.Value
)
```

### Submit button pattern

```powerfx
If(
    Form3.Valid;  // or: true (when validation is handled in ButtonCanvas3_1)

    // 1. Generate PDF snapshot (optional)
    Set(VarPDFSG; PDF(PDFServicioGestores));;

    // 2. Build flow parameters
    Set(ParametrosFlujo; {
        text_1: If(IsBlank(DataCardValue1.Value); ""; DataCardValue1.Value);
        text_4: User().Email;
        // ... other fields
        file:   {contentBytes: ""; name: ""};  // always initialize file params
        file_1: {contentBytes: ""; name: ""};
        file_2: {contentBytes: ""; name: ""};
        file_3: {contentBytes: ""; name: ""}
    });;

    // 3. Trigger flow
    'IAMServiceRequest-PowerApps_Planner'.Run(
        If(IsBlank(DataCardValue346.Value); ""; DataCardValue346.Value);  // text_16
        If(RadioGroupCanvas1.Selected.Value = "Sí"; ResumenGestores; "");  // text_17
        If(RadioGroupCanvas12.Selected.Value = "Sí"; ResumenClientes; ""); // text_18
        If(RadioGroupCanvas17.Selected.Value = "Sí"; ResumenPAM; "");      // text_19
        If(RadioGroupCanvas23.Selected.Value = "Sí"; ResumenB2B; "");      // text_20
        ParametrosFlujo
    );;

    // 4. Submit form to SharePoint
    SubmitForm(Form3);;

    // 5. UI feedback
    Set(varCerrarFormulario; true);;
    Set(varEnviar; true);;
    Notify("Request submitted successfully"; NotificationType.Success);

    // Else:
    Notify("Please complete all required fields"; NotificationType.Error)
)
```

---

## Power Automate Flow Structure

### Trigger: PowerApps V2

| Parameter | Internal name | Content |
|---|---|---|
| text_16 | ResumenDescripcionGeneral | General description fields |
| text_17 | ResumenGestores | Gestores service fields |
| text_18 | ResumenClientes | Clientes service fields |
| text_19 | ResumenPrivilegiados | PAM service fields |
| text_20 | ResumenM2MB2B | M2M/B2B service fields |

### Per-service block structure

```
Condition: @{triggerBody()?['boolean_X']} is equal to true
├── True:
│   ├── Create item (SharePoint) — log to audit list
│   ├── Create task (Planner) — with title and bucket
│   └── Update task details (Planner) — Notes with concat():
│       concat(
│           '====== DESCRIPCIÓN GENERAL ======', '\n',
│           triggerBody()?['text_16'],
│           '\n\n====== RESPUESTAS DEL SERVICIO X ======', '\n',
│           triggerBody()?['text_17']
│       )
└── False: (skip)
```

> **Architecture decision:** Planner connector does not support native Comment creation. HTTP/Graph API was considered but rejected due to Premium licensing risk in enterprise tenant. All content is consolidated in the task **Notes** field via `Update task details`.

---

## Known Limitations & Design Decisions

| Decision | Reason |
|---|---|
| Planner Notes instead of Comments | Planner connector has no native Create Comment action; Graph API requires Premium license |
| SharePoint IDplanner columns must be Single-line | Multi-line columns cannot be used in OData `$filter` — breaks "cerrar" flows |
| `Form3.Valid` replaced by `true` in some buttons | DataCards with `Gallery/Table()` in `.Update` cause permanent `Form3.Valid = false` |
| PowerApps V2 trigger — no new positional args | Schema sync broken after connector remove/re-add; new values must go via `ParametrosFlujo` record |
| RBA label in Planner not implemented | V2 trigger limitation — text_21 never arrives as direct trigger body field; requires separate SP-triggered flow |

---

## Setup Guide

### Prerequisites
- Power Apps per-user or per-app license
- SharePoint site with owner permissions
- Microsoft Planner — existing plan or create new
- Power Automate standard license (no Premium required)

### Step 1 — SharePoint list
1. Create list: `IAM Service Requests`
2. Add columns following the schema above
3. **Critical:** Set `DGP_Descripcion` and all multi-line fields to **Multiple lines of text**
4. **Critical:** Keep all `*_IDPlanner` columns as **Single line of text**

### Step 2 — Planner
1. Create or identify your Planner plan
2. Create buckets per service type (Gestores, Clientes, PAM, M2M/B2B)
3. Note the Plan ID and Bucket IDs — needed in flow configuration

### Step 3 — Power Automate flow
1. Import the flow template (see `/flows/`)
2. Update SharePoint site URL and list name
3. Update Planner Plan ID and Bucket IDs
4. Publish the flow

### Step 4 — Power Apps
1. Import the app package (see `/app/`)
2. Connect to your SharePoint list
3. Connect to your Power Automate flow
4. Update environment-specific variables in `App.OnStart`
5. Publish and share with users

---

## Folder Structure

```
pp-iam-canvas-starter/
├── README.md
├── docs/
│   ├── architecture.md          # Detailed architecture decisions
│   ├── sharepoint-schema.md     # Full column reference
│   ├── powerfx-patterns.md      # Reusable Power Fx patterns
│   └── known-issues.md          # Troubleshooting guide
├── flows/
│   └── IAM-PowerApps-Planner/   # Flow export (.zip)
├── app/
│   └── IAMRequestApp.msapp      # Canvas App export
└── scripts/
    └── setup-sharepoint.ps1     # PowerShell list setup script
```

---

## Lessons Learned

These were discovered during the enterprise deployment and documented for reuse:

1. **`Form3.Valid` with `Gallery/Table()` in `.Update`** → always causes `false`. Replace `Form3.Valid` with `true` when pre-validation is done in the submit button.
2. **RadioGroup `.Default` is mandatory** when the control substitutes a required field. Without it, `Selected.Value = Blank()` on load and `OnChange` never fires on first interaction.
3. **Choice columns require `{Value: "string"}` in `.Update`**, not a plain string.
4. **Invisible Required fields silently block submission.** Rule: `Visible = false` → `Required = false`, always.
5. **V2 trigger positional args cannot be added post-deploy.** Schema sync is broken even after remove/re-add. Use the `ParametrosFlujo` record for new values.
6. **SharePoint Multi-line columns break OData filters.** Keep ID/reference columns as Single-line.

---

## Contributing

Issues and PRs welcome. This starter is intentionally kept generic — if you adapt it for a specific industry or service type, consider opening a PR with your variant.

---

## License

MIT
