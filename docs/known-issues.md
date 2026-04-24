# Known Issues & Troubleshooting

---

## Form3.Valid = false (silent failure)

**Symptom:** Submit button shows "Please complete all required fields" even when all fields are filled.

**Causes & Fixes:**

| Cause | Fix |
|---|---|
| SharePoint column is Single-line but receives >255 chars | Change column to **Multiple lines of text** in SharePoint |
| DataCard with `Gallery/Table()` in `.Update` | Replace `Form3.Valid` with `true` in submit button; handle validation manually |
| Invisible DataCard with `Required = true` | Set `Required = false` on all `Visible = false` DataCards |
| DatePicker with Blank value | Set `Default = If(IsBlank(ThisItem.Field); Blank(); ThisItem.Field)` |
| Choice column `.Update` using plain string | Change to `{Value: RadioGroup.Selected.Value}` |

---

## TriggerInputSchemaMismatch

**Symptom:** Flow fails with `"Expected String but got Null"` on file parameters.

**Fix:** Always initialize all file parameters in `ParametrosFlujo` before conditionally patching:

```powerfx
file:   {contentBytes: ""; name: ""};
file_1: {contentBytes: ""; name: ""};
file_2: {contentBytes: ""; name: ""};
file_3: {contentBytes: ""; name: ""}
```

---

## OData filter error on IDplanner fields

**Symptom:** "Cannot use field IDplanner of type Note in filter expression"

**Cause:** `IDplanner` column was changed to Multi-line text, which SharePoint does not allow in OData `$filter`.

**Fix:** Revert `*_IDPlanner` columns to **Single line of text**. Planner IDs are short strings — multi-line is never needed.

---

## New V2 trigger arguments not syncing

**Symptom:** New positional argument added to PowerApps V2 trigger shows error `"Required properties are missing from object: text_21"` in the app.

**Cause:** The PowerApps V2 connector does not reliably sync schema after remove/re-add.

**Fix:** Do not add new positional arguments after the flow is already connected to the app. Pass new values inside the `ParametrosFlujo` record instead.

**Alternative for values that must be resolved server-side:** Use a separate "Get items" action in the flow to read from SharePoint after submission.

---

## RadioGroup blocks validation on first interaction

**Symptom:** Validation fails silently on first page load or first form open, then works after interacting with controls.

**Cause:** RadioGroup has no `Default` set → `Selected.Value = Blank()` → `OnChange` never fires → control variable stays in wrong initial state.

**Fix:**
```powerfx
// For Choices() items
RadioGroup.Default = {Value: "Default Option"}

// For static array
RadioGroup.Default = "Default Option"
```

---

## PDF labels showing blank values

**Symptom:** PDF preview screen shows empty labels for fields that have values in the form.

**Common causes:**
1. Label `Text` references old dropdown option names that no longer exist in SharePoint
2. Collection-based label using `Concat(colXXX; ...)` but collection is empty (not populated in PDF preview context)
3. Gallery replaced by RadioGroup but label still references the old Gallery control

**Fixes:**
- Update label text conditions to match current SharePoint option values
- Wrap `Concat` with `If(IsEmpty(col); ""; Left(Concat(...)))`
- Update labels referencing old Gallery to use `RadioGroup.Selected.Value`

---

## Planner task notes not populated

**Symptom:** Planner task is created but Notes field is empty.

**Cause:** Flow `concat()` uses display name instead of internal key.

**Fix:** In Power Automate `Update task details`, use:
```
triggerBody()?['text_16']   ✅
triggerBody()?['ResumenDescripcionGeneral']   ❌
```

---

## "Notify() corrupts flow parameter values"

**Symptom:** After adding a `Notify()` call for debugging, flow receives wrong values.

**Cause:** `Notify()` in Power Fx interferes with variable state when placed inline in complex `Set()` chains.

**Fix:** Remove all `Notify()` debug calls before testing flow integration.
