# Linked Field Value Fetching — React Implementation

Internal reference for the team. Covers how values are automatically fetched from linked (Link) fields and populated into other form fields, using the `fetch_from` field property.

---

## Overview

When a user selects a value in a Link field, other fields in the same form can be automatically populated with data from the linked document. This is driven by the `fetch_from` property on a field definition.

**Example:** Selecting a `District` in a form auto-fills the `Province` field by reading `province` from the selected District document.

The field property looks like this:

```json
{
  "fieldname": "province",
  "fieldtype": "Data",
  "label": "Province",
  "fetch_from": "dist_code.province"
}
```

`fetch_from` format: `<link_fieldname>.<source_field_on_linked_doc>`

---

## Architecture at a Glance

```
User selects a value in a Link field
       |
       v
Form.onValuesChange fires
       |
       v
handleFetchFrom(form, properties, changedField, changedValue)
       |
       ├─ Scans properties for fields where fetch_from starts with changedField
       ├─ Collects { fieldname, sourceField } pairs
       |
       v
POST frappe.client.validate_link
       | { doctype, docname, fields: [...sourceFields] }
       v
Returns field values from the linked document
       |
       v
form.setFieldsValue({ province: "Punjab", ... })
```

### Files involved

| File | Role |
|------|------|
| `src/utils/formUtils.js` | `handleFetchFrom` — core fetch logic; `shouldClearDeps` — clears fetched fields on unset |
| `src/components/form/DynamicForm.jsx` | Calls `handleFetchFrom` inside `onValuesChange` |
| `src/components/form/DynamicFormTabs.jsx` | Same — calls `handleFetchFrom` inside `onValuesChange` |

---

## 1. How `fetch_from` is Configured

Each field that should auto-populate defines a `fetch_from` string in its property config. The format is always:

```
<link_fieldname>.<field_on_linked_doctype>
```

Multiple fields can all fetch from the same Link field:

```json
[
  { "fieldname": "dist_code",  "fieldtype": "Link",   "options": "District" },
  { "fieldname": "province",   "fieldtype": "Data",   "fetch_from": "dist_code.province" },
  { "fieldname": "region",     "fieldtype": "Data",   "fetch_from": "dist_code.region" },
  { "fieldname": "dist_name",  "fieldtype": "Data",   "fetch_from": "dist_code.district_name" }
]
```

When `dist_code` changes, all three dependent fields are fetched in a single API call.

---

## 2. `handleFetchFrom` — Core Logic

`handleFetchFrom` is called inside `onValuesChange` on every field change. It does a single pass over `properties` to:

1. Find the doctype of the changed Link field (from `prop.options`)
2. Collect all fields that have `fetch_from` pointing to the changed field

```js
// src/utils/formUtils.js
export const handleFetchFrom = async (form, properties, changedField, changedValue) => {
  if (!form || !properties?.length || !changedField) return;

  let linkFieldDoctype = null;
  const fieldsToFetch = [];

  for (const prop of properties) {
    // Step 1: find the doctype of the link field that changed
    if (prop.fieldname === changedField && prop.options) {
      linkFieldDoctype = prop.options;
    }

    // Step 2: collect fields that fetch from this link field
    if (prop.fetch_from) {
      const [linkField, sourceField] = prop.fetch_from.split(".");
      if (linkField === changedField && sourceField) {
        fieldsToFetch.push({ fieldname: prop.fieldname, sourceField });
      }
    }
  }

  // Nothing to fetch — exit early
  if (fieldsToFetch.length === 0) return;

  // Link field was cleared — clear all dependent fields without an API call
  if (!changedValue) {
    const clearUpdates = fieldsToFetch.reduce((acc, { fieldname }) => {
      acc[fieldname] = null;
      return acc;
    }, {});
    form.setFieldsValue(clearUpdates);
    return;
  }

  if (!linkFieldDoctype) {
    console.warn(`No doctype found for link field: ${changedField}`);
    return;
  }

  // Fetch all source fields in one API call
  const fieldnames = fieldsToFetch.map((f) => f.sourceField);

  const formData = new FormData();
  formData.append("doctype", linkFieldDoctype);
  formData.append("docname", changedValue);
  formData.append("fields", JSON.stringify(fieldnames));

  const response = await fetch(
    `${window.location.origin}/api/method/frappe.client.validate_link`,
    { method: "POST", body: formData },
  );

  const data = await response.json();
  if (!data?.message) return;

  // Map API response back to form field names and apply
  const apiUpdates = fieldsToFetch.reduce((acc, { fieldname, sourceField }) => {
    const value = data.message[sourceField];
    if (value !== undefined && value !== null) {
      acc[fieldname] = value;
    }
    return acc;
  }, {});

  if (Object.keys(apiUpdates).length > 0) {
    form.setFieldsValue(apiUpdates);
  }
};
```

Key points:
- Only one API call is made regardless of how many fields fetch from the same link field.
- If the link field is cleared (`changedValue` is falsy), dependent fields are cleared locally — no API call needed.
- If the link field's doctype can't be found in `properties`, a warning is logged and the function exits.

---

## 3. Where It's Called

Both form components call `handleFetchFrom` inside `Form.onValuesChange`, which fires on every field change:

```jsx
// DynamicForm.jsx and DynamicFormTabs.jsx
<Form
  onValuesChange={async (changedValues) => {
    const changedField = Object.keys(changedValues)[0];
    if (!changedField) return;

    const changedValue = changedValues[changedField];

    // Auto-populate fields that fetch from this link field
    await handleFetchFrom(form, properties, changedField, changedValue);

    // Clear fields that depend on this field (display_depends_on, m_filters, etc.)
    shouldClearDeps(form, changedField, properties, data);
  }}
>
```

`handleFetchFrom` is `async` and awaited — the form waits for the API response before continuing.

---

## 4. Clearing Fetched Fields on Unset

`shouldClearDeps` handles the case where a link field is cleared after its dependent fields have already been populated. It checks if the source link field is now empty and clears the fetched field:

```js
// src/utils/formUtils.js — inside shouldClearDeps
if (prop.fetch_from) {
  const fetchFromField = prop.fetch_from.split(".")[0]; // e.g. "dist_code"

  // If the link field is now empty but the fetched field still has a value, clear it
  if (
    fetchFromField &&
    !form.getFieldValue(fetchFromField) &&
    form.getFieldValue(prop.fieldname)
  ) {
    form.setFieldValue(prop.fieldname, null);
  }
}
```

This runs synchronously after `handleFetchFrom` on every `onValuesChange` event.

---

## 5. The API — `frappe.client.validate_link`

`handleFetchFrom` uses `frappe.client.validate_link` to fetch field values from the linked document. Despite the name, this endpoint also returns field data when `fields` is provided.

Request (FormData):

```
doctype  = "District"
docname  = "DIS-001"
fields   = ["province", "region", "district_name"]
```

Response:

```json
{
  "message": {
    "province": "Punjab",
    "region": "Central",
    "district_name": "Lahore"
  }
}
```

The response is then mapped back to the form field names using the `fieldsToFetch` array built during the property scan.

---

## 6. Full Example — End to End

Given this field configuration:

```json
[
  {
    "fieldname": "dist_code",
    "fieldtype": "Link",
    "label": "District",
    "options": "District"
  },
  {
    "fieldname": "province_name",
    "fieldtype": "Data",
    "label": "Province",
    "fetch_from": "dist_code.province",
    "read_only": 1
  },
  {
    "fieldname": "region_name",
    "fieldtype": "Data",
    "label": "Region",
    "fetch_from": "dist_code.region",
    "read_only": 1
  }
]
```

Flow when user selects `"DIS-001"` in the District field:

```
onValuesChange({ dist_code: "DIS-001" })
  |
  handleFetchFrom(form, properties, "dist_code", "DIS-001")
    |
    linkFieldDoctype = "District"
    fieldsToFetch = [
      { fieldname: "province_name", sourceField: "province" },
      { fieldname: "region_name",   sourceField: "region" }
    ]
    |
    POST frappe.client.validate_link
      doctype=District, docname=DIS-001, fields=["province","region"]
    |
    response.message = { province: "Punjab", region: "Central" }
    |
    form.setFieldsValue({
      province_name: "Punjab",
      region_name: "Central"
    })
```

Flow when user clears the District field:

```
onValuesChange({ dist_code: null })
  |
  handleFetchFrom → changedValue is falsy
    form.setFieldsValue({ province_name: null, region_name: null })
    (no API call)
```

---

## 7. Adding `fetch_from` to a New Field

To make a field auto-populate from a linked document:

1. Ensure the Link field has `options` set to the target doctype (e.g., `"District"`).
2. On the dependent field, set `fetch_from` to `"<link_fieldname>.<field_on_linked_doc>"`.
3. Mark the dependent field as `read_only: 1` so users can't manually override the fetched value.
4. No code changes needed — `handleFetchFrom` picks it up automatically from the property config.

```json
{
  "fieldname": "tehsil_name",
  "fieldtype": "Data",
  "label": "Tehsil Name",
  "fetch_from": "tcode.tehsil_name",
  "read_only": 1
}
```

The only requirement is that the Link field (`tcode`) must appear in the same `properties` array so `handleFetchFrom` can resolve its doctype.
