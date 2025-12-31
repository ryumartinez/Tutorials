This documentation provides a comprehensive guide to **WatermelonDB (v0.27.1)**, a reactive database framework designed for building powerful React and React Native applications that scale to tens of thousands of records while remaining fast.

---

## Core Philosophy: Why WatermelonDB?

Standard tools like Redux or MobX with persistence adapters often struggle with scale because loading a full database into JavaScript is expensive, leading to slow app launches. WatermelonDB solves this through two primary mechanisms:

*
**Lazy Loading:** Data is only loaded when explicitly requested.


*
**Native Threading:** All querying is performed directly on a rock-solid **SQLite** database on a separate native thread.


*
**Observability:** Unlike using SQLite directly, Watermelon is fully observable; when a record changes, all dependent UI components automatically re-render.



## Key Architectural Concepts

To use WatermelonDB, developers must define three main layers:

### 1. Database Schema

The schema defines the tables and columns of the underlying database.

*
**Tables:** Conventionally named in plural and snake_case (e.g., `posts`).


*
**Column Types:** Supports `string`, `number`, and `boolean`.


*
**Indexing:** You can enable indexing on specific columns (like `_id` fields) to speed up queries, though it has a cost for creation and update speed.



### 2. Models

Models are JavaScript classes that represent types of things in your app (e.g., `Post`, `Comment`).

*
**Decorators:** Fields are defined using ES6 decorators like `@field`, `@text`, and `@date`.


*
**Associations:** Relations are defined as `has_many` or `belongs_to`.


*
**Derived Fields:** Getters can be used to calculate values based on database fields.



### 3. Writers and Readers

All modifications to the database must occur within a **Writer**.

*
**Consistency:** Because WatermelonDB is highly asynchronous, writers ensure that no two processes modify the data simultaneously.


*
**Batching:** For performance, multiple changes should be batched together to avoid excessive back-and-forth with the database.



---

## Powerful Features

| Feature | Description |
| --- | --- |
| **Reactivity** | Connect components via `withObservables` so they update automatically when data changes.

 |
| **Synchronization** | Built-in sync engine that coordinates "pulling" and "pushing" changes with a remote backend.

 |
| **Migrations** | A mechanism to add new tables and columns in a backward-compatible way without losing user data.

 |
| **Multiplatform** | Supports iOS, Android, Windows (experimental), web, and Node.js.

 |

---

## Advanced Usage & Performance

*
**Turbo Login:** An optimization introduced in v0.23 that makes initial syncs up to 5.3x faster by bypassing some JavaScript processing.


*
**Unsafe SQL:** While discouraged for beginners, an "escape hatch" exists to execute raw SQL queries for complex performance optimizations.


*
**Replacement Sync:** Added in v0.25, this allows the server to send a full dataset to replace the local database while preserving unpushed local changes.



> **Security Note:** Never trust user input directly in queries. Always sanitize user input using `Q.sanitizeLikeString` to prevent SQL injection.
>
>
