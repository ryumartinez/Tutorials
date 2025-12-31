This documentation outlines the frontend architecture and synchronization logic for your **WatermelonDB (v0.27.1)** implementation. The setup is optimized for high performance using JSI and the Watermelon Sync protocol to handle enterprise-scale product data.

---

## 1. Database Configuration (`index.ts`)

The database is the root object that manages the connection between the reactive UI and the underlying SQLite storage.

*
**SQLite Adapter**: Configured with `jsi: true` to enable the high-performance JavaScript Interface, which is significantly faster than the legacy bridge.


*
**Database Instance**: The instance is exported directly to ensure it is always available for imports across the app, preventing "undefined" errors during initialization.



---

## 2. Data Modeling

### Schema Definition (`schema.ts`)

The schema acts as the "source of truth" for the underlying database tables.

* **Version Management**: Currently at version `1`. Any future changes to table structures must increment this version.


*
**Indexing**: Critical fields like `item_id`, `bar_code`, and `data_area_id` are marked with `isIndexed: true` to ensure fast query resolution.



### Product Model (`product.ts`)

Models map database rows to JavaScript objects using decorators.

*
**Field Decorators**: Uses `@text` for string fields (which automatically trims whitespace) and `@field` for boolean values.


*
**Naming Conventions**: JavaScript properties use `camelCase`, while the `@text` and `@field` decorators map them to the `snake_case` columns defined in the schema.



---

## 3. Synchronization Logic (`sync.ts`)

The `pullAndPush` function coordinates data exchange with your .NET backend using the `synchronize()` helper.

### Pull Mechanism

*
**Initial Sync (Turbo Mode)**: If `lastPulledAt` is null/0, the app requests a "Turbo Login". It returns `syncJson` (raw text) instead of a parsed object to bypass expensive JavaScript processing, making the first sync up to 5.3x faster.


*
**Incremental Sync**: For subsequent pulls, it returns a standard `changes` object containing only the new updates since the last timestamp.


*
**Data Integrity Check**: In development mode (`__DEV__`), the logic includes a manual check to ensure all incoming products have valid IDs before processing.



### Push Mechanism

*
**Local Changes**: WatermelonDB automatically tracks locally created, updated, or deleted records.


* **Conflict Handling**: The `last_pulled_at` timestamp is sent to the server. If the server returns a `409 Conflict`, the sync fails, forcing the client to pull remote changes first to ensure data consistency.



### Diagnostics & Logging

*
**SyncLogger**: Utilizes the built-in `SyncLogger` to capture structured diagnostic data, such as start/finish times and error details.


*
**Censorship**: Remember to act responsibly with these logs as they may contain private user information; do not send them to external services without proper sanitization.



---

## 4. Migration Strategy (`migrations.ts`)

Migrations allow you to update the database schema without clearing user data.

*
**Backward Compatibility**: The `migrationsEnabledAtVersion` is set to `1` in the sync config to ensure that even if the schema changes, the sync process can request missing columns from the backend.


*
**Workflow**: When adding new fields, you must define the change in `migrations.ts` first, update `schema.ts`, and finally bump the schema version.



This documentation provides a technical overview of the frontend implementation for **WatermelonDB (v0.27.1)**, integrating with a **.NET backend**. It covers schema definition, model creation, and the synchronization logic required for enterprise-grade product data.

---

## 1. Database Initialization (`index.ts`)

The database instance is the central point of the WatermelonDB architecture, managing the connection between models and the storage adapter.

```typescript
import { Database } from '@nozbe/watermelondb';
import SQLiteAdapter from '@nozbe/watermelondb/adapters/sqlite';
import Product from "@/database/product";
import {mySchema} from "@/database/schema";
import migrations from "@/database/migrations";

// 1. Create the adapter
const adapter = new SQLiteAdapter({
  schema: mySchema,
  migrations,
  jsi: true,            // High performance mode
  dbName: 'watermelon', // This creates 'watermelon.db'
});

// 2. Export the database instance directly
// This ensures that 'import { database }' is never undefined
export const database = new Database({
  adapter,
  modelClasses: [Product],
});

```

---

## 2. Schema and Migrations

The schema defines the structure of your local tables, while migrations handle updates in a backward-compatible way.

### Schema Definition (`schema.ts`)

Indexes are applied to columns frequently used in lookups to ensure performance at scale.

```typescript
import { appSchema, tableSchema } from '@nozbe/watermelondb';

export const mySchema = appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'products',
      columns: [
        { name: 'name', type: 'string' },
        { name: 'item_id', type: 'string', isIndexed: true },
        { name: 'bar_code', type: 'string', isIndexed: true },
        { name: 'brand_code', type: 'string' },
        { name: 'brand_name', type: 'string' },
        { name: 'color_code', type: 'string' },
        { name: 'color_name', type: 'string' },
        { name: 'size_code', type: 'string' },
        { name: 'size_name', type: 'string' },
        { name: 'unit', type: 'string' },
        { name: 'data_area_id', type: 'string', isIndexed: true },
        { name: 'invent_dim_id', type: 'string' },
        { name: 'is_required_batch_id', type: 'boolean' },
      ],
    }),
  ],
});

```

### Migrations Workflow (`migrations.ts`)

Always use migrations once an app has been shipped to prevent user data loss.

```typescript
import { schemaMigrations } from '@nozbe/watermelondb/Schema/migrations'

export default schemaMigrations({
  migrations: [
    // Leave this empty for now if you have no changes yet
  ],
})

```

---

## 3. Product Model (`product.ts`)

Models use decorators to map JavaScript properties to database columns. JavaScript uses `camelCase`, while the database uses `snake_case`.

```typescript
import { Model } from '@nozbe/watermelondb';
import { field, text } from '@nozbe/watermelondb/decorators';

export default class Product extends Model {
  static table = 'products';

  @text('name') name!: string;
  @text('item_id') itemId!: string;
  @text('bar_code') barCode!: string;
  @text('brand_code') brandCode!: string;
  @text('brand_name') brandName!: string;
  @text('color_code') colorCode!: string;
  @text('color_name') colorName!: string;
  @text('size_code') sizeCode!: string;
  @text('size_name') sizeName!: string;
  @text('unit') unit!: string;
  @text('data_area_id') dataAreaId!: string;
  @text('invent_dim_id') inventDimId!: string;
  @field('is_required_batch_id') isRequiredBatchId!: boolean;
}

```

---

## 4. Sync Logic (`sync.ts`)

The synchronization process utilizes **Turbo Login** for the initial sync to maximize speed and uses a **SyncLogger** for diagnostics.

```typescript
import { synchronize, SyncPullArgs, SyncPushArgs } from '@nozbe/watermelondb/sync';
import SyncLogger from '@nozbe/watermelondb/sync/SyncLogger';
import { database } from './index';

export async function pullAndPush(): Promise<void> {
  const logger = new SyncLogger(10); // Keep last 10 logs

  try {
    await synchronize({
      database,
      log: logger.newLog(),
      pullChanges: async ({ lastPulledAt, schemaVersion, migration }: SyncPullArgs) => {
        const isFirstSync = lastPulledAt === null || lastPulledAt === 0;
        const url = `http://10.0.2.2:5117/api/sync/pull?last_pulled_at=${lastPulledAt ?? 0}&turbo=${isFirstSync}`;

        const response = await fetch(url);
        if (!response.ok) throw new Error(await response.text());

        if (isFirstSync) {
          // Use Turbo Login: return raw JSON text directly
          const json = await response.text();
          return { syncJson: json };
        } else {
          const { changes, timestamp } = await response.json();
          return { changes, timestamp };
        }
      },
      pushChanges: async ({ changes, lastPulledAt }: SyncPushArgs) => {
        const response = await fetch(`http://10.0.2.2:5117/api/sync/push`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ changes, last_pulled_at: lastPulledAt }),
        });

        if (!response.ok) throw new Error(await response.text());
      },
      migrationsEnabledAtVersion: 1, // Placeholder for existing apps
      unsafeTurbo: true,             // Enable Turbo Mode
      sendCreatedAsUpdated: true,    // Suppress warnings for specific backends
    });

    console.log('Sync successful. Diagnostics:', logger.formattedLogs);
  } catch (error) {
    console.error('Sync failed:', error);
    console.log('Failure Diagnostics:', logger.formattedLogs);
  }
}

```
