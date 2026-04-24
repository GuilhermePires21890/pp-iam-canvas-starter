# Architecture Decisions

---

## ADR-001 — Planner Notes vs Comments

**Decision:** Use Planner task **Notes** (Description field) instead of Comments.

**Context:** The requirement was to store structured request data on the Planner task so approvers can read the full request without leaving Planner.

**Options considered:**

| Option | Pros | Cons |
|---|---|---|
| Planner Comments | Native UI, familiar to users | No native action in Power Automate connector |
| Graph API (HTTP) | Full Planner API access, supports Comments | Requires HTTP Premium connector — licensing risk in enterprise tenant |
| Planner Notes (Update task details) | Standard connector, no Premium | Plain text only, no rich text via API |

**Decision:** Planner Notes via `Update task details`. Compensate for plain text with structured separators (`======` headers, `------` dividers).

**Consequence:** Approvers see a structured but plain-text summary. Rich text formatting requires manual editing in Planner web UI.

---

## ADR-002 — SharePoint vs Dataverse as backend

**Decision:** SharePoint Lists as primary data store.

**Context:** Enterprise tenant with existing SharePoint infrastructure. Dataverse would require additional licensing and environment provisioning.

**Tradeoffs:**

| Factor | SharePoint | Dataverse |
|---|---|---|
| Licensing | Included in M365 | Requires Power Apps Per User/App |
| Column types | Limited (no polymorphic, no rollup) | Full schema control |
| OData filtering | Supported (with Single-line constraint) | Full FetchXML/OData |
| Form integration | Native `SubmitForm()` | Native with Model-Driven; Canvas requires manual `Patch()` |
| Audit trail | Basic version history | Full audit log with Dataverse auditing |

**Decision driver:** Zero additional licensing cost. SharePoint constraints were manageable for this use case.

---

## ADR-003 — Single Form3 vs multiple forms

**Decision:** Single `Form3` instance shared across all 4 service types, with conditional DataCard visibility.

**Context:** Original design used separate screens per service type. This caused maintenance overhead when shared fields needed updating.

**Decision:** Merge into single Form3. Use `_requestTypeFilter` variable to control which DataCards are visible.

**Pattern:**
```powerfx
DataCard.Visible = _requestTypeFilter = "Service Type Name"
                  && RadioGroupParent.Selected.Value = "Sí"
```

**Consequence:** Form3 has ~100 DataCards. Debugging requires `varDebugForm` pattern. Any change to shared fields only requires one update.

---

## ADR-004 — PDF generation vs Planner Notes

**Decision (Phase 2 change):** Remove PDF generation from submit flow. Replace with structured Planner Notes.

**Original behavior:** Submit button generated PDF → uploaded to SharePoint folder → linked in Planner task.

**Problems with PDF approach:**
- SharePoint folder creation required separate flow action
- PDF files accumulated with no lifecycle management
- Approvers needed to navigate to SharePoint to read the request

**New behavior:** All request data concatenated into Planner task Notes at submission time. No files created.

**Migration impact:** 4 flow actions removed per service block. 4 `Set(VarPDF...; PDF(...))` calls removed from submit buttons.

---

## ADR-005 — V2 trigger parameter strategy

**Decision:** Pass new flow parameters via `ParametrosFlujo` record, not as new positional V2 arguments.

**Context:** After Phase 1 deployment, Phase 2 required passing 5 additional text parameters to the flow.

**Attempted:** Add `text_16` to `text_20` as positional V2 trigger arguments.

**Problem:** PowerApps V2 connector schema does not reliably sync after the app is already connected. New positional arguments show as missing even after remove/re-add of the connector.

**Decision:** Pass new parameters as the first positional arguments in a new `.Run()` call, with the existing record as the last argument.

**New `.Run()` signature:**
```powerfx
FlowName.Run(
    text_16_value;   // new positional
    text_17_value;   // new positional
    text_18_value;   // new positional
    text_19_value;   // new positional
    text_20_value;   // new positional
    ParametrosFlujo  // existing record (last)
)
```

**Lesson:** Design the full trigger schema upfront. Adding positional V2 arguments post-deployment is unreliable.
