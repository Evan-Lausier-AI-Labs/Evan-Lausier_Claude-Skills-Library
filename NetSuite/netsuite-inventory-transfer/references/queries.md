# SuiteQL Queries Reference

Queries are organized into two groups:
- **Inventory Queries (INV-):** Used for Options 1 and 2. Query the master inventory/on-hand tables directly — do not require a transaction reference.
- **Transaction Queries (T-):** Used for Options 3, 4, and 5. Look up items via transaction records.

---

## Inventory Queries (Options 1 & 2)

### INV-A1: Find Serials by Serial Number

**When to use:** Option 1 — user provides specific serial number(s)

**Purpose:** Look up current location, item, and on-hand status for each serial number directly from the inventory tables.

```sql
SELECT 
    inv.id          AS inventory_number_id,
    inv.inventorynumber AS serial_number,
    inv.item        AS item_id,
    i.itemid        AS item_name,
    nib.location    AS source_location_id,
    l.name          AS source_location_name,
    nib.quantityonhand,
    i.subsidiary
FROM inventorynumber inv
INNER JOIN inventorynumberlocation nib ON nib.inventorynumber = inv.id
INNER JOIN item i ON i.id = inv.item
INNER JOIN location l ON l.id = nib.location
WHERE inv.inventorynumber IN ('{serial_1}', '{serial_2}')
AND nib.quantityonhand > 0
ORDER BY nib.location, inv.item, inv.inventorynumber
```

**Key Notes:**
- `inventorynumber.inventorynumber` = human-readable serial string (e.g. "SN-0001-AAAA") — used in transfer payload `serialNumbers` field
- `inventorynumber.id` = internal numeric ID (not used directly in transfer payloads)
- Filter `quantityonhand > 0` ensures only transferable serials are returned
- Group results by `source_location_id` after fetching — separate transfers per source location

**⚠ Batch Integrity Rule (>500 serials):** When batching the IN clause for large serial lists:
- **Build each batch directly from the stored serial list using index ranges** (e.g., `serials[0:500]`, `serials[500:1000]`, etc.). Do NOT reconstruct serial strings from memory or re-derive them.
- **Verify batch boundaries:** Each serial must appear in exactly one batch. No gaps, no overlaps.
- **If any batch reports serials as "not found", re-verify the IN clause used the correct serial strings from the source list before reporting to the user.** Batch construction errors (wrong strings in the query) are a known failure mode that causes false "not found" results.

**After running:** Group serials by source location and item. Present confirmation summary. Create one transfer per source location.

---

### INV-B0: Item Lookup (Always Run First for Option 2)

**When to use:** Option 2 — always run this first to determine the item's serialization status before choosing the on-hand lookup strategy.

**Purpose:** Get item metadata to decide whether to use INV-B1 (non-serialized) or INV-B1s (serialized).

```sql
SELECT 
    i.id            AS item_id,
    i.itemid        AS item_name,
    i.displayname,
    i.isserialitem,
    i.itemtype,
    i.isinactive,
    i.subsidiary
FROM item i
WHERE UPPER(i.itemid) = UPPER('{item_sku}')
AND i.isinactive = 'F'
```

**After running:**
- **0 results** → "Item '{sku}' not found (or is inactive)"
- **1 result, `isserialitem = 'T'`** → Use **INV-B1s** for on-hand count (any item type)
- **1 result, `isserialitem = 'F'`, `itemtype = 'InvtPart'`** → Use **INV-B1** for on-hand count
- **1 result, `isserialitem = 'F'`, `itemtype = 'Assembly'`** → Use **INV-B1a** (Assembly fallback — `inventoryitemlocations` does not contain Assembly items)
- **Multiple results** → Apply item resolution rules (exact match, active only, etc.)

**Key point:** This query does NOT filter on `itemtype`. Both `InvtPart` and `Assembly` items are valid for inventory transfers.

---

### INV-B1: Find Non-Serial Item On-Hand by Location

**When to use:** Option 2 — item is **not** serialized (determined by INV-B0)

**Purpose:** Verify on-hand quantity at the source location before creating a transfer.

```sql
SELECT 
    i.id            AS item_id,
    i.itemid        AS item_name,
    i.isserialitem,
    il.location     AS source_location_id,
    l.name          AS source_location_name,
    il.quantityonhand,
    il.quantityavailable,
    i.subsidiary
FROM item i
INNER JOIN inventoryitemlocations il ON il.item = i.id
INNER JOIN location l ON l.id = il.location
WHERE UPPER(i.itemid) IN (UPPER('{item_sku_1}'), UPPER('{item_sku_2}'))
AND il.location = {source_location_id}
AND i.isinactive = 'F'
AND il.quantityonhand > 0
ORDER BY i.itemid
```

**After running:**
- Verify `quantityavailable >= requested_qty` before proceeding
- If user requested "all" → use `quantityonhand` as transfer qty

> **Note:** The `inventoryitemlocations` table only contains data for `InvtPart` items. Assembly items do not appear in this table. For serialized items (including Assembly), use INV-B1s instead. For **non-serialized Assembly items**, use INV-B1a below.

---

### INV-B1a: Verify Non-Serialized Assembly Item On-Hand (Fallback)

**When to use:** Option 2 or revalidation (Options 3–5) — item is **not serialized** AND `itemtype = 'Assembly'` (determined by INV-B0 or T3). This is required because `inventoryitemlocations` does not contain Assembly items.

**Purpose:** Verify that the non-serialized Assembly item has sufficient total on-hand inventory before creating a transfer. Per-location on-hand is not available via SuiteQL for Assembly items.

**Method:** Use `ns_getRecord` to read the item record:
```
Tool: ns_getRecord
Parameters:
  recordType: "assemblyitem"
  id: {item_id}
```

**Read the `totalQuantityOnHand` field from the response.**

**After running:**
- If `totalQuantityOnHand >= requested_qty` → proceed with the requested/transaction quantity
- If `totalQuantityOnHand < requested_qty` → "Only {totalQuantityOnHand} units of {item_name} available (total across all locations); {requested} requested. Proceed with {totalQuantityOnHand}?"
- If `totalQuantityOnHand = 0` → "Item '{item_name}' has zero total on-hand. Skipping."

> **Known limitation:** This checks total on-hand across ALL locations, not per-location. It is a safety guard — it confirms the inventory exists somewhere — but cannot guarantee it is at the specific source location. Per-location verification for non-serialized Assembly items is not available via SuiteQL. If the transfer fails at creation time due to insufficient on-hand at the source location, NetSuite will reject it with an error.

---

### INV-B1s: Find Serialized Item On-Hand by Location

**When to use:** Option 2 — item **is** serialized (any item type: InvtPart or Assembly, determined by INV-B0)

**Purpose:** Count on-hand serials at the source location using `inventorynumberlocation`. This is the only reliable source for serialized Assembly items, which do not appear in `inventoryitemlocations`.

```sql
SELECT 
    i.id            AS item_id,
    i.itemid        AS item_name,
    i.isserialitem,
    i.itemtype,
    nib.location    AS source_location_id,
    l.name          AS source_location_name,
    COUNT(inv.id)   AS quantityonhand,
    i.subsidiary
FROM item i
INNER JOIN inventorynumber inv ON inv.item = i.id
INNER JOIN inventorynumberlocation nib ON nib.inventorynumber = inv.id
INNER JOIN location l ON l.id = nib.location
WHERE UPPER(i.itemid) = UPPER('{item_sku}')
AND nib.location = {source_location_id}
AND i.isinactive = 'F'
AND nib.quantityonhand > 0
GROUP BY i.id, i.itemid, i.isserialitem, i.itemtype, nib.location, l.name, i.subsidiary
ORDER BY i.itemid
```

**After running:**
- **0 results** → "Item '{sku}' not found at {location} or has zero serialized quantity on hand"
- Validate `quantityonhand >= requested_qty`
- If user requested "all" → use returned `quantityonhand` as transfer qty
- If `quantityonhand < requested_qty` → "Only {available} serials available at {location}; {requested} requested. Proceed with available?"
- Then proceed to **INV-B2** (≤500 serials) or **INV-B3** (>500 serials) to fetch serial IDs

---

### INV-B2: Fetch Serial IDs for Item at Location (All at Once)

**When to use:** Options 2, 3, 4, 5 — item is serialized, total serials at location **≤500**. For Options 3–5, cap the result count at MIN(T3 transaction count, INV-B1s revalidated count).

```sql
SELECT 
    inv.id          AS inventory_number_id,
    inv.inventorynumber AS serial_number,
    inv.item        AS item_id,
    nib.location    AS source_location_id
FROM inventorynumber inv
INNER JOIN inventorynumberlocation nib ON nib.inventorynumber = inv.id
WHERE inv.item = {item_id}
AND nib.location = {source_location_id}
AND nib.quantityonhand > 0
ORDER BY inv.id
```

---

### INV-B3: Fetch Serial IDs for Item at Location (Cursor Pagination)

**When to use:** Options 2, 3, 4, 5 — item is serialized, total serials at location **>500**. For Options 3–5, cap the result count at MIN(T3 transaction count, INV-B1s revalidated count).

⚠️ **CRITICAL:** Do NOT use OFFSET with SELECT DISTINCT — it fails silently. Use cursor-based pagination. Do NOT use `pageSize` parameter — it triggers server-side OFFSET pagination which conflicts with DISTINCT.

```sql
SELECT 
    inv.id          AS inventory_number_id,
    inv.inventorynumber AS serial_number,
    inv.item        AS item_id,
    nib.location    AS source_location_id
FROM inventorynumber inv
INNER JOIN inventorynumberlocation nib ON nib.inventorynumber = inv.id
WHERE inv.item = {item_id}
AND nib.location = {source_location_id}
AND nib.quantityonhand > 0
AND inv.id > {last_inventory_number_id}
ORDER BY inv.id
FETCH FIRST 500 ROWS ONLY
```

**Cursor Pagination Pattern:**
```
First batch:  AND inv.id > 0  (or omit condition)
Second batch: AND inv.id > {last_id_from_batch_1}
Third batch:  AND inv.id > {last_id_from_batch_2}
...continue until query returns 0 rows
```

---

## Transaction Queries (Options 3, 4 & 5)

### Discovery Query T-A: Find Transactions by Date Range

**When to use:** Option 4 — user provides a date or date range without specific transaction numbers

**Purpose:** Find all matching transactions in the given date range that have on-hand inventory items.

**Note:** Convert user date input to YYYY-MM-DD. Apply type filter only if user specified a transaction type; otherwise omit the `t.type` condition.

```sql
SELECT DISTINCT
    t.id        AS transaction_id,
    t.tranid    AS transaction_number,
    t.type      AS transaction_type,
    t.trandate  AS transaction_date,
    e.entityid  AS entity_name
FROM transaction t
LEFT JOIN entity e ON t.entity = e.id
INNER JOIN transactionline tl ON tl.transaction = t.id
INNER JOIN item i ON tl.item = i.id
WHERE t.trandate >= TO_DATE('{start_date}', 'YYYY-MM-DD')
AND t.trandate <= TO_DATE('{end_date}', 'YYYY-MM-DD')
AND t.type IN ('{type_1}', '{type_2}')   -- omit if no type filter
AND tl.mainline = 'F'
AND tl.taxline = 'F'
AND i.itemtype IN ('InvtPart', 'Assembly')
ORDER BY t.trandate, t.tranid
```

**After running:** Report found transactions to user. Run Query T3 for each.

---

### Discovery Query T-B: Find Transactions by Entity and Date Range

**When to use:** Option 5 — user provides a vendor or customer name with a date range

```sql
SELECT DISTINCT
    t.id        AS transaction_id,
    t.tranid    AS transaction_number,
    t.type      AS transaction_type,
    t.trandate  AS transaction_date,
    e.entityid  AS entity_name
FROM transaction t
INNER JOIN entity e ON t.entity = e.id
INNER JOIN transactionline tl ON tl.transaction = t.id
INNER JOIN item i ON tl.item = i.id
WHERE t.trandate >= TO_DATE('{start_date}', 'YYYY-MM-DD')
AND t.trandate <= TO_DATE('{end_date}', 'YYYY-MM-DD')
AND t.type IN ('{type_1}', '{type_2}')   -- omit if no type filter
AND (UPPER(e.entityid) LIKE UPPER('%{entity_name}%')
     OR UPPER(e.altname) LIKE UPPER('%{entity_name}%'))
AND tl.mainline = 'F'
AND tl.taxline = 'F'
AND i.itemtype IN ('InvtPart', 'Assembly')
ORDER BY t.trandate, t.tranid
```

---

### Discovery Query T-C: Find Entity by Name

**Purpose:** Validate entity name and get exact match when user provides a partial name

```sql
SELECT 
    id,
    entityid    AS entity_name,
    altname,
    type
FROM entity
WHERE UPPER(entityid) LIKE UPPER('%{search_term}%')
OR UPPER(altname) LIKE UPPER('%{search_term}%')
ORDER BY entityid
FETCH FIRST 10 ROWS ONLY
```

---

### Query T1: Find Transaction by Number

**When to use:** Option 3 — user provides specific transaction number(s)

```sql
SELECT 
    t.id        AS transaction_id,
    t.tranid    AS transaction_number,
    t.type      AS transaction_type,
    t.trandate  AS transaction_date,
    e.entityid  AS entity_name
FROM transaction t
LEFT JOIN entity e ON t.entity = e.id
WHERE t.tranid = '{transaction_number}'
```

---

### Query T2: Count Items on Transaction

**Purpose:** Get total item/serial count for progress reporting

```sql
SELECT 
    SUM(CASE WHEN i.isserialitem = 'T' THEN 1 ELSE 0 END) AS serial_item_count,
    SUM(CASE WHEN i.isserialitem = 'F' THEN tl.quantity ELSE 0 END) AS nonserial_item_count
FROM transactionline tl
INNER JOIN item i ON tl.item = i.id
WHERE tl.transaction = {transaction_id}
AND tl.mainline = 'F'
AND tl.taxline = 'F'
AND i.itemtype IN ('InvtPart', 'Assembly')
```

---

### Query T3: Get Item/Location Groups (ALWAYS RUN THIS FOR TRANSACTION-BASED OPTIONS)

⚠️ **CRITICAL:** Always run this query before fetching serials or building transfers. It determines the processing strategy.

```sql
SELECT 
    tl.item                         AS item_id,
    i.itemid                        AS item_name,
    i.isserialitem,
    tl.location    AS source_location_id,
    l.name                          AS source_location_name,
    COUNT(DISTINCT ia.id)           AS serial_count,
    SUM(tl.quantity)                AS line_quantity
FROM transactionline tl
INNER JOIN transaction t ON tl.transaction = t.id
INNER JOIN item i ON tl.item = i.id
LEFT JOIN location l ON tl.location = l.id
LEFT JOIN inventoryassignment ia ON ia.transaction = tl.transaction 
    AND ia.transactionline = tl.linesequencenumber
WHERE tl.transaction = {transaction_id}
AND tl.mainline = 'F'
AND tl.taxline = 'F'
AND i.itemtype IN ('InvtPart', 'Assembly')
AND i.itemid IN ('{item_1}', '{item_2}')   -- omit if no item filter
GROUP BY tl.item, i.itemid, i.isserialitem, tl.location, l.name
ORDER BY source_location_id, item_id
```

**Use this data to:**
1. Determine whether each item is serialized or not
2. Process each item group separately
3. Know when source location changes (requires buffer flush)
4. Choose correct fetch strategy (all at once vs. cursor pagination)

**Important:** Use `COUNT(DISTINCT ia.id)` for serial counts — never `SUM(tl.quantity)`, which inflates by ~2× due to NetSuite sub-line structure.

---

### Query T4: Fetch Serials for Item on Transaction (All at Once)

> **⚠ DEPRECATED for transfer serial fetching.** Transaction serials may have moved since the bill/PO was created. Use **INV-B2** instead to fetch serials currently on-hand at the source location. T4 may still be used for informational or audit purposes only.

**When to use:** Informational/audit only — NOT for building transfer payloads for Options 3–5.

```sql
SELECT DISTINCT
    tl.item                         AS item_id,
    tl.location    AS source_location_id,
    ia.inventorynumber              AS inventory_number_id,
    inv.inventorynumber             AS serial_number
FROM transactionline tl
INNER JOIN transaction t ON tl.transaction = t.id
INNER JOIN item i ON tl.item = i.id
INNER JOIN inventoryassignment ia ON ia.transaction = tl.transaction 
    AND ia.transactionline = tl.linesequencenumber
INNER JOIN inventorynumber inv ON inv.id = ia.inventorynumber
WHERE tl.transaction = {transaction_id}
AND tl.mainline = 'F'
AND tl.taxline = 'F'
AND i.isserialitem = 'T'
AND tl.item = {item_id}
ORDER BY inventory_number_id
```

**Key Notes:**
- Use `ia.inventorynumber` directly (NOT `receiptinventorynumber` — that field does not exist)
- Use the `serial_number` column value (human-readable string) when building the transfer payload
- Use `SELECT DISTINCT` to avoid duplicates from multiple inventory assignment records

---

### Query T5: Fetch Serials for Item on Transaction (Cursor Pagination)

> **⚠ DEPRECATED for transfer serial fetching.** Transaction serials may have moved since the bill/PO was created. Use **INV-B3** instead to fetch serials currently on-hand at the source location. T5 may still be used for informational or audit purposes only.

**When to use:** Informational/audit only — NOT for building transfer payloads for Options 3–5.

⚠️ **CRITICAL:** Do NOT use OFFSET with SELECT DISTINCT. Do NOT use `pageSize` parameter.

```sql
SELECT DISTINCT
    tl.item                         AS item_id,
    tl.location    AS source_location_id,
    ia.inventorynumber              AS inventory_number_id,
    inv.inventorynumber             AS serial_number
FROM transactionline tl
INNER JOIN transaction t ON tl.transaction = t.id
INNER JOIN item i ON tl.item = i.id
INNER JOIN inventoryassignment ia ON ia.transaction = tl.transaction 
    AND ia.transactionline = tl.linesequencenumber
INNER JOIN inventorynumber inv ON inv.id = ia.inventorynumber
WHERE tl.transaction = {transaction_id}
AND tl.mainline = 'F'
AND tl.taxline = 'F'
AND i.isserialitem = 'T'
AND tl.item = {item_id}
AND ia.inventorynumber > {last_inventory_number_id}
ORDER BY inventory_number_id
FETCH FIRST 500 ROWS ONLY
```

**Cursor Pagination Pattern:**
```
First batch:  WHERE ia.inventorynumber > 0
Second batch: WHERE ia.inventorynumber > {last_id_from_batch_1}
...continue until query returns 0 rows
```

**Important:** Multi-transaction `IN (...)` clauses on heavy joins can cause SuiteQL stalls — query one transaction at a time for serial fetches.

---

### Query T6: Fetch Serials by Item AND Location on Transaction

> **⚠ DEPRECATED for transfer serial fetching.** Transaction serials may have moved since the bill/PO was created. Use **INV-B2/B3** with the location filter instead (INV-B2/B3 already filter by item AND location). T6 may still be used for informational or audit purposes only.

**When to use:** Informational/audit only — NOT for building transfer payloads for Options 3–5.

```sql
SELECT DISTINCT
    tl.item                         AS item_id,
    tl.location    AS source_location_id,
    ia.inventorynumber              AS inventory_number_id,
    inv.inventorynumber             AS serial_number
FROM transactionline tl
INNER JOIN transaction t ON tl.transaction = t.id
INNER JOIN item i ON tl.item = i.id
INNER JOIN inventoryassignment ia ON ia.transaction = tl.transaction 
    AND ia.transactionline = tl.linesequencenumber
INNER JOIN inventorynumber inv ON inv.id = ia.inventorynumber
WHERE tl.transaction = {transaction_id}
AND tl.mainline = 'F'
AND tl.taxline = 'F'
AND i.isserialitem = 'T'
AND tl.item = {item_id}
AND tl.location = {source_location_id}
AND ia.inventorynumber > {last_inventory_number_id}
ORDER BY inventory_number_id
FETCH FIRST 500 ROWS ONLY
```

---

## Utility Queries

### Query 7: Validate Location by Name

```sql
SELECT id, name, subsidiary, isinactive
FROM location
WHERE UPPER(name) LIKE UPPER('%{search_term}%')
AND isinactive = 'F'
ORDER BY name
```

---

### Query 8: Validate Serial at Location

**Purpose:** Check if a specific serial currently exists at a location (for troubleshooting)

```sql
SELECT 
    inv.id              AS inventory_number_id,
    inv.inventorynumber AS serial_number,
    nib.location        AS current_location_id,
    l.name              AS current_location_name,
    nib.quantityonhand
FROM inventorynumber inv
INNER JOIN inventorynumberlocation nib ON nib.inventorynumber = inv.id
INNER JOIN location l ON l.id = nib.location
WHERE inv.inventorynumber = '{serial_number}'
AND nib.quantityonhand > 0
```

---

### Item Resolution Query (when SKU matches multiple items)

**Only needed when the primary query returns ambiguous results.**

```sql
SELECT 
    i.id            AS item_id,
    i.itemid        AS item_name,
    i.displayname,
    i.isserialitem,
    i.itemtype,
    i.isinactive,
    il.location     AS location_id,
    l.name          AS location_name,
    il.quantityonhand,
    il.quantityavailable
FROM item i
LEFT JOIN inventoryitemlocations il ON il.item = i.id
LEFT JOIN location l ON l.id = il.location
WHERE UPPER(i.itemid) LIKE UPPER('%{item_sku}%')
AND i.isinactive = 'F'
AND i.itemtype IN ('InvtPart', 'Assembly')
AND (il.quantityonhand > 0 OR il.quantityonhand IS NULL)
ORDER BY 
    CASE WHEN UPPER(i.itemid) = UPPER('{item_sku}') THEN 0 ELSE 1 END,
    i.itemid
FETCH FIRST 10 ROWS ONLY
```

---

## Key Database Tables

| Table | Purpose |
|-------|---------|
| inventorynumber | Serial number master (human-readable serial → internal ID) |
| inventorynumberlocation | Current on-hand serials by location (works for all item types) |
| inventoryitemlocations | Current on-hand quantity by location (**InvtPart only — Assembly items do not appear**) |
| item | Item master (serial flag, item type, active status) |
| transaction | Transaction header (bill, PO, etc.) |
| transactionline | Transaction line items |
| inventoryassignment | Serial number assignments on transactions |
| location | Location master |
| entity | Vendor/customer master |

## Boolean Values in SuiteQL

| Value | Meaning |
|-------|---------|
| 'T' | True |
| 'F' | False |

## Transaction Type Strings

| Type String | Meaning |
|-------------|---------|
| `VendBill` | Vendor Bill |
| `PurchOrd` | Purchase Order |
| `ItemRcpt` | Item Receipt |
