This documentation outlines your .NET backend implementation for **WatermelonDB Sync (v0.27.1)**. Your implementation correctly handles enterprise-scale data (100k+ records) by utilizing **Entity Framework Core**, **SQL Server**, and the **WatermelonDB Sync Protocol**.

---

## 1. Data Model & Metadata

The `WatermelonProduct` class includes the specific metadata fields required by the sync protocol to track changes accurately.

| Property | Type | Purpose |
| --- | --- | --- |
| `Id` | `string` | **Global Unique ID**. WatermelonDB expects the backend to use the same ID as the client.

 |
| `LastModified` | `long` | **Server-side timestamp** (Unix ms). Used to identify changes since `lastPulledAt`.

 |
| `ServerCreatedAt` | `long` | Used specifically to differentiate between `created` and `updated` records during a pull.

 |
| `IsDeleted` | `bool` | **Soft delete** flag. Required to track and send deleted IDs to the client.

 |

>
> **Note:** Your implementation uses `snake_case_lower` for JSON serialization to match the WatermelonDB database convention.
>
>

---

## 2. Sync Pull Implementation (`GET /api/sync/pull`)

This endpoint retrieves server-side changes to be applied to the local database.

### Pull Logic

*
**Timestamping:** The server marks `serverTimestamp` at the start of the request to ensure consistency.


*
**Change Detection:** It fetches records where `LastModified > lastPulledAt`.


* **Record Categorization:**
*
**Created:** Records where `ServerCreatedAt > lastPulledAt`.


*
**Updated:** Records where `ServerCreatedAt <= lastPulledAt`.


*
**Deleted:** Records marked `IsDeleted == true`.




*
**Turbo Mode:** For initial syncs (`lastPulledAt == 0`), your code supports **Turbo Login** by returning raw JSON strings to bypass unnecessary JavaScript parsing on the client.



---

## 3. Sync Push Implementation (`POST /api/sync/push`)

This endpoint applies local client changes to the server database using a transactional batch process.

### Conflict Detection logic

Your implementation follows the **"Client Wins"** philosophy but enforces a strict **Conflict Check**:

1. The client sends its `lastPulledAt` timestamp.


2. The server checks if the existing record's `LastModified` is newer than the client's `lastPulledAt`.


3.
**Conflict:** If `existing.LastModified > request.LastPulledAt`, an `InvalidOperationException("CONFLICT")` is thrown.


4.
**Action:** The controller returns `409 Conflict`, forcing the client to pull remote changes before attempting another push.



### Transactional Integrity

*
**Transaction Scope:** All `Created`, `Updated`, and `Deleted` operations are wrapped in a `IDbContextTransaction`.


*
**Error Handling:** If any record fails (e.g., validation or conflict), the entire batch is rolled back to prevent partial, inconsistent state.


*
**Upsert Capability:** If a "Created" record already exists (common in network retries), the server gracefully updates it rather than failing.



---

## 4. Performance & Scalability

*
**Indexing:** The `AppDbContext` creates a database index on `LastModified` to ensure fast pull queries even with 100,000+ records.


*
**Bulk Processing:** The seeding logic and push processor use batching and `ToDictionaryAsync` to avoid **N+1 query** performance bottlenecks.


* **Containers:** The implementation uses `Testcontainers.MsSql` for a reproducible, enterprise-grade SQL Server environment.

### Recommended Next Steps

Would you like me to help you create the **React Native/TypeScript** side of this, specifically the `pullChanges` and `pushChanges` functions that will call these endpoints?

This documentation integrates your C# implementation with the official **WatermelonDB (v0.27.1)** synchronization protocol. It covers the data structures, service logic, and API endpoints required to sync enterprise-grade product data.

---

## 1. Data Transfer Objects (DTOs)

The following records define the JSON structure required by the **Watermelon Sync Protocol**.

```csharp
// Matches the "changes" object shape required by WatermelonDB
public record TableChanges(
    [property: JsonPropertyName("created")] List<object> Created,
    [property: JsonPropertyName("updated")] List<object> Updated,
    [property: JsonPropertyName("deleted")] List<string> Deleted
);

// The response sent to the client during a Pull Sync
public record SyncPullResponse(
    Dictionary<string, TableChanges>? Changes,
    long Timestamp,
    string? SyncJson = null // Used for Turbo Login optimization
);

// The request received from the client during a Push Sync
public record SyncPushRequest(
    [property: JsonPropertyName("changes")] Dictionary<string, TableChanges> Changes,
    [property: JsonPropertyName("last_pulled_at")] long LastPulledAt
);

```

---

## 2. Pull Sync Logic

The `GetPullChangesAsync` method implements the **Pull Endpoint** requirements: returning all changes since `lastPulledAt`.

```csharp
public async Task<SyncPullResponse> GetPullChangesAsync(long lastPulledAt, bool requestTurbo = false)
{
    // 1. Mark current server time BEFORE querying to ensure consistency
    long serverTimestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
    bool isFirstSync = lastPulledAt == 0;

    // 2. Fetch changes based on last_modified timestamp
    var allChanges = await context.Products
        .Where(p => isFirstSync || p.LastModified > lastPulledAt)
        .ToListAsync();

    var tableChanges = new TableChanges(
        // New records since last sync
        Created: allChanges
            .Where(p => !p.IsDeleted && (isFirstSync || p.ServerCreatedAt > lastPulledAt))
            .Cast<object>().ToList(),
        // Modified records since last sync
        Updated: allChanges
            .Where(p => !p.IsDeleted && !isFirstSync && p.ServerCreatedAt <= lastPulledAt)
            .Cast<object>().ToList(),
        // IDs of records deleted since last sync
        Deleted: allChanges
            .Where(p => p.IsDeleted)
            .Select(p => p.Id).ToList()
    );

    var responseData = new Dictionary<string, TableChanges> { { "products", tableChanges } };

    // 3. Support Turbo Login for massive initial datasets
    if (isFirstSync && requestTurbo)
    {
        var syncObj = new { changes = responseData, timestamp = serverTimestamp };
        string rawJson = JsonSerializer.Serialize(syncObj, SyncOptions);
        return new SyncPullResponse(null, serverTimestamp, rawJson);
    }

    return new SyncPullResponse(responseData, serverTimestamp);
}

```

---

## 3. Push Sync & Conflict Resolution

The `ProcessPushChangesAsync` method handles incoming local changes and enforces **Conflict Detection**.

```csharp
public async Task ProcessPushChangesAsync(SyncPushRequest request)
{
    // Push endpoints MUST be fully transactional
    using var tx = await context.Database.BeginTransactionAsync();
    try
    {
        if (request.Changes.TryGetValue("products", out var productChanges))
        {
            long now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();

            // Bulk fetch to avoid N+1 performance issues
            var incomingCreatedUpdated = productChanges.Created.Concat(productChanges.Updated).ToList();
            // ... (ID extraction logic)

            foreach (var item in incomingCreatedUpdated)
            {
                // ... (Parsing logic)
                if (existingRecords.TryGetValue(id, out var existing))
                {
                    // CONFLICT DETECTION: If server record changed after client's last pull [cite: 1796, 1797]
                    if (existing.LastModified > request.LastPulledAt)
                    {
                        throw new InvalidOperationException("CONFLICT");
                    }
                    UpdateProductFields(existing, raw, now);
                }
                else
                {
                    // Create if record doesn't exist (handles edge cases)
                    context.Products.Add(CreateNewProduct(id, raw, now));
                }
            }
            // ... (Deletion logic via soft-delete)
        }

        await context.SaveChangesAsync();
        await tx.CommitAsync();
    }
    catch (Exception ex)
    {
        await tx.RollbackAsync(); // Revert all changes on error [cite: 1800]
        throw;
    }
}

```

---

## 4. API Controller

The controller exposes the endpoints to the mobile/web frontend.

```csharp
[ApiController]
[Route("api/sync")]
public class WatermelonController(WatermelonService dbService) : ControllerBase
{
    [HttpGet("pull")]
    public async Task<ActionResult> Pull(
        [FromQuery(Name = "last_pulled_at")] long lastPulledAt,
        [FromQuery] bool turbo = false)
    {
        var response = await dbService.GetPullChangesAsync(lastPulledAt, turbo);

        // Return raw content for Turbo mode to avoid client-side parsing overhead
        if (response.SyncJson != null) return Content(response.SyncJson, "application/json");
        return Ok(response);
    }

    [HttpPost("push")]
    public async Task<IActionResult> Push([FromBody] SyncPushRequest request)
    {
        try
        {
            await dbService.ProcessPushChangesAsync(request);
            return Ok(new { ok = true });
        }
        catch (InvalidOperationException ex) when (ex.Message == "CONFLICT")
        {
            // Forces client to pull and resolve conflicts locally
            return Conflict(new { error = "Server has newer changes. Please pull first." });
        }
    }
}

```
