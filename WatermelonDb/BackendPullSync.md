For a WatermelonDB **pull sync**, the backend is responsible for providing a consistent view of all changes that have occurred since the client's last synchronization.

### The Pull Response Structure

The backend must return a JSON object with two top-level keys: `changes` and `timestamp`.

*
**`changes`**: A map where each key is a table name. Each table contains three arrays:


*
**`created`**: Full raw records for items added to the server since the last pull.


*
**`updated`**: Full raw records for items modified on the server since the last pull.


*
**`deleted`**: An array of **IDs only** for records removed from the server since the last pull.




*
**`timestamp`**: An integer representing the current server time. This value will be stored by the client and sent back as `lastPulledAt` in the next sync request.



---

### Example Response JSON

```json
{
  "changes": {
    "posts": {
      "created": [
        { "id": "p1", "title": "New Post", "is_published": true }
      ],
      "updated": [
        { "id": "p2", "title": "Updated Title", "is_published": false }
      ],
      "deleted": ["p3"]
    },
    "comments": {
      "created": [],
      "updated": [],
      "deleted": []
    }
  },
  "timestamp": 1704028800000
}

```



---

### Critical Implementation Rules

To ensure data integrity, the backend must follow these strict requirements:

*
**Format Consistency**: Record field names must match the **database schema** (e.g., `snake_case` like `is_published`), not the JavaScript model names (e.g., `isPublished`).


*
**Whitelisting**: You must whitelist acceptable collection and column names to prevent security vulnerabilities (e.g., avoiding `__proto__`).


*
**Exclude Internal Fields**: Never include WatermelonDB's internal columns like `_status` or `_changed` in the response.


*
**No Duplicates**: A single record ID must not appear more than once within the response for a specific table.


*
**First Sync**: If the client sends a `lastPulledAt` of `null` or `0`, the backend must return **all accessible records** as `created`.


*
**Consistency (Atomic View)**: All queries for different tables should ideally be performed in a single transaction or write lock. If this isn't possible, you must mark the `timestamp` **before** you start querying to ensure no changes are missed in the next cycle.



### Handling Migrations

If the client sends a `migration` object, it means their local schema has changed. In this case, the backend must:

* Include all records from any **new tables** added since the user's last sync.


* Include all records where **newly added columns** have a non-default value.



When performing a **pull sync**, the backend receives several parameters from the client to identify the state of the local database and determine what new information needs to be sent down.

### Required Parameters

The backend typically receives these arguments through URL parameters in a `GET` request:

*
**`lastPulledAt`**: A timestamp (integer) representing the last time the client successfully pulled changes from the server. If this is the client's first synchronization, this value will be `null` or `0`.


*
**`schemaVersion`**: An integer indicating the current version of the local database schema.


*
**`migration`**: An object (or `null`) that describes schema changes since the last sync. This is used to maintain consistency if the app was updated and new tables or columns were added.



---

### Understanding the Migration Object

If the `migration` parameter is not null, it contains details the backend needs to handle "Migration Syncs":

| Property | Description |
| --- | --- |
| **`from`** | The schema version the client had during their last successful sync.

 |
| **`tables`** | An array of table names that were added to the local database since the last sync.

 |
| **`columns`** | An object specifying which columns were added to which existing tables since the last sync.

 |

---

### Backend Logic Responsibilities

Upon receiving these parameters, the backend must perform the following logic to prepare the response:

*
**Initial Sync Handling**: If `lastPulledAt` is `null` or `0`, the backend must return **all** records the user has access to as "created."


*
**Incremental Fetching**: If `lastPulledAt` is provided, the backend must find all records in all collections that were created, updated, or deleted on the server **after** that timestamp.


*
**Migration Support**: If the `migration` object indicates new tables were added, the backend must include all records for those tables. If new columns were added to existing tables, the backend must include records where those specific columns have non-default values.


*
**Security & Validation**: The backend should whitelist the `tables` and `columns` requested in the migration object to ensure the client isn't requesting internal or sensitive fields.



Would you like to see the **Push Sync** requirements, which explain what the backend receives when the client sends local changes up to the server?
