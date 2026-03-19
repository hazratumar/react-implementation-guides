# Workflow — React Implementation

Internal reference for the team. Covers the full frontend workflow system: how it detects workflow-enabled documents, fetches state, renders actions, and handles the two form surfaces (page and modal).

---

## Architecture at a Glance

```
RenderForm
  └─ detects workflow_state field → setHasWorkflow(true)

FormPage / FormModal
  └─ useWorkflowInfo(doctype, recordId)
       └─ GET xpert_app.apis.workflow.get_workflow_info
  └─ renders WorkflowActions in footer (or WorkflowStateBadge in title)

WorkflowActions
  └─ useWorkflowApi()
       └─ POST xpert_app.apis.workflow.run_workflow_action
  └─ on success → mutate() + onSuccess()
```

### Files involved

| File | Role |
|------|------|
| `src/hooks/useWorkflow.js` | Data fetching (`useWorkflowInfo`) and action execution (`useWorkflowApi`) |
| `src/components/common/WorkflowActions.jsx` | Renders one button per available workflow action |
| `src/components/common/WorkflowStateBadge.jsx` | Displays current workflow state as a tag |
| `src/components/form/RenderForm.jsx` | Detects workflow presence from form metadata |
| `src/components/form/FormPage.jsx` | Full-page form — orchestrates footer button visibility |
| `src/components/modal/FormModal.jsx` | Modal form — same logic, different container |
| `src/components/drawer/DrawerHeader.jsx` | Drawer/tab context — controls workflow Add button |
| `xpert_app/apis/workflow.py` | Backend API endpoints |

---

## 1. Detecting Workflow Presence

`RenderForm.jsx` inspects the form metadata returned by the API. If the field list includes `workflow_state`, the document type has a workflow attached.

```jsx
// RenderForm.jsx
useEffect(() => {
  const docMeta = data?.message?.[0];
  if (!docMeta) return;

  if (setHasWorkflow && docMeta.properties) {
    const hasWorkflow = docMeta.properties.some((f) => f.fieldname === "workflow_state");
    setHasWorkflow(hasWorkflow);
  }
}, [data, setHasWorkflow]);
```

The result is lifted to the parent (`FormPage` or `FormModal`) via the `setHasWorkflow` callback prop. This is the gate that controls whether any workflow logic runs at all.

---

## 2. Hooks — Fetching Info and Running Actions

Both hooks live in `src/hooks/useWorkflow.js` and use `frappe-react-sdk` under the hood.

### `useWorkflowInfo`

Fetches the current workflow state and available actions for a document. The `enabled` flag is the key guard — it prevents API calls until the form is confirmed open in EDIT mode with a valid record.

```js
export const useWorkflowInfo = (doctype, name, enabled = true) => {
  const shouldFetch = Boolean(enabled && doctype && name);

  const { data, isLoading, error, mutate } = useFrappeGetCall(
    shouldFetch ? "xpert_app.apis.workflow.get_workflow_info" : null,
    shouldFetch ? { doctype, name } : null,
    shouldFetch ? `workflow-info-${doctype}-${name}` : null,
  );

  return {
    workflowInfo:    data?.message,
    workflowState:   data?.message?.workflow_state,
    workflowActions: data?.message?.actions,
    isLoading,
    error,
    mutate,
  };
};
```

The backend returns:

```json
{
  "doctype": "Project Concept I",
  "name": "PC-0001",
  "workflow_state": "Draft",
  "docstatus": 0,
  "actions": ["Submit for Review", "Reject"],
  "has_workflow": true
}
```

### `useWorkflowApi`

Wraps the POST call to apply a workflow action.

```js
export const useWorkflowApi = () => {
  const { call } = useFrappePostCall("xpert_app.apis.workflow.run_workflow_action");

  const runWorkflowAction = useCallback(
    ({ doctype, name, action, comment = "" }) => call({ doctype, name, action, comment }),
    [call],
  );

  return { runWorkflowAction };
};
```

The backend returns `{ new_state, docstatus }` after applying the transition via Frappe's `apply_workflow()`.

---

## 3. WorkflowActions Component

`WorkflowActions.jsx` renders one button per available action. It is used in both `FormPage` and `FormModal` footers.

```jsx
const handleActionClick = useCallback(async (action) => {
  setLoadingAction(action);
  try {
    await runWorkflowAction({ doctype, name: recordId, action, comment: "" });

    notification.success({
      message: "Workflow Action Applied",
      description: `Action "${action}" has been applied successfully.`,
      placement: "bottom",
      duration: 3,
    });

    mutate();           // re-fetch workflow info → updates available actions
    if (onSuccess) onSuccess();
  } catch (error) {
    notification.error({
      message: error?.exc_type === "ValidationError" ? "Validation Error" : "Failed to Apply Action",
      description: <div dangerouslySetInnerHTML={{ __html: errorMessage }} />,
      placement: "bottom",
      duration: 5,
    });
  } finally {
    setLoadingAction(null);
  }
}, [runWorkflowAction, doctype, recordId, mutate, onSuccess]);
```

While one action is in-flight, all other buttons are disabled to prevent double-submission:

```jsx
workflowActions.map((action) => (
  <Button
    key={action}
    onClick={() => handleActionClick(action)}
    loading={loadingAction === action}
    disabled={loadingAction !== null && loadingAction !== action}
  >
    {action}
  </Button>
))
```

---

## 4. Footer Visibility Logic

Both `FormPage` and `FormModal` share the same conditional logic to decide what appears in the footer. The rules are:

```jsx
// Only fetch workflow info when in EDIT mode and workflow is confirmed present
const shouldFetchWorkflow = type === "EDIT" && hasWorkflow === true;
const { workflowActions } = useWorkflowInfo(doctype, recordId, shouldFetchWorkflow);

// Show workflow action buttons when all of these are true:
// - editing an existing record
// - form has no unsaved changes
// - document has a workflow
// - at least one action is available
const shouldShowWorkflowActions =
  type === "EDIT" && recordId && !isFormTouched && hasWorkflow && workflowActions?.length > 0;

// Show the standard Save/Update button when any of these are true:
// - creating a new record (ADD mode)
// - form has unsaved changes (user is mid-edit)
// - document has no workflow
// - workflow has no available actions
const shouldShowFormButtons =
  !shouldHideUpdateButton &&
  (type === "ADD" || isFormTouched || hasWorkflow === false || workflowActions?.length === 0);
```

The two flags are not mutually exclusive — both can be `false` at the same time (e.g., `Reimbursement Requests` hides the update button entirely via `hideUpdateButtonDoctypes`).

The most important behaviour: **editing a field flips `isFormTouched` to `true`**, which immediately swaps workflow buttons out for the Save button. This prevents applying a workflow action on a dirty form.

---

## 5. FormPage

`FormPage.jsx` is the full-page form route. After a successful workflow action, `onSuccess` triggers a full page reload.

```jsx
// Footer — FormPage.jsx
{shouldShowWorkflowActions && (
  <WorkflowActions
    doctype={doctype}
    recordId={recordId}
    formOpen={true}
    formType={type}
    onSuccess={() => window.location.reload()}
  />
)}
```

The Cancel button is shown whenever either the form buttons or workflow buttons are visible, so the user always has a way out.

---

## 6. FormModal

`FormModal.jsx` runs the same workflow logic inside an Ant Design `Modal`. There are a few differences worth noting.

### Trigger buttons

```jsx
{type === "ADD" ? (
  permissions?.create && (
    <Button type="primary" onClick={showModal}>
      <Plus size={20} /> {buttonTexts.open}
    </Button>
  )
) : (
  <p className="table-menu" onClick={showModal}>
    <SquarePen size={20} /> Edit
  </p>
)}
```

On `showModal`, `isFormTouched` and `savedRecordId` are reset. For ADD mode, `form.resetFields()` is also called to ensure a clean state.

### Workflow state badge in the title

When editing a workflow-enabled record, the current state is shown as a badge next to the modal title via `WorkflowStateBadge`.

```jsx
title={
  <div style={{ display: "flex", alignItems: "center", gap: "12px" }}>
    <label>{formTitle}</label>
    {type === "EDIT" && record?.name && hasWorkflow === true && (
      <WorkflowStateBadge
        doctype={doctype}
        recordName={record.name}
        enabled={open}
      />
    )}
  </div>
}
```

`WorkflowStateBadge` calls `useWorkflowInfo` internally and renders a `StatusTag`. The `enabled={open}` prop ensures it only fetches while the modal is actually open.

```jsx
const WorkflowStateBadge = ({ doctype, recordName, enabled = true }) => {
  const { workflowState } = useWorkflowInfo(doctype, recordName, enabled && !!doctype && !!recordName);
  if (!workflowState) return null;
  return <StatusTag status={workflowState} label={workflowState} />;
};
```

### Footer

The footer is passed as an array to the Modal's `footer` prop. Read-only users get an empty array so no buttons render at all.

```jsx
footer={
  isReadOnly
    ? []
    : [
        (shouldShowFormButtons || shouldShowWorkflowActions) && (
          <Button key="cancel" onClick={handleCancel}>Cancel</Button>
        ),
        shouldShowFormButtons && !isSubmitted && (
          <Button key="submit" type="primary" form={`form-${doctype}-${type}`} htmlType="submit">
            {type === "ADD" && isMultiTabForm && !isLastTab ? "Next" : buttonTexts.save}
          </Button>
        ),
        shouldShowWorkflowActions && (
          <WorkflowActions
            key="workflow"
            doctype={doctype}
            recordId={record?.name}
            formOpen={open}
            formType={type}
            onSuccess={closeModals}
          />
        ),
      ].filter(Boolean)
}
```

`onSuccess` calls `closeModals()` instead of reloading the page — the modal closes and the parent list view refreshes naturally.

### Unsaved changes guard

The modal has its own unsaved changes check. An extra `isCreateAndEmpty` guard prevents the warning from appearing when the user opens an ADD modal, types nothing, and cancels.

```jsx
const handleCancel = ({ skipUnsavedCheck = false } = {}) => {
  if (loading || isUploading) return;

  const touched = form.isFieldsTouched();
  const hasValuesToSave = hasNonEmptyFormValues(form);
  const isCreateAndEmpty = type === "ADD" && !hasValuesToSave;

  const shouldShowUnsaved =
    !isReadOnly && !skipUnsavedCheck && touched && hasValuesToSave && !isCreateAndEmpty;

  shouldShowUnsaved ? setShowUnsaved(true) : closeModals();
};
```

## 7. Drawer / Tab Context

`DrawerHeader.jsx` handles a different scenario: workflow-related tabs on a parent record (e.g., a "Workflow" tab inside a drawer). The Add button visibility depends on three conditions:

- `isWorkflowInitialized` — `initialize_workflow === 1` on the record
- `isRecordSubmitted` — `docstatus === 1`
- `isLastAssignedUser` — current user matches `last_assigned_to`

```jsx
if (isWorkflow && (isWorkflowInitialized || !isRecordSubmitted)) {
  return null; // hide Add button
}

if (isWorkflow && canCreate && isRecordSubmitted) {
  if (!resultObject?.last_assigned_to || isLastAssignedUser) {
    return <FormModal ... initialized={isLastAssignedUser} />;
  }
  return <Button title="Workflow Initialized" />;
}
```

---

## FormPage vs FormModal — Differences

| Concern | FormPage | FormModal |
|---------|----------|-----------|
| Container | Full page route | Ant Design `Modal` |
| Trigger | URL navigation | Button / row menu click |
| After workflow action | `window.location.reload()` | `closeModals()` |
| Unsaved changes modal | Route-level overlay | Nested inside the modal |
| Workflow state display | Not shown | Badge in modal title |
| Read-only footer | Cancel button only | Empty array `[]` |
