# Routes & Menu Items — Implementation

Internal reference for the team. Covers how to add new pages to the app by configuring Routes and Menu Items in Frappe, and how those configurations map to React components on the frontend.

---

## How It All Connects

The app uses a database-driven routing system. There is no hardcoded route list in the frontend. Instead:

1. Routes and Menu Items are configured in Frappe (via the admin UI or fixtures)
2. The frontend fetches them at runtime via `get_parent_and_child_data`
3. `RoutesProvider` normalizes the data and exposes it through `RouteContext`
4. `routeFactory.jsx` converts the data into React Router `<Route>` elements dynamically

```
Frappe DB (Routes + Menu Items)
       |
       v
GET xpert_app.layout.doctype.routes.api.get_parent_and_child_data
       |
       v
RoutesProvider → RouteContext
       |
       v
routeFactory.jsx → <Route> elements
       |
       v
Sidebar renders navigation from the same RouteContext data
```

### What the API returns

`get_parent_and_child_data` returns an array of Route objects, each with a `subRoutes` array built by `get_menu_items_optimized`. Menu Items with a `parent_menu_item` set are nested under their parent as `subMenu`. The full shape looks like:

```json
[
  {
    "name": "Planning",
    "path": "/planning",
    "page_view": "Menu",
    "sequence_number": 3,
    "roles": ["Manager", "User"],
    "tabs": [],
    "subRoutes": [
      {
        "path": "project-concepts",
        "label": "Project Concepts",
        "page": "Project Concept I",
        "page_view": "List",
        "form_type": "Modal",
        "form_title": "Add Project Concept",
        "update_form_title": "Edit Project Concept",
        "add_button_title": "Add New",
        "cancel_button_title": "Cancel",
        "update_button_title": "Update",
        "modal_height": null,
        "modal_width": null,
        "tabs": [],
        "subMenu": []
      }
    ]
  }
]
```

Items with `is_hidden: 1` are excluded from the query entirely. Items whose roles don't match the current user's roles are filtered out before the response is returned.

---

## Route Hierarchy — 3 Levels

The system supports up to 3 levels of navigation:

```
Level 1 — Route          (page_view: "Menu")
  └─ Level 2 — Menu Item (page_view: "List" | "Sub Menu" | "Custom Page" | "Report")
       └─ Level 3 — Sub Menu Item (page_view: "List" | "Custom Page" | "Report")
```

URL structure mirrors the hierarchy:

```
/planning                               ← Level 1 Route
/planning/project-concepts              ← Level 2 Menu Item
/planning/project-concepts/active       ← Level 3 Sub Menu Item
```

Level 3 items are Menu Items with `parent_menu_item` set to a Level 2 `Sub Menu` item. The backend groups them under `subMenu` on the parent. There is no Level 4 — `Sub Menu` is only valid at Level 2.

---

## Page View Types

`page_view` determines what React component gets rendered for a route.

| page_view | Available on | Renders | Auto-creates /form and /view? |
|-----------|-------------|---------|-------------------------------|
| `Menu` | Routes only | `<MenuRoute>` — redirects to first child | No |
| `Sub Menu` | Menu Items only | `<SubMenuRoute>` — redirects to first child | No |
| `List` | Both | `<RenderTable>` — doctype list with actions | Yes |
| `Custom Page` | Both | Custom React component via `CUSTOM_PAGE_MAP` | No |
| `Report` | Both | `<ReportsWrapper>` | No |

`Menu` is only available on Routes (Level 1). `Sub Menu` is only available on Menu Items (Level 2). All other types are available on both.

---

## Step 1 — Create a Route (Level 1)

A Route is the top-level sidebar entry. It acts as a container for Menu Items.

Go to: **Frappe Admin → Routes → New**

| Field | Value | Required | Notes |
|-------|-------|----------|-------|
| Label | `Planning` | Yes | Displayed in sidebar. Must be unique. |
| Path | `planning` | Yes | URL segment — no slashes, no spaces. Must be unique. |
| Page View | `Menu` | Yes | Always `Menu` for top-level routes |
| Icon | `DashboardIcon` | Yes | Select from available icons (see icon list below) |
| Colour | `Light Green` | No | Sidebar accent colour — defaults to `Light Green` |
| Sequence Number | `3` | No | Controls sidebar order — lower numbers appear first |
| Visibility | `Public` | No | `Private` hides from sidebar but allows direct URL access |
| Is Hidden | unchecked | No | When checked, excluded from the API entirely |
| Roles | `[Manager, User]` | Yes | User must have at least one matching role to see this route |

**Available icons:** `DashboardIcon`, `CommunityEngage`, `Logbook`, `SocialProfiling`, `Survey`, `Settings`, `Reimbursement`, `DataImport`, `Report`

A Route with `page_view: Menu` renders `<MenuRoute>`, which automatically redirects to its first Menu Item when visited directly.

---

## Step 2 — Create a Menu Item (Level 2)

Menu Items are the second-level entries linked to a Route.

Go to: **Frappe Admin → Menu Items → New**

All Menu Items require:

| Field | Required | Notes |
|-------|----------|-------|
| Route | Yes | Link to the parent Route |
| Label | Yes | Sidebar label |
| Path | Yes | URL segment — kebab-case, must be unique across all Menu Items |
| Page View | Yes | Determines what renders |
| Order | No | Position within the route — lower numbers appear first |
| Visibility | No | `Public` (default) / `Private` |
| Is Hidden | No | Excludes from API entirely when checked |
| Roles | No | If empty, item is not shown to anyone |

Additional fields depend on the `page_view` chosen.

---

### Option A — List with Form Modal

Renders a doctype list. The Add button and row Edit action open a `FormModal` overlay.

| Field | Value | Notes |
|-------|-------|-------|
| Page View | `List` | |
| Page | `Project Concept I` | The Frappe DocType to list. Required when `page_view` is `List`. |
| Form Type | `Modal` | Opens add/edit in an Ant Design modal |
| Form Title | `Add Project Concept` | Modal title when creating a new record |
| Update Form Title | `Edit Project Concept` | Modal title when editing an existing record |
| Add Button Title | `Add New` | Label on the Add button in the list header |
| Update Button Title | `Update` | Label on the Save button inside the modal |
| Cancel Button Title | `Cancel` | Label on the Cancel button. Defaults to `Cancel` if left blank. |
| Modal Width | `1000px` | Optional. Defaults to `850px`. Accepts `px` or `vh` units. |
| Modal Height | `600px` | Optional. Defaults to `auto`. Controls the modal body scroll height. |

`FormModal` reads all of these from `currentPageData` at runtime:

```jsx
// FormModal.jsx
const { currentPageData } = useContext(RouteContext);
const doctype     = doctypeProp || currentPageData?.page;
const formTitle   = getFormTitle(type, savedRecordId, isReadOnly, currentPageData);
const modalHeight = formatModalDimensions(currentPageData?.modal_height) || "auto";
const modalWidth  = formatModalDimensions(currentPageData?.modal_width)  || "850px";
```

`getFormTitle` returns `form_title` for ADD mode and `update_form_title` for EDIT mode.

---

### Option B — List with Form Page

Same as Option A but the form opens as a full-page route instead of a modal. Use this for wide or complex forms.

| Field | Value | Notes |
|-------|-------|-------|
| Page View | `List` | |
| Page | `Project Concept I` | |
| Form Type | `Page` | Navigates to `/<path>/form` instead of opening a modal |
| Form Title | `Add Project Concept` | Page title for new records |
| Update Form Title | `Edit Project Concept` | Page title for editing |
| Add Button Title | `Add New` | |
| Update Button Title | `Update` | |
| Cancel Button Title | `Cancel` | |

`modal_width` and `modal_height` are ignored for `Page` form type.

`routeFactory.jsx` automatically creates `/form` and `/view` child routes for every `List` menu item, regardless of `form_type`:

```jsx
// routeFactory.jsx — auto-created for every List page_view
<Route path={`${path}/form`} element={<FormPage />} />
<Route path={`${path}/view`} element={<ViewPage />} />
```

`FormPage` reads `recordId` from the query string to determine ADD vs EDIT mode:

```jsx
// FormPage.jsx
const [searchParams] = useSearchParams();
const recordId = searchParams.get("recordId");
const type = recordId ? "EDIT" : "ADD";

// After cancel, navigates back to the list
const listViewPath = pathname?.split("/form")[0] || "/";
```

Clicking Add → navigates to `/<path>/form`
Clicking Edit → navigates to `/<path>/form?recordId=<name>`

---

### Option C — Custom Page

Renders a custom React component instead of a list. Used for dashboards, maps, calendars, and other non-standard views.

| Field | Value | Notes |
|-------|-------|-------|
| Page View | `Custom Page` | |
| Custom Page | `NA Profiling Dashboard` | Must exactly match a key in `CUSTOM_PAGE_MAP` in `getCustomPage.jsx` |

`routeFactory.jsx` calls `getCustomPage(routeProps)` which looks up the component by name:

```jsx
// routeFactory.jsx
if (page_view === PAGE_VIEW.CUSTOM_PAGE) {
  return getCustomPage(routeProps);
}
```

```jsx
// getCustomPage.jsx
const CUSTOM_PAGE_MAP = {
  "Refusal Social Profile":   RefusalSocialProfile,
  "NA Profiling Dashboard":   NAProfilingDashboard,
  "Social Profile Dashboard": SocialProfileDashboard,
  "NA Calendar":              NACalendar,
  "CE Activities":            CEActivitiesDashboard,
  "Validation Dashboard":     ValidationDashboard,
  "Image Report":             ImageReport,
  "Activities Map":           ActivitiesMap,
};
```

If the key is not registered, the user sees a 404 result:
> "The custom page 'My Feature' is not registered. Add it to CUSTOM_PAGE_MAP in getCustomPage.jsx"

---

### Option D — Report

Renders a Frappe query report.

| Field | Value | Notes |
|-------|-------|-------|
| Page View | `Report` | |
| Report Name | `Monthly Activity Summary` | Required when `page_view` is `Report`. Must match the Frappe report name exactly. |

---

### Option E — Sub Menu (Level 2 container)

Groups multiple Level 3 items under a collapsible sidebar entry. Does not render any content itself — it redirects to its first child.

| Field | Value | Notes |
|-------|-------|-------|
| Page View | `Sub Menu` | |
| Label | `Project Concepts` | Sidebar label for the group |
| Path | `project-concepts` | URL segment |

A `Sub Menu` item renders `<SubMenuRoute>`, which redirects to its first child when visited directly. `form_type`, `page`, and all label fields are ignored.

---

## Step 3 — Create Sub Menu Items (Level 3)

Sub Menu Items are Menu Items with `parent_menu_item` set to a Level 2 `Sub Menu` item. They support the same `page_view` options as Level 2 except `Sub Menu` (no 4th level).

Go to: **Frappe Admin → Menu Items → New**

| Field | Value | Notes |
|-------|-------|-------|
| Route | `Planning` | Same parent Route as the Sub Menu item |
| Parent Menu Item | `Project Concepts` | Link to the Level 2 `Sub Menu` item |
| Label | `Active Projects` | |
| Path | `active-projects` | |
| Page View | `List` | |
| Page | `Project Concept I` | |
| Form Type | `Modal` or `Page` | |

The backend groups these under `subMenu` on the parent item. `routeFactory.jsx` renders them as nested `<Route>` children inside the `<SubMenuRoute>`.

The resulting URL is: `/planning/project-concepts/active-projects`

---

## Form Type — Modal vs Page vs Custom vs None

`form_type` controls how the Add/Edit form opens for `List` pages. It is only relevant when `page_view` is `List`.

| form_type | Behavior | When to use |
|-----------|----------|-------------|
| `Modal` | Add/Edit opens in an Ant Design modal overlay | Default. Most forms. |
| `Page` | Add/Edit navigates to `/<path>/form` full page route | Wide or complex forms needing full viewport |
| `Custom` | Custom component handles the form via `CustomButton` | Special cases with non-standard add/edit flows |
| `None` | No Add/Edit button is shown | Read-only lists |

Both `FormPage` and `FormModal` read `currentPageData` from `RouteContext` to get the doctype, labels, and dimensions:

```jsx
// Inside FormPage.jsx and FormModal.jsx
const { currentPageData } = useContext(RouteContext);
const doctype = currentPageData?.page;
```

---

## Adding a Custom Page (Code Change Required)

Custom pages require registering the React component in `getCustomPage.jsx`. This is the only feature that requires a code change.

**Step 1 — Build the component**

```jsx
// src/pages/myFeature/index.jsx
const MyFeaturePage = () => {
  return <div>My Feature</div>;
};

export default MyFeaturePage;
```

**Step 2 — Register it in `CUSTOM_PAGE_MAP`**

```jsx
// src/components/utils/getCustomPage.jsx
import MyFeaturePage from "../../pages/myFeature";

const CUSTOM_PAGE_MAP = {
  // existing entries...
  "My Feature": MyFeaturePage,   // key must match the "Custom Page" field in Frappe
};
```

**Step 3 — Create the Menu Item in Frappe**

| Field | Value |
|-------|-------|
| Page View | `Custom Page` |
| Custom Page | `My Feature` |

The string in the `Custom Page` field must exactly match the key in `CUSTOM_PAGE_MAP`.

Custom page components receive all route props and can also read `currentPageData` from `RouteContext`:

```jsx
import { useContext } from "react";
import { RouteContext } from "../../contexts/RouteContext";

const MyFeaturePage = (props) => {
  const { currentPageData } = useContext(RouteContext);
  // currentPageData.custom_page → "My Feature"
  return <div>{currentPageData?.custom_page}</div>;
};
```

---

## How `currentPageData` Is Resolved

`RoutesProvider` matches the current URL path against the route tree and sets `currentPageData` to the deepest matching node. Every page component (`FormPage`, `FormModal`, `RenderTable`, custom pages) reads its configuration from here.

```
URL: /planning/project-concepts
  → currentRoute    = { path: "/planning", ... }
  → currentSubRoute = { path: "project-concepts", page: "Project Concept I", form_type: "Modal", ... }
  → currentPageData = currentSubRoute

URL: /planning/project-concepts/active-projects
  → currentSubMenu  = { path: "active-projects", page: "Project Concept I", ... }
  → currentPageData = currentSubMenu
```

`currentPageData` shape:

```js
{
  page:                "Project Concept I",   // Frappe DocType name
  page_view:           "List",
  form_type:           "Modal",
  form_title:          "Add Project Concept",
  update_form_title:   "Edit Project Concept",
  add_button_title:    "Add New",
  update_button_title: "Update",
  cancel_button_title: "Cancel",
  modal_height:        "600px",
  modal_width:         "1000px",
  custom_page:         null,
  report_name:         null,
  colour:              "Light Green",
  visibility:          "Public",
  is_hidden:           0,
  tabs:                [...],
}
```

---

## Path Naming Rules

Paths are normalized to kebab-case by `RoutesProvider` before being used as URL segments.

```
"Project Concepts"  →  "project-concepts"
"NA Calendar"       →  "na-calendar"
"CE Activities"     →  "ce-activities"
```

Best practice: enter paths already in kebab-case in Frappe to avoid ambiguity.

- No leading or trailing slashes
- No spaces — use hyphens
- Must be unique across all Menu Items (the `path` field has a unique constraint)

---

## Role-Based Visibility

Routes and Menu Items both have a `Roles` table. A user must have at least one matching role to see the item.

**How it works end-to-end:**

1. `get_parent_and_child_data` queries `Routes Roles` and only returns Routes where the current user's roles match. Routes with no matching role are excluded entirely.
2. `get_menu_items_optimized` does the same for Menu Items — it queries `Routes Roles` with `parenttype: "Menu Items"` and only includes items where the user has a matching role.
3. On the frontend, `RoutesProvider` builds `currentPageData` from the already-filtered data, so no additional role checks are needed in components.

**Visibility vs Is Hidden:**

| Setting | Effect |
|---------|--------|
| `visibility: Public` | Shown in sidebar (default) |
| `visibility: Private` | Hidden from sidebar, but direct URL access still works |
| `is_hidden: 1` | Excluded from the API response entirely — no access at all |

---

## Quick Reference — Field Cheatsheet

### Routes (Level 1)

| Field | Required | Type | Purpose |
|-------|----------|------|---------|
| `label` | Yes | Data | Sidebar display name. Must be unique. |
| `path` | Yes | Data | URL segment (kebab-case). Must be unique. |
| `page_view` | Yes | Select | Always `Menu` for Level 1 |
| `icon` | Yes | Select | Sidebar icon |
| `colour` | No | Select | Sidebar accent colour. Defaults to `Light Green`. |
| `sequence_number` | No | Int | Sidebar order — lower = higher up |
| `roles` | Yes | Table | Access control — user needs at least one match |
| `visibility` | No | Select | `Public` / `Private` |
| `is_hidden` | No | Check | Hides from API entirely |
| `form_title` | No | Data | Add form title (if Route itself has a form) |
| `update_form_title` | No | Data | Edit form title |
| `add_button_title` | No | Data | Add button label |
| `update_button_title` | No | Data | Save button label |
| `cancel_button_title` | No | Data | Cancel button label. Defaults to `Cancel`. |
| `modal_height` | No | Data | Modal body height (e.g. `600px`, `60vh`) |
| `modal_width` | No | Data | Modal width (e.g. `1000px`). Defaults to `850px`. |

### Menu Items (Level 2 & 3)

| Field | Required | Type | Purpose |
|-------|----------|------|---------|
| `route` | Yes | Link → Routes | Parent Route |
| `label` | Yes | Data | Sidebar display name |
| `path` | Yes | Data | URL segment (kebab-case). Must be unique across all Menu Items. |
| `page_view` | Yes | Select | `List` / `Sub Menu` / `Custom Page` / `Report` |
| `page` | If List | Link → DocType | Frappe DocType to list |
| `custom_page` | If Custom Page | Data | Key in `CUSTOM_PAGE_MAP` |
| `report_name` | If Report | Data | Frappe report name |
| `parent_menu_item` | If Level 3 | Link → Menu Items | Parent Sub Menu item |
| `form_type` | No | Select | `Modal` / `Page` / `Custom` / `None` |
| `form_title` | No | Data | Add modal/page title |
| `update_form_title` | No | Data | Edit modal/page title |
| `add_button_title` | No | Data | Add button label |
| `update_button_title` | No | Data | Save button label |
| `cancel_button_title` | No | Data | Cancel button label. Defaults to `Cancel`. |
| `modal_height` | No | Data | Modal body height (e.g. `600px`) |
| `modal_width` | No | Data | Modal width (e.g. `1000px`). Defaults to `850px`. |
| `colour` | No | Select | Accent colour |
| `order_sequence` | No | Int | Position within parent — lower = higher up |
| `visibility` | No | Select | `Public` (default) / `Private` |
| `is_hidden` | No | Check | Hides from API entirely |
| `roles` | No | Table | Access control — if empty, item is not shown |
