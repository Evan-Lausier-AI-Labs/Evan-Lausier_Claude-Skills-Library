# NetSuite Skills

Skills for interacting with NetSuite ERP via the NetSuite MCP integration. These skills automate record creation, inventory operations, and data lookups using SuiteQL queries and the NetSuite REST API.

## Skills in This Category

| Skill | Output | Trigger Phrases |
|-------|--------|-----------------|
| [Inventory Transfer](#inventory-transfer) | NetSuite Inventory Transfer records | "transfer", "move inventory", "move items", "move serials", serial numbers, item SKUs with source/destination locations, vendor bill transfers, transaction-based transfers |

---

## Inventory Transfer

Creates Inventory Transfer records in NetSuite to move serialized and non-serialized items between warehouse locations. Supports five request types: by serial number, by item + source location, by specific transaction, by date range, and by entity + date range.

**When to use:** Any request to relocate stock between NetSuite locations — even if the user doesn't say "inventory transfer" explicitly. Triggers on mentions of serial numbers, item SKUs with locations, transaction numbers (BILL-*, PO-*, IT-*), or uploaded spreadsheets containing serial/item data.

**When NOT to use:** General NetSuite queries, reporting, or non-inventory record operations.

**Request types:**

| # | Request Type | User Provides |
|---|-------------|---------------|
| 1 | By Serial Number | Serial number(s), Destination, Date |
| 2 | By Item + Source Location | Item SKU(s), Source, Qty, Destination, Date |
| 3 | Specific Transactions | Transaction number(s), Destination, Date |
| 4 | Transactions by Date | Date range, Destination, Transfer Date |
| 5 | Transactions by Entity+Date | Entity name, Date range, Destination, Transfer Date |

**Key behaviors:**
- Reads all reference files (`queries.md`, `transfer-creation.md`, `output-templates.md`, `processing-rules.md`) before starting any operation
- Asks for memo as the first step if user didn't provide one
- Presents confirmation summary and waits for user approval before creating any records
- Handles serialized items (100 serials/line max, 5 lines/transfer, 500 serials/transfer)
- Uses stream processing for large batches (>500 serials) to avoid context overflow
- Revalidates transaction-sourced items against current on-hand inventory
- Supports file-based input (`.xlsx`, `.csv`, `.tsv`) with auto-detection of known formats

**MCP tools used:** `ns_runCustomSuiteQL`, `ns_createRecord`, `ns_getRecord`, `ns_updateRecord`
