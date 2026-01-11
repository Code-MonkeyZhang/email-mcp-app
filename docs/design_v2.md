# MCP Email Server v2.0 Design Document (Draft)

## 🚀 Overview

The goal of this project is to upgrade the MCP Email Server from a **stateless instant proxy** to an **intelligent email hub with local indexing capabilities**. By introducing a local SQLite database and a full synchronization mechanism, we aim to achieve millisecond-level queries, complex SQL retrieval, contact management (including blocklists), and complete email status tracking.

Core Philosophy: **SQL-First for Reading, Tools for Writing**.

---

## 🛠️ 1. Tool Interface Design (MCP Tools)

Tools are categorized into three types: **Query (SQL)**, **Management (Tools)**, and **Action**.

### 1.1 Email Query (SQL Read-Only)

**Tool Name:** `query_emails`
**Description:** Execute read-only SQL queries on the local email index. Supports flexible searching, filtering, and counting.
**Parameters:**
- `account_name`: `str` (Required) - The account name.
- `sql_query`: `str` (Required) - A `SELECT` statement targeting the `emails` table.
  - **Example**: `SELECT * FROM emails WHERE folder='INBOX' AND is_read=0 ORDER BY date DESC LIMIT 10`

### 1.2 Contact Management (SQL Read-Write)

**Tool Name:** `manage_contacts_sql`
**Description:** Use SQL to Perform CRUD operations on contacts, including blocklist management.
**Parameters:**
- `account_name`: `str` (Required)
- `sql_query`: `str` (Required) - Supports `SELECT`, `INSERT`, `UPDATE`, `DELETE`.
  - **Example (Block)**: `UPDATE contacts SET is_blocked=1 WHERE email_address='spam@x.com'`
  - **Example (Query)**: `SELECT * FROM contacts WHERE name LIKE '%Bob%'`

### 1.3 Email Content & Actions

**Tool Name:** `get_emails_content`
**Description:** Fetch full email bodies (used when `body_preview` in SQL results is insufficient).
**Parameters:**
- `account_name`: `str` (Required)
- `email_uids`: `list[str]` (Required) - UIDs obtained from SQL results.

**Tool Name:** `send_email`
**Description:** Send a new email.
**Parameters:**
- `account_name`: `str` (Required)
- `recipients`: `list[str]` (Required)
- `subject`: `str` (Required)
- `body`: `str` (Required)
- `attachments`: `list[str]` (Optional)
- `cc`, `bcc`, `html`... (Standard optional parameters)

**Tool Name:** `update_email_status`
**Description:** Mark email status (read/unread) and sync back to the server.
**Parameters:**
- `account_name`: `str` (Required)
- `email_uids`: `list[str]` (Required)
- `mark_as`: `str` (Required) - `'read'` or `'unread'`

### 1.4 Account Configuration

**Tool Name:** `list_available_accounts` / `add_email_account`
**Description:** View and add email account configurations.

---

## 💾 2. Database Design (Schema)

Uses **SQLite** for storage, with the file path defined by `db_location` in the configuration.

### 2.1 Email Table (`emails`) - **Read-Only Exposure**

| Field Name | Type | Description |
|:---|:---|:---|
| `uid` | TEXT (PK) | IMAP Unique Identifier (UID) |
| `subject` | TEXT | Email Subject |
| `sender_email` | TEXT | Sender Email Address |
| `sender_name` | TEXT | Sender Name |
| `recipients` | TEXT | Recipient List (JSON String) |
| `date` | DATETIME | Sent Date (ISO8601) |
| `body_preview` | TEXT | First 200 characters of the body |
| `is_read` | BOOLEAN | Read Status (0: Unread, 1: Read) |
| `folder` | TEXT | **Folder** (INBOX, Sent, Drafts, Trash) |
| `account_name` | TEXT | Account Name (Internal field for multi-account isolation) |

### 2.2 Contact Table (`contacts`) - **Read-Write Exposure**

| Field Name | Type | Description |
|:---|:---|:---|
| `email_address` | TEXT (PK) | Contact Email Address |
| `name` | TEXT | Contact Name |
| `is_blocked` | BOOLEAN | **Block Status** (0: Normal, 1: Blocked) |
| `notes` | TEXT | Notes/Remarks |

---

## ✨ 3. New Features & Strategies

### 3.1 Synchronization Strategy (Background Automatic Run)
- **Full Folder Sync**: Automatically syncs the four standard folders: `INBOX`, `Sent`, `Drafts`, `Trash`.
- **Incremental Sync**: Implements incremental fetching based on IMAP `UID` and `UIDVALIDITY` to avoid duplicate downloads.
- **Body Strategy**: Downloads only metadata and a 200-character preview (stored in `body_preview`) by default. Full bodies are downloaded on-demand (via `get_emails_content`).

### 3.2 Blocklist Mechanism
- **Implementation**: Marked via the `is_blocked` field in the `contacts` table.
- **Query Filtering**: The `query_emails` tool's default behavior remains unchanged (shows all). However, at the system level (Prompt level), the LLM will be instructed to check the `is_blocked` status, or the LLM can write its own filtering logic: `WHERE sender_email NOT IN (SELECT email_address FROM contacts WHERE is_blocked=1)`.

### 3.3 Interaction Paradigm
- **Query via SQL**: Grants the LLM extreme flexibility, supporting complex combined queries (e.g., filtering by time, folder, status, sender combination).
- **Manage via SQL**: Contact CRUD operations use a unified SQL interface for consistency.
- **Action via Tool**: Operations involving server interaction, such as sending emails or marking status, retain dedicated tools.

