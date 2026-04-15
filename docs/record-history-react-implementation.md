# Record History — React Implementation

Internal reference for the team. Covers how record audit history is fetched, normalized, grouped, and rendered in the frontend history drawer.

---

## Architecture at a Glance

```
ViewDrawer
  └─ RecordHistory
       ├─ useFrappeGetCall()
       ├─ xpert_app.apis.document.get_document_history
       ├─ normalize version history entries
       └─ group by calendar day for timeline rendering
```

### Files involved

| File                                                   | Role                                                     |
| ------------------------------------------------------ | -------------------------------------------------------- |
| `xpert_app_ui/src/components/common/RecordHistory.jsx` | Fetches, formats, and renders the audit history timeline |
| `xpert_app_ui/src/components/modal/ViewDrawer.jsx`     | Hosts the History tab inside the record drawer           |
| `xpert_app/apis/document.py`                           | Backend history API used by the component                |

---

## 1. Data Fetching

`RecordHistory.jsx` only fetches data when the drawer/tab is open and both `doctype` and `recordName` are available.

```jsx
const shouldFetchHistory = open && !!doctype && !!recordName;
const swrKey = shouldFetchHistory ? `history-${doctype}-${recordName}` : null;

const { data, isLoading, error, mutate } = useFrappeGetCall("xpert_app.apis.document.get_document_history", { doctype, docname: recordName }, swrKey);
```

That guard keeps the SWR key stable and prevents the hook from fetching when the drawer is closed or the record context is incomplete.

---

## 2. Backend Contract

The frontend expects `get_document_history` to return an object shaped like this:

```json
{
  "doctype": "Some DocType",
  "docname": "REC-0001",
  "created_by": { "email": "user@example.com", "full_name": "User Name" },
  "created_at": "2026-04-15 10:30:00.000000",
  "last_modified_by": { "email": "user@example.com", "full_name": "User Name" },
  "last_modified_at": "2026-04-15 12:15:00.000000",
  "history": [
    {
      "version_id": "...",
      "modified_by": { "email": "...", "full_name": "..." },
      "modified_at": "...",
      "changed_fields": [],
      "added_rows": [],
      "removed_rows": [],
      "row_changes": []
    }
  ]
}
```

On the backend, the API in [xpert_app/apis/document.py](xpert_app/apis/document.py#L390) reads `Version` rows, resolves user display names, and maps system and child-table field labels before returning normalized history entries.

---

## 3. Normalization Logic

The component converts each version row into a render-friendly item.

```jsx
const normalizedHistory = sortedHistory.map((entry) => ({
  key: entry.version_id,
  actionType: "Updated",
  user: entry.modified_by || historyPayload?.last_modified_by,
  timestamp: entry.modified_at,
  content: buildHistoryContent(entry),
  description: buildChangeSummary(entry),
  changedFields: entry.changed_fields || [],
  addedRows: entry.added_rows?.length || 0,
  removedRows: entry.removed_rows?.length || 0,
  rowChanges: entry.row_changes?.length || 0,
}));
```

If the payload includes `created_at`, the component appends a synthetic `Created` entry so the timeline always shows the record origin.

---

## 4. Timeline Grouping

After normalization, the data is grouped by day and sorted newest-first.

```jsx
const groupedTimelineData = timelineData.reduce((groups, item) => {
  const dateKey = dayjs(item.timestamp).isValid() ? dayjs(item.timestamp).format("YYYY-MM-DD") : "unknown-date";

  if (!groups[dateKey]) {
    groups[dateKey] = [];
  }

  groups[dateKey].push(item);
  return groups;
}, {});
```

The visible date label uses `Today`, `Yesterday`, or the weekday name, followed by the formatted date from system settings when available.

---

## 5. Render States

The component has four main states:

1. Loading state with an Ant Design `Spin`.
2. Error state with a retry action.
3. Empty state when no history exists.
4. Rendered timeline with day groups and activity cards.

Each history card shows:

- user avatar and display name
- role or designation when available
- relative timestamp with a tooltip for the full formatted date
- action tag such as `Created` or `Updated`
- field-level changes and row counts when present

---

## 6. Utility Functions

`RecordHistory.jsx` uses small helpers to keep the card rendering readable.

- `stringifyValue` normalizes null, empty, and object values.
- `buildHistoryContent` creates the compact multiline text block.
- `buildChangeSummary` creates the sentence shown under each card title.
- `buildDateLabel` converts a day key into a display label for the section header.

These helpers keep the render function focused on layout instead of data shaping.

---

## 7. Drawer Integration

`ViewDrawer.jsx` mounts `RecordHistory` as the `History` tab.

```jsx
<RecordHistory doctype={doctype} recordName={record?.name} open={open} />
```

The `open` prop is important because it controls whether the fetch runs at all. Closing the drawer destroys the content, so the next open starts from a fresh request.

---

## Behavior Notes

- The history list is sorted by timestamp before grouping, so each date section is internally newest-first.
- If the API returns malformed or missing timestamps, the component falls back to an `Unknown Date` bucket.
- The retry buttons call `mutate()` to re-fetch the same history payload without reloading the drawer.
- The SWR key is derived from `doctype` and `recordName`, so different records get isolated cache entries.
