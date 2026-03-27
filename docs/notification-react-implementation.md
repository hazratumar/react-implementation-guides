# Notification — React Implementation

Internal reference for the team. Covers how the notification bell is implemented on the frontend: real-time updates via socket, fetching notification logs, marking as read, and navigating to the related record.

---

## Architecture at a Glance

```
useAuth()
  └─ provides currentUser.name (used as filter for Notification Log)

Notification
  └─ GET  frappe.client.get_list (Notification Log)  → fetches notifications for current user
  └─ POST frappe.desk.doctype.notification_log.notification_log.mark_as_read
  └─ POST frappe.desk.doctype.notification_log.notification_log.mark_all_as_read

useEventListener("Notification Log")
  └─ socket.emit("doctype_subscribe", "Notification Log")
  └─ socket.on("list_update")  → triggers mutate() on new notifications
```

### Files involved

| File | Role |
|------|------|
| `src/components/layout/Notification.jsx` | Bell icon, dropdown panel, notification list, mark-as-read logic |
| `src/hooks/useEventListener.js` | Subscribes to Frappe socket events for a given doctype |
| `src/hooks/useAuth.js` | Provides `currentUser` — used to filter notifications by user |
| `src/utils/index.js` | `getFormPath` — resolves the frontend route for a doctype |

---

## 1. Fetching Notifications

Notifications are fetched from the `Notification Log` doctype, filtered to the current user and ordered newest-first.

```jsx
const { data, mutate, isValidating, isLoading } = useFrappeGetDocList("Notification Log", {
  fields: ["*"],
  filters: [["for_user", "=", currentUser?.name]],
  orderBy: { field: "creation", order: "desc" },
});
```

Each entry in `data` has the shape:

```json
{
  "name": "NL-00123",
  "subject": "Document <b>REC-001</b> has been shared with you",
  "from_user": "admin@example.com",
  "for_user": "john@example.com",
  "document_type": "Leave Application",
  "document_name": "HR-LAP-0001",
  "read": 0,
  "creation": "2026-03-27 10:45:00"
}
```

The unread count is derived from the fetched list:

```js
const unreadNotifications = data?.filter((item) => item.read === 0).length || 0;
```

---

## 2. Real-Time Updates via Socket

`useEventListener` subscribes to Frappe's socket channel for `Notification Log`. When a new notification is created or updated, `mutate()` is called to re-fetch the list — but only if no request is already in flight.

```jsx
useEventListener("Notification Log", () => {
  if (!isValidating && !isLoading) {
    mutate();
  }
});
```

Internally, `useEventListener` handles the full socket lifecycle:

```js
// hooks/useEventListener.js
socket.emit("doctype_subscribe", doctype);
socket.on("list_update", handler);          // fires when any doc of this type changes

// on reconnect — re-subscribe in case the connection dropped
socket.io.on("reconnect", () => {
  socket.emit("doctype_subscribe", doctype);
});

// cleanup on unmount
return () => {
  socket.off("list_update", handler);
  socket.io.off("reconnect", onReconnect);
  socket.emit("doctype_unsubscribe", doctype);
};
```

The handler only calls the callback when `data.doctype` matches the subscribed doctype, so multiple `useEventListener` calls on different doctypes don't interfere.

---

## 3. Marking Notifications as Read

### Single notification

When a user clicks a notification, it is marked as read before navigating to the related record.

```jsx
const { call: markAsRead } = useFrappePostCall(
  "frappe.desk.doctype.notification_log.notification_log.mark_as_read",
);

const handleNotificationClick = async (item) => {
  if (!item.document_type || !item.document_name) return;

  if (item.name && item.read === 0) {
    await markAsRead({ docname: item.name });
    mutate();
  }

  const formPath = getFormPath(item.document_type);
  if (formPath) {
    navigate(`${formPath}?recordId=${item.document_name}`);
  }

  setOpen(false);
};
```

Navigation only happens if `getFormPath` returns a valid route for the notification's `document_type`. If the doctype has no registered path, the click still marks the item as read but does not navigate.

### All notifications

The "Mark all read" button appears only when there are unread notifications. It calls the bulk endpoint and re-fetches.

```jsx
const { call: markAllAsRead } = useFrappePostCall(
  "frappe.desk.doctype.notification_log.notification_log.mark_all_as_read",
);

const handleMarkAllAsRead = async () => {
  await markAllAsRead();
  mutate();
};
```

---

## 4. Navigation to Related Record

`getFormPath` (from `src/utils/index.js`) maps a Frappe doctype to its frontend route. The notification's `document_name` is appended as a query param.

```js
// utils/index.js
const DOCTYPE_PATHS = {
  // e.g. "Leave Application": { form: "/leave/form", list: "/leave" }
};

export const getFormPath = (doctype) => {
  return DOCTYPE_PATHS[doctype]?.form || "";
};
```

To support navigation from a new doctype's notifications, add an entry to `DOCTYPE_PATHS`:

```js
const DOCTYPE_PATHS = {
  "My Doctype": { list: "/my-doctype", form: "/my-doctype/form" },
};
```

---

## 5. UI Structure

```
Bell icon (Badge showing unread count)
│
└── Dropdown (click to open)
    └── Card (420px wide, 40vh tall)
        └── Tabs
            └── "Notifications (unread/total)"   [Mark all read button]
                └── Scrollable list
                    └── Per notification row
                        ├── Avatar (first letter of from_user) + blue dot if unread
                        ├── subject (rendered as HTML)
                        └── relative timestamp (e.g. "3 minutes ago")
```

Unread notifications are visually distinguished by:
- A blue `Badge` dot on the avatar
- `fontWeight: 600` on the subject text

The dropdown is controlled via `open` state and `onOpenChange` — clicking outside closes it automatically via Ant Design's `Dropdown` behavior.

---

## 6. Timestamp Formatting

Timestamps use `dayjs` with the `relativeTime` plugin:

```js
import dayjs from "dayjs";
import relativeTime from "dayjs/plugin/relativeTime";

dayjs.extend(relativeTime);

dayjs(item.creation).fromNow()  // → "3 minutes ago", "2 hours ago", etc.
```

---

## API Reference

| Frappe API | Method | Purpose |
|------------|--------|---------|
| `frappe.client.get_list` (Notification Log) | GET | Fetches all notifications for the current user |
| `frappe.desk.doctype.notification_log.notification_log.mark_as_read` | POST | Marks a single notification as read by `docname` |
| `frappe.desk.doctype.notification_log.notification_log.mark_all_as_read` | POST | Marks all notifications as read for the current user |
