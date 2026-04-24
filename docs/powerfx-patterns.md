# Power Fx Patterns — IAM Request Management

Reusable patterns extracted from a production enterprise IAM deployment.

---

## 1. Choice column validation (RadioGroup)

SharePoint Choice columns require a Record in `.Update`, not a plain string.

```powerfx
// WRONG — silent Form3.Valid = false
DataCard.Update = RadioGroup.Selected.Value

// CORRECT
DataCard.Update = {Value: RadioGroup.Selected.Value}

// With free-text "Other" option
DataCard.Update = If(
    RadioGroup.Selected.Value = "Other";
    {Value: TextInputOther.Text};
    {Value: RadioGroup.Selected.Value}
)
```

---

## 2. RadioGroup Default (mandatory)

```powerfx
// For Choices() items:
RadioGroup.Default = {Value: "Option A"}

// For static Items array:
RadioGroup.Default = "Option A"

// Why: without Default, Selected.Value = Blank() on load
// → OnChange never fires on first user interaction
// → control variables stay in wrong initial state
```

---

## 3. Conditional Required pattern

```powerfx
// Safe conditional required
DataCard.Required = If(
    ParentDropdown.Selected.Value = "Expected Value"
    && ParentDataCard.Required;
    true;
    false
)

// Rule: Visible = false → Required = false (always)
// Invisible required fields silently block Form3.Valid
```

---

## 4. Submit button validation

```powerfx
// WRONG — unconditional blank check blocks even hidden fields
|| IsBlank(DataCardValue68.Value)

// CORRECT — only validate if field is actually required
|| (IsBlank(DataCardValue68.Value) && DataCard68.Required)
```

---

## 5. Dynamic height for long text in PDF screens

```powerfx
// Label height — expands for long text
Label.Height = Max(25; RoundUp(Len(Self.Text) / 45; 0) * 18)

// Separator Y — follows the label above
Separator.Y = Label.Y + Label.Height + 4

// For two-column layouts — take the max of both sides
Separator.Y = Max(
    LabelLeft.Y + LabelLeft.Height;
    LabelRight.Y + LabelRight.Height
) + 4
```

---

## 6. Date field in Form3

```powerfx
// Default — handles blank gracefully
DataCard.Default = If(
    IsBlank(ThisItem.DateField);
    Blank();
    ThisItem.DateField
)

// Update
DataCard.Update = DatePickerControl.SelectedDate

// Clear button
Icon.OnSelect = Reset(DatePickerControl)
```

---

## 7. Form3.Valid debug pattern

```powerfx
// Add a temporary label to identify which field blocks submission
Set(
    varDebugForm;
    "Valid=" & Text(Form3.Valid) &
    " | Field1_Req=" & Text(DataCard1.Required) &
    " | Field1_Blank=" & Text(IsBlank(DataCardValue1.Value)) &
    " | Radio=" & RadioGroup.Selected.Value
)
// Remove label after fix is confirmed
```

---

## 8. Planner notes concatenation (Power Automate)

```
// In "Update task details" action — Notes field
concat(
    '====================================================== DESCRIPCIÓN GENERAL =======================================================', '\n',
    triggerBody()?['text_16'],
    '\n\n',
    '----------------------------------------------------------------------------------------------------------------------', '\n',
    '====================================================== RESPUESTAS DEL SERVICIO X =============================================', '\n',
    triggerBody()?['text_17'],
    '\n',
    '----------------------------------------------------------------------------------------------------------------------'
)
```

> Note: Use `triggerBody()?['text_16']` (internal key), NOT `triggerBody()?['ResumenDescripcionGeneral']` (display name). The display name is not reliable as the trigger body key.

---

## 9. Multi-service flow parameters

```powerfx
// Initialize with empty file params — prevents TriggerInputSchemaMismatch
Set(ParametrosFlujo; {
    text_1: If(IsBlank(DataCardValue1.Value); ""; DataCardValue1.Value);
    text_4: User().Email;
    file:   {contentBytes: ""; name: ""};
    file_1: {contentBytes: ""; name: ""};
    file_2: {contentBytes: ""; name: ""};
    file_3: {contentBytes: ""; name: ""}
});;

// Conditionally patch file params for selected services
If(
    RadioGroupService1.Selected.Value = "Yes";
    Set(ParametrosFlujo; Patch(ParametrosFlujo; {
        file: {contentBytes: VarPDFService1; name: "Service1-Request.pdf"}
    }))
)
```

---

## 10. Concat with empty guard (PDF labels)

```powerfx
// Prevents runtime error when collection is empty
If(
    IsEmpty(colCheckboxValues);
    "";
    Left(
        Concat(colCheckboxValues; Value & ", ");
        Len(Concat(colCheckboxValues; Value & ", ")) - 2
    )
)
```
