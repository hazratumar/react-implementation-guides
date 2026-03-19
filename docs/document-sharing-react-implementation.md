# Document Sharing — React Implementation

Internal reference for the team. Covers how document sharing is implemented on the frontend: permission gating, the share modal, adding users, and managing per-user permissions.

---

## Architecture at a Glance

```
usePermission(doctype)
  └─ GET xpert_app.apis.api.check_permissions
       └─ returns { share: 1 | 0, ... }

DynamicTable
  └─ getMenuItems(record)
       └─ { key: "share", label: <ShareRecord record={record} />, hidden: !permissions?.share }

ShareRecord
  └─ GET frappe.desk.form.load.getdoc  → fetches current shared users (docinfo.shared)
  └─ POST frappe.share.add             → adds a new user to the share list
  └─ POST frappe.share.set_permission  → toggles read/write per user or everyone
  └─ POST frappe.client.validate_link  → validates user exists before adding
```

### Files involved

| File | Role |
|------|------|
| `src/components/common/ShareRecord.jsx` | Share modal — full UI for adding users and managing permissions |
| `src/components/table/DynamicTable.jsx` | Injects `ShareRecord` into the row action menu |
| `src/hooks/usePermission.js` | Fetches user permissions including the `share` flag |
| `src/assets/css/share-record.css` | Styles for the share modal |
| `xpert_app/apis/api.py` | `check_permissions` endpoint — returns `share: 1 | 0` |

---

## 1. Permission Gating

The share option only appears in the row action menu if the current user has the `share` permission on the doctype. This is checked via `usePermission`.

```js
// hooks/usePermission.js
const usePermission = (doctype) => {
  const { data: userPermissions } = useFrappeGetCall(
    "xpert_app.apis.api.check_permissions",
    { doctype },
  );

  const permissions = useMemo(() => ({
    read:   message.read   || 0,
    create: message.create || 0,
    update: message.update || 0,
    delete: message.delete || 0,
    share:  message.share  || 0,   // ← controls share visibility
  }), [userPermissions]);

  return { permissions };
};
```

The backend (`check_permissions`) evaluates `frappe.permissions.has_permission(doctype, ptype="share")` and returns `share: 1` or `share: 0`.

---

## 2. Injecting ShareRecord into the Table

`DynamicTable` builds the row action dropdown via `getMenuItems`. The share item is hidden when `permissions.share` is falsy.

```jsx
// DynamicTable.jsx
import ShareRecord from "../common/ShareRecord";

const getMenuItems = useCallback((record) => {
  return [
    {
      key: "share",
      label: <ShareRecord record={record} />,
      hidden: !permissions?.share,   // hidden if user lacks share permission
    },
    // ...other items (update, view, delete)
  ].filter((item) => !item.hidden);
}, [permissions?.share]);
```

The `record` prop passed to `ShareRecord` only needs `record.name` — the doctype is pulled from `RouteContext` inside the component.

---

## 3. ShareRecord Component

`ShareRecord` is a self-contained modal. It handles the full sharing lifecycle: fetching current shares, adding users, and toggling permissions.

### State and API calls

```jsx
const ShareRecord = ({ record }) => {
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [selectedUser, setSelectedUser] = useState(null);

  const { currentPageData } = useContext(RouteContext);
  const doctype = currentPageData?.page;

  // Fetch current shared users — only when modal is open
  const { data, isLoading, mutate } = useFrappeGetCall(
    "frappe.desk.form.load.getdoc",
    { doctype, name: record?.name },
    isModalOpen && !!record?.name && !!doctype ? `getdoc-${doctype}-${record?.name}` : null,
  );

  const { call: setPermission } = useFrappePostCall("frappe.share.set_permission");
  const { call: addUser }       = useFrappePostCall("frappe.share.add");
  const { call: validateUser }  = useFrappePostCall("frappe.client.validate_link");

  const sharedUsers = useMemo(() => data?.docinfo?.shared || [], [data?.docinfo?.shared]);
};
```

The cache key `getdoc-${doctype}-${record?.name}` is set to `null` when the modal is closed, so no background fetching happens.

`data.docinfo.shared` is an array of share entries returned by Frappe:

```json
[
  { "user": "", "everyone": 1, "read": 1, "write": 0 },
  { "user": "john@example.com", "everyone": 0, "read": 1, "write": 1 }
]
```

---

## 4. Adding a User

Before adding, the user is validated against the `User` doctype via `frappe.client.validate_link`. If validation fails, the selection is cleared and an error notification is shown.

```jsx
const handleUserSelect = useCallback(async (user) => {
  if (!user) { setSelectedUser(null); return; }

  try {
    await validateUser({ doctype: "User", docname: user, fields: [] });
    setSelectedUser(user);
  } catch {
    notification.error({ message: "Invalid User", placement: "bottom" });
    setSelectedUser(null);
  }
}, [validateUser]);

const handleAddUser = useCallback(async (user) => {
  if (!user) return;

  await addUser({
    doctype,
    name: record.name,
    user,
    read: 0,
    write: 0,
    submit: 0,
    share: 0,
    notify: 1,   // sends a notification email to the added user
  });

  setSelectedUser(null);
  mutate();   // re-fetch shared users list
}, [addUser, doctype, mutate, record.name]);
```

New users are added with all permissions set to `0` by default. Permissions are then set individually via the checkboxes in the Manage Permissions section.

The user selector uses `DynamicSelect` with a filter that restricts results to enabled users with the `DO` designation:

```jsx
<DynamicSelect
  doctype="User"
  placeholder="Choose a user to add..."
  value={selectedUser}
  onChange={handleUserSelect}
  filter_name="enabled,user_designation"
  filter_value="1,DO"
/>
```

---

## 5. Managing Permissions

Permissions are toggled per user (or for everyone) via `frappe.share.set_permission`. The supported permission types are `read` and `write`.

```jsx
const PERMISSIONS = ["read", "write"];

const handlePermissionChange = useCallback(
  async (user, permission, value, isEveryone = false) => {
    await setPermission({
      doctype,
      name: record.name,
      user: isEveryone ? "" : user,   // empty string = everyone
      permission_to: permission,
      value: value ? 1 : 0,
      everyone: isEveryone ? 1 : 0,
    });

    mutate();
  },
  [doctype, record, setPermission, mutate],
);
```

The "Everyone" row uses `user: ""` and `everyone: 1`. Individual user rows pass the user's email and `everyone: 0`.

Current permission state is read from `sharedUsers`:

```jsx
const getPermissionValue = useCallback(
  (user, permission, isEveryone = false) => {
    if (isEveryone) return everyonePermissions[permission] === 1;
    const userPerm = sharedUsers.find((u) => u.user === user);
    return userPerm?.[permission] === 1;
  },
  [sharedUsers, everyonePermissions],
);
```

---

## 6. Modal UI Structure

```
Modal: Share "RECORD-NAME"
│
├── Add New User
│   ├── DynamicSelect (User picker, filtered to enabled DO users)
│   └── Add User button (disabled until a valid user is selected)
│
├── ─────────────────────────────────────
│
└── Manage Permissions
    ├── Header row: User | Read | Write
    ├── Everyone row (everyone: 1)
    └── Individual user rows (one per shared user)
```

The entire modal is wrapped in a `<Spin>` that activates during any of the four API calls (`isLoading`, `settingPermission`, `addingUser`, `validatingUser`), preventing interaction while a request is in flight.

---

## 7. Loading States

All four API operations have individual loading flags that feed into the global `<Spin>`:

```jsx
<Spin spinning={isLoading || settingPermission || addingUser || validatingUser}>
  {/* modal content */}
</Spin>
```

The Add User button also has its own `loading` and `disabled` state:

```jsx
<Button
  loading={addingUser}
  disabled={!selectedUser || addingUser || validatingUser}
  onClick={() => handleAddUser(selectedUser)}
>
  Add User
</Button>
```

---

## API Reference

| Frappe API | Method | Purpose |
|------------|--------|---------|
| `xpert_app.apis.api.check_permissions` | GET | Returns `share: 1\|0` for the current user |
| `frappe.desk.form.load.getdoc` | GET | Fetches `docinfo.shared` — current share list |
| `frappe.share.add` | POST | Adds a user to the document's share list |
| `frappe.share.set_permission` | POST | Toggles a specific permission for a user |
| `frappe.client.validate_link` | POST | Validates a user exists before adding |
