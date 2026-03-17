# 🏷️ Add Tag From Location — Zoho Projects Deluge Script

## Overview

This Deluge script automatically assigns a **location-based tag** to both a **task** and its **parent project** in Zoho Projects. It works by reading a custom field (Location URL) from a task, fetching the corresponding record from a **Zoho Books Custom Module**, extracting the **BAIR Region** value, and then associating that region as a tag within Zoho Projects.

This is part of the **BairQuality Portal** automation suite built on Zoho Creator (Portal ID: `783442526`).

---

## 🔁 Script Flow Summary

```
Task (Zoho Projects)
    └─► Read UDF_CHAR1 (Location URL)
            └─► Extract Books Org ID + Location ID
                    └─► Fetch Location Record from Zoho Books
                            └─► Read cf_bair_region field
                                    └─► Find or Create Tag in Zoho Projects
                                            ├─► Associate Tag → Task
                                            └─► Associate Tag → Project
```

---

## 📋 Prerequisites

| Requirement | Details |
|---|---|
| Zoho Projects Connection | `zoho_projects_conn` — must have OAuth scope for Projects API v3 and Books API v3 |
| Portal ID | `783442526` (BairQuality portal) |
| Portal Name | `bairquality` |
| Zoho Books Custom Module | `cm_location` — must contain field `cf_bair_region` |
| Task Custom Field | `UDF_CHAR1` — stores the Zoho Books Location URL |

---

## 🔧 Input Parameters

| Parameter | Type | Description |
|---|---|---|
| `task_id` | String | The ID of the Zoho Projects task to tag |
| `project_id` | String | The ID of the project that contains the task |

These are passed into the function before the script begins. Both are required for all API calls.

---

## 📌 Step-by-Step Explanation

### Step 1 — Fetch Task Details

```
GET /restapi/portal/{portal_name}/projects/{project_id}/tasks/{task_id}/
```

The script calls the Zoho Projects REST API to retrieve full task details. It then iterates over the `custom_fields` array in the response to find the field with `column_name == "UDF_CHAR1"`.

This field stores a URL pointing to the related location record in Zoho Books Custom Module, for example:

```
https://books.zoho.com/app/60012345#cm_location/9876543210
```

> **Exit Condition:** If `UDF_CHAR1` is empty or does not contain `cm_location`, the script logs a message and stops execution — no tag operation is attempted.

---

### Step 2 — Extract IDs from the Location URL

The URL contains two pieces of critical information that are extracted using Deluge string functions:

| Value | Extraction Method | Example Result |
|---|---|---|
| `books_org_id` | Between `/app/` and `#` | `60012345` |
| `location_id` | After `cm_location/` | `9876543210` |

These IDs are used to build the Zoho Books API call in the next step.

---

### Step 3 — Fetch BAIR Region from Zoho Books

```
GET https://www.zohoapis.com/books/v3/cm_location/{location_id}?organization_id={books_org_id}
```

The script calls the Zoho Books API to retrieve the location's custom module record. It reads the field `cf_bair_region` from the `module_record_hash` object in the response.

The BAIR Region value is a text string like `"North"`, `"South"`, `"Central"`, etc. — this becomes the tag name.

> **Exit Condition:** If `cf_bair_region` is empty or not found in the response, the script logs and exits without proceeding.

---

### Step 4 — Get Existing Tags from Zoho Projects

```
GET https://projectsapi.zoho.com/api/v3/portal/{portal_id}/tags?range=200
```

The script fetches up to 200 existing tags from the portal. It loops through all tags and checks if a tag with the same name as `bair_region` already exists.

If a match is found, the `tag_id` is captured and the script skips tag creation entirely (Step 5).

---

### Step 5 — Create Tag If It Doesn't Exist

If no matching tag was found in Step 4, the script creates a new tag:

```
POST https://projectsapi.zoho.com/api/v3/portal/{portal_id}/tags
```

**Request body format:**

```json
{
  "tags": "[{\"name\":\"North\",\"color_class\":\"bg-tag1\"}]"
}
```

> **Note:** The `tags` parameter must be sent as a **JSON-encoded string array**, not a native map. This is a known requirement of the Zoho Projects v3 Tags API.

After the creation call:
- If the response contains a `tags` array, the `tag_id` is extracted from the first element.
- If the response does not contain the `tag_id` (occasional API behavior), the script **fetches all tags again** to locate the newly created tag by name.

---

### Step 6 — Associate Tag with the Task

```
POST https://projectsapi.zoho.com/api/v3/portal/{portal_id}/projects/{project_id}/tags/associate
     ?tag_id={tag_id}&entity_id={task_id}&entityType=5
```

The tag is linked to the specific task using the `associate` endpoint.

**entityType Values:**
| Value | Entity |
|---|---|
| `2` | Project |
| `5` | Task |

---

### Step 7 — Associate Tag with the Project

```
POST https://projectsapi.zoho.com/api/v3/portal/{portal_id}/projects/{project_id}/tags/associate
     ?tag_id={tag_id}&entity_id={project_id}&entityType=2
```

The same tag is also associated with the **parent project**, ensuring visibility at both the task level and the project level in the BairQuality portal dashboard.

> **Condition:** This step only runs if `tag_id` is non-empty and non-null (i.e., the tag was successfully found or created).

---

## 🌐 API Endpoints Used

| # | Method | Endpoint | Purpose |
|---|---|---|---|
| 1 | GET | `/restapi/portal/{name}/projects/{pid}/tasks/{tid}/` | Fetch task + custom fields |
| 2 | GET | `https://www.zohoapis.com/books/v3/cm_location/{id}` | Fetch location record |
| 3 | GET | `/api/v3/portal/{id}/tags?range=200` | List all existing tags |
| 4 | POST | `/api/v3/portal/{id}/tags` | Create new tag (if needed) |
| 5 | POST | `/api/v3/portal/{id}/projects/{pid}/tags/associate` | Tag → Task (entityType=5) |
| 6 | POST | `/api/v3/portal/{id}/projects/{pid}/tags/associate` | Tag → Project (entityType=2) |

---

## ⚠️ Error Handling & Exit Conditions

| Condition | Behavior |
|---|---|
| `UDF_CHAR1` is empty or missing `cm_location` | Logs info and exits — no tag applied |
| `cf_bair_region` is empty in Books response | Logs info and exits — no tag applied |
| Tag creation returns no `tag_id` | Falls back to re-fetching tag list by name |
| No `tag_id` available at step 6 | Logs error and skips both association calls |

---

## 🔑 Connection Requirements

The script uses a single Zoho connection named **`zoho_projects_conn`** for all API calls — both Zoho Projects and Zoho Books. This connection must be authorized with the following OAuth scopes:

```
ZohoProjects.tags.ALL
ZohoProjects.tasks.ALL
ZohoBooks.modules.READ
```

---

## 📁 Variables Reference

| Variable | Description |
|---|---|
| `portal_id` | Numeric portal ID (`783442526`) |
| `portal_name` | Portal name string (`bairquality`) |
| `location_url` | Zoho Books URL from UDF_CHAR1 |
| `books_org_id` | Organization ID extracted from location URL |
| `location_id` | Location record ID extracted from location URL |
| `bair_region` | The region name from `cf_bair_region` — becomes the tag name |
| `tag_id` | The Zoho Projects tag ID, either found or created |

---

## 📝 Notes

- **Portal ID vs Portal Name:** Steps 1 and 2 use `portal_name` (string) in the REST API URL, while Steps 4–7 use `portal_id` (numeric) in the v3 API URL. Both are required.
- **Tag Color:** All new tags are created with `color_class: "bg-tag1"` (default blue). This can be changed to `bg-tag2` through `bg-tag9` for different colors.
- **Tag Range:** The tag fetch uses `range=200`, which means up to 200 portal tags are checked before creating a new one. Increase this value if your portal has more than 200 tags.
- **Idempotency:** The script is safe to run multiple times on the same task. If the tag already exists and is already associated, Zoho Projects handles the duplicate gracefully.

---

## 👤 Author

**Rafiullah Nikzad**  
Senior Zoho Developer — CloudZ Technologies  
Zoho Afghanistan Community  
Portfolio: [rafiullahnikzad.netlify.app](https://rafiullahnikzad.netlify.app)
