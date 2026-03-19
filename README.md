# React Implementation Guides

Internal reference docs for the frontend team. This repo covers how key features are implemented in the React + Frappe app — from routing and navigation to forms, workflows, and document sharing.

## Docs

- [Document Sharing](docs/document-sharing-react-implementation.md)
  How document sharing works on the frontend: permission gating via `usePermission`, the `ShareRecord` modal, adding users, and managing per-user read/write permissions using Frappe's share APIs.

- [Linked Field Fetch](docs/linked-field-fetch-react-implementation.md)
  How `fetch_from` field properties auto-populate form fields when a Link field changes. Covers `handleFetchFrom` in `formUtils.js`, the single-API-call pattern, and how to add `fetch_from` to new fields without code changes.

- [Routes & Menu Items](docs/routes-and-menu-items-implementation.md)
  How to add new pages to the app by configuring Routes and Menu Items in Frappe. Covers the 3-level hierarchy (Route → Menu Item → Sub Menu Item), all `page_view` types (List, Custom Page, Report, Sub Menu), form types (Modal vs Page), and role-based visibility.

- [Workflow](docs/workflow-react-implementation.md)
  The full frontend workflow system: detecting workflow-enabled doctypes, fetching state and available actions via `useWorkflowInfo`, rendering `WorkflowActions` buttons, and the differences between `FormPage` and `FormModal` workflow handling.

## Stack

- React + React Router (frontend)
- Frappe (backend — DocTypes, APIs, permissions, workflows)
- frappe-react-sdk (data fetching hooks)
- Ant Design (UI components)
