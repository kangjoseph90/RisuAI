# RFC: Performance Overhaul & Database Architecture Transition

## 1. Problem Analysis

The current application uses a "Singleton DB" (Document Store) architecture where the entire application state (chat history, characters, settings) is stored in a single JSON-like object (`RisuSave` format).

### Current Bottlenecks
As identified, this leads to three major performance issues that scale linearly $O(N)$ with the size of the chat history:

1.  **Snapshot Cost:**
    *   **Cause:** Every save operation (even for a single new message) requires serializing the entire `db` object.
    *   **Impact:** As the chat grows, `JSON.stringify` (or `msgpackr.encode`) takes increasingly longer, blocking the main thread (or worker).

2.  **Encoding Cost:**
    *   **Cause:** The monolithic file is often compressed (`fflate` / `gzip`) and encoded.
    *   **Impact:** CPU spikes during auto-saves.

3.  **Network Traffic:**
    *   **Cause:**
        *   **Full Save:** Sends the entire file size over the network.
        *   **Patch Sync:** While `/api/patch` sends small diffs, the *server* still needs to load the full file, apply the patch, hash the full result, and write the full file back to disk. This puts high load on the server IO and CPU.

## 2. Is Snapshotting Inevitable?

**No.** Snapshotting the entire state is a consequence of the current data model, which treats the entire database as a single "Document".

By changing the data model to a **Relational** or **Log-Structured** model, we can eliminate the need to snapshot the entire history for every small change.

## 3. Proposed "Ultimate" Solution

The ultimate solution is to transition from a Monolithic Document Store to a **Transactional Database** (specifically **SQLite**).

### Architecture Changes

#### A. Backend (Server/Local Storage)
Replace `RisuSave` (file-based) with **SQLite**.

*   **Why SQLite?**
    *   It is serverless and works locally (via `sqlite-wasm` or native bindings) and on the backend.
    *   It supports updates to single rows without rewriting the entire database file.
    *   It handles concurrency better than rewriting files.

*   **Schema Design (Example):**
    ```sql
    CREATE TABLE characters (
        id TEXT PRIMARY KEY,
        name TEXT,
        data BLOB -- JSON payload for character settings
    );

    CREATE TABLE chats (
        id TEXT PRIMARY KEY,
        character_id TEXT,
        created_at INTEGER
    );

    CREATE TABLE messages (
        id TEXT PRIMARY KEY,
        chat_id TEXT,
        timestamp INTEGER,
        role TEXT,
        content TEXT,
        FOREIGN KEY(chat_id) REFERENCES chats(id)
    );
    ```

#### B. Network (Sync Protocol)
Move to an **Incremental / Delta Sync** protocol.

*   **Current:** `Client State` <-> `Server State` (Full Match)
*   **Proposed:** `POST /api/messages` (Append only)
    *   **Sending a message:** Client sends `{ chat_id, content, ... }`. Server inserts it into `messages` table.
    *   **Fetching updates:** Client requests `GET /api/sync?since=<timestamp>`. Server returns only rows modified after that time.

#### C. Frontend (Client)
*   **Lazy Loading:** Do not load all 10,000 messages into the DOM or memory at start.
*   **Pagination:** Load the last 50 messages. When the user scrolls up, fetch/query the previous 50 from the local SQLite database.

## 4. Intermediate Solution (Mitigation)

If a full migration to SQLite is too resource-intensive immediately, a **Chunked Storage** approach can be used.

1.  **Split `database.bin`:**
    *   Keep `master.bin` for settings and character lists.
    *   Create separate files for each chat: `chat_<uuid>.bin`.
2.  **Lazy Load Chats:**
    *   Only load the `chat_<uuid>.bin` of the currently active character.
    *   When switching characters, unload the previous one and load the new one.
3.  **Impact:**
    *   Snapshots only affect the active chat file (much smaller).
    *   Network sync only transfers the active chat.

## 5. Summary of Benefits

| Feature | Current (Singleton) | Proposed (SQLite/Delta) |
| :--- | :--- | :--- |
| **Write Cost** | $O(Total Size)$ | $O(1)$ (Size of new message) |
| **Read Cost** | $O(Total Size)$ | $O(Limit)$ (Pagination) |
| **Network** | High (Full File/Hash) | Low (Only Deltas) |
| **Crash Risk** | High (Corrupts full DB) | Low (ACID Transactions) |

## 6. Recommendation

For a robust, scalable chat application ("RisuAI"), adopting **SQLite** is the industry standard solution to the performance problems described. It solves the snapshot, encoding, and network issues simultaneously.
