In a WatermelonDB **push sync**, the client sends all local changes (records created, updated, or deleted locally since the last sync) to the backend to be persisted in the server's database.

### 1. What the Backend Receives (The Request)

The backend receives a `POST` request containing a `changes` object and a `lastPulledAt` timestamp.

**The Request Body (JSON):**

*
**`changes`**: A dictionary where keys are table names and values conform to the standard Watermelon Sync Protocol:


*
**`created`**: An array of full raw records created locally.


*
**`updated`**: An array of full raw records modified locally.


*
**`deleted`**: An array of **IDs only** for records deleted locally.




*
**`lastPulledAt`**: The timestamp of the last successful **pull** performed by this client. This is used by the server to detect conflicts (e.g., if a record was modified on the server *after* this timestamp, a conflict exists).



**Example Request:**

```json
{
  "changes": {
    "tasks": {
      "created": [{ "id": "t1", "name": "Buy milk", "is_finished": false }],
      "updated": [{ "id": "t0", "is_finished": true }],
      "deleted": ["t_old"]
    }
  },
  "lastPulledAt": 1600000000000
}

```

---

### 2. Backend Processing Requirements

The backend must follow these rules to ensure data integrity and reliable synchronization:

*
**Atomicity/Transactions**: The push endpoint MUST be fully transactional. If applying any single change fails, the entire batch must be rolled back, and an error code must be returned.


*
**Conflict Detection**: If a record in the `changes` object was modified on the server **after** the client's `lastPulledAt`, the server MUST abort the push and return an error code. This forces the client to pull again and resolve the conflict locally.


* **Upsert Logic**:
* If a "new" record ID already exists on the server, the server should update it instead of returning an error (this handles cases where a previous push reached the server but the client never received the confirmation).


* If an "update" is sent for a record that doesn't exist, the server should generally create it to keep the sync from failing permanently.




*
**Internal Fields**: The server MUST ignore the `_status` and `_changed` fields if they are sent by the client.


*
**Data Sanitization**: The server should validate and sanitize record fields (e.g., whitelisting column names and ensuring correct data types) rather than just failing, to prevent users from getting stuck in an "unsyncable" state.


*
**Descendant Cleanup**: The server should ideally delete all descendants of a record when that record is deleted to avoid "orphans" in the database.



---

### 3. What the Backend Responds

The backend response for a push sync is typically simpler than a pull sync, as its primary purpose is to confirm success or signal a conflict.

*
**On Success**: A success status code (e.g., `200 OK` or `204 No Content`).


*
**On Failure/Conflict**: An error status code (e.g., `409 Conflict` or `400 Bad Request`).



### (Advanced) Partial Rejections

Introduced in v0.24, the backend can optionally respond with an `experimentalRejectedIds` parameter. This allows the server to accept most of the batch while rejecting specific records that failed validation, allowing the rest of the synchronization to proceed.

To implement conflict detection during a **push sync**, the backend must ensure that the client's local changes are based on the most recent version of the data available on the server.

### Conflict Detection Logic

When the backend receives a `pushChanges` request, it compares the client's `lastPulledAt` timestamp against the server's own `last_modified` timestamps for the records being updated or deleted.

* **The Check**: For every record in the `updated` or `deleted` arrays, the server must verify:


`server_record.last_modified <= request.lastPulledAt`.


*
**The Conflict**: If `server_record.last_modified > request.lastPulledAt`, it means another user or process updated that specific record after this client last pulled data.


*
**The Resolution**: The backend **must abort** the entire push transaction and return an error code (e.g., `409 Conflict`). This forces the client to perform a `pull` first, resolve the conflict locally (usually via a "client-wins" or custom resolver), and then attempt to push again.



---

### Implementation Recommendations

The documentation suggests a specific database pattern to make this logic reliable and performant:

*
**`last_modified` Column**: Add a `last_modified` field to every table on your server.


*
**Millisecond Resolution**: Use at least millisecond resolution for these timestamps to avoid overlapping changes.


*
**Monotonicity**: Ensure that timestamps are unique and always increasing (monotonic) to protect against server clock drifts or leap seconds.


*
**Server-Side Bump**: Always update the `last_modified` field to `NOW()` on the server every time a record is created or updated; never trust a timestamp sent from the client's local clock.


*
**Tracking Deletions**: To track deleted records, you can either use a "soft delete" (a `is_deleted` boolean) or a dedicated `deleted_records` table that stores the `record_id` and the `timestamp` of when it was removed.



### Summary of Request/Response Flow

| Step | Action | Requirements |
| --- | --- | --- |
| **1. Receive Push** | Backend receives `changes` and `lastPulledAt`.

 | Must be a `POST` request.

 |
| **2. Validate** | Check `last_modified` for every record in the push.

 | Whitelist column/table names for security.

 |
| **3. Apply** | If no conflicts, update server database.

 | Must be fully transactional.

 |
| **4. Respond** | Return success or a conflict error.

 | Error forces client to pull before re-pushing.

 |
