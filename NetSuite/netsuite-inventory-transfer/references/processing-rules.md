# Processing Rules Reference

## Table of Contents
1. [Critical Behavioral Rules](#critical-behavioral-rules) — READ FIRST
2. [High-Level Processing Flows](#high-level-processing-flows) — Options 1–5
3. [File-Based Input Handling](#file-based-input-handling)
4. [Transfer Buffer Workflow](#transfer-buffer-workflow-per-location-group)
5. [Stream Processing](#stream-processing-for-serialized-batches-500-serials)
6. [Detailed Processing Algorithm](#detailed-processing-algorithm)
7. [Document Numbering During Execution](#document-numbering-during-execution)
8. [Resume Capability](#resume-capability)
9. [Error Handling](#error-handling)
10. [Progress Reporting](#progress-reporting)
11. [Context Management — NEVER/ALWAYS](#context-management--never-do-these)
12. [Processing Example](#processing-example-mixed-items-hitting-line-limit)

---

## Critical Behavioral Rules

> **These rules apply to EVERY transfer operation. Violating them is the most common failure mode.**

### Output Discipline
- **Do NOT narrate internal planning.** No buffer math, no line counting, no transfer strategy, no revalidation calculations. Plan silently, output only the four phases: Discovery → Confirmation → Execution → Completion.
- **Do NOT echo query results.** No SuiteQL output, no row counts, no raw data.
- **Do NOT show JSON payloads or serial number lists.**
- **Revalidation is silent when counts match.** Only show discrepancy notes when on-hand differs.

### Assembly Item Handling
- `inventoryitemlocations` does NOT work for Assembly items — it returns zero.
- **Serialized Assembly** → Use INV-B1s (queries `inventorynumberlocation`)
- **Non-serialized Assembly** → Use INV-B1a: `ns_getRecord` with `recordType: "assemblyitem"`, read `totalQuantityOnHand` field. This gives total on-hand across all locations (per-location not available via SuiteQL).
- Check `itemtype` from INV-B0 or T3. If `Assembly` + non-serialized → INV-B1a, never INV-B1.

### Revalidation (Options 3–5)
- Transactions are discovery-only — never trust transaction counts as current on-hand.
- Revalidate using: Serialized → INV-B1s | Non-serialized InvtPart → INV-B1 | Non-serialized Assembly → INV-B1a
- Use MIN(transaction count, on-hand count) for transfer quantity.
- Skip items with zero on-hand. Report discrepancies only when counts differ.

---

## High-Level Processing Flows

**Option 1 — By Serial Number (Inventory-First):**
```
1. User provides: Serial number(s), Destination, Date, (optional) Memo
           ↓
1a. IF no memo provided → ask: "Would you like to add a memo?" → WAIT for response
           ↓
2. Run INV-A1: Look up each serial in inventorynumber/inventorynumberlocation
   ⚠ If >500 serials, batch the IN clause (~500 per query).
   ⚠ MUST run ALL batches — do NOT skip batches or assume unqueried serials match.
   ⚠ BATCH INTEGRITY: Each batch MUST use the EXACT serial strings from the user's
     input (typed or file). Write batches to disk FIRST, then read each batch file
     when building the query. Do NOT reconstruct serial strings from memory.
     Do NOT generate or modify serial strings. If a serial appears "not found",
     verify the query used the exact string from the source before reporting it missing.
           ↓
3. Group results by source location and item
           ↓
4. Present confirmation summary → await user approval
           ↓
5. FOR EACH location group:
   └─ Run transfer buffer workflow → Create transfer(s)
           ↓
6. Report: "Complete: X serials in Y transfers"
```

**Option 2 — By Item + Source Location (Inventory-First):**
```
1. User provides: Item SKU(s), Source Location, Quantity, Destination, Date, (optional) Memo
           ↓
1a. IF no memo provided → ask: "Would you like to add a memo?" → WAIT for response
           ↓
2. Run INV-B0: Look up item metadata (serialized? item type?)
           ↓
3a. IF serialized → STOP. Do NOT auto-select serials.
    - Run INV-B1s to get on-hand count (for user reference)
    - Ask user: "{item} is serialized. I need the specific serial numbers.
      You can type them or upload a file. There are {count} on hand at {location}."
    - WAIT for user to provide serials (typed or file upload)
    - Once serials received → proceed as Option 1 (INV-A1 to verify each serial)
           ↓
3b. IF non-serialized → Run INV-B1: Verify item qty on-hand at source location
    - Validate quantityavailable ≥ requested quantity
           ↓
4. Present confirmation summary → await user approval
           ↓
5. FOR EACH item:
   └─ Run transfer buffer workflow → Create transfer(s)
           ↓
6. Report: "Complete: X items in Y transfers"
```

**Option 3 — Specific Transactions (Transaction-Based):**
```
1. User provides: Transaction #(s), Destination, Date, (optional) Item Filter, (optional) Memo
           ↓
1a. IF no memo provided → ask: "Would you like to add a memo?" → WAIT for response
           ↓
2. FOR EACH transaction number:
   └─ Run T1: Look up transaction → Get transaction_id
           ↓
3. Run T3: Get item/location groups (filtered by item if specified)
           ↓
4. REVALIDATE: For each item group from T3, verify current on-hand at source location
   ├─ Serialized → Run INV-B1s at source location; use MIN(T3 count, on-hand count)
   ├─ Non-serialized (InvtPart) → Run INV-B1 at source location; use MIN(T3 qty, on-hand qty)
   └─ Non-serialized (Assembly) → Run INV-B1a (ns_getRecord assemblyitem); verify totalQuantityOnHand ≥ T3 qty
   → Exclude items with zero on-hand; note discrepancies
           ↓
5. Present confirmation summary (with discrepancy notes if any) → await user approval
           ↓
6. FOR EACH item group: Run transfer buffer workflow → Create transfer(s)
           ↓
7. Report: "Complete: X transactions processed, Y items in Z transfers"
```

**Option 4 — Transactions by Date:**
```
1. User provides: Date range, Transaction Type, Destination, Transfer Date, (optional) Memo
           ↓
1a. IF no memo provided → ask: "Would you like to add a memo?" → WAIT for response
           ↓
2. Run T-A: Find matching transactions in date range
           ↓
3. Report: "Found X transactions: BILL-001, PO-002, ..."
           ↓
4. FOR EACH transaction: Run T3 → Get item/location groups
           ↓
5. REVALIDATE: For each item group from T3, verify current on-hand at source location
   ├─ Serialized → Run INV-B1s at source location; use MIN(T3 count, on-hand count)
   ├─ Non-serialized (InvtPart) → Run INV-B1 at source location; use MIN(T3 qty, on-hand qty)
   └─ Non-serialized (Assembly) → Run INV-B1a (ns_getRecord assemblyitem); verify totalQuantityOnHand ≥ T3 qty
   → Exclude items with zero on-hand; note discrepancies
           ↓
6. Present confirmation summary (with discrepancy notes if any) → await user approval
           ↓
7. FOR EACH item group: Run transfer buffer workflow → Create transfer(s)
           ↓
8. Report: "Complete: X transactions processed, Y items in Z transfers"
```

**Option 5 — Transactions by Vendor/Customer and Date:**
```
1. User provides: Entity name, Date range, Transaction Type, Destination, Transfer Date, (optional) Memo
           ↓
1a. IF no memo provided → ask: "Would you like to add a memo?" → WAIT for response
           ↓
2. Run T-B: Find matching transactions by entity and date
           ↓
3. Report: "Found X transactions: BILL-001, BILL-002, ..."
           ↓
4. FOR EACH transaction: Run T3 → Get item/location groups
           ↓
5. REVALIDATE: For each item group from T3, verify current on-hand at source location
   ├─ Serialized → Run INV-B1s at source location; use MIN(T3 count, on-hand count)
   ├─ Non-serialized (InvtPart) → Run INV-B1 at source location; use MIN(T3 qty, on-hand qty)
   └─ Non-serialized (Assembly) → Run INV-B1a (ns_getRecord assemblyitem); verify totalQuantityOnHand ≥ T3 qty
   → Exclude items with zero on-hand; note discrepancies
           ↓
6. Present confirmation summary (with discrepancy notes if any) → await user approval
           ↓
7. FOR EACH item group: Run transfer buffer workflow → Create transfer(s)
           ↓
8. Report: "Complete: X transactions processed, Y items in Z transfers"
```

---

## File-Based Input Handling

When the user uploads a spreadsheet (`.xlsx`, `.csv`, `.tsv`), parse it before determining the request type. Use the file-reading skill or built-in tools to read the file contents.

### Step 1: Check for Known File Format

Before fuzzy matching, check if the file headers match a known format. Known formats auto-map without requiring user confirmation of column mapping.

#### Known Format: Vendor Bill SN List

**How to recognize:** Row 1 headers include ALL of these columns (exact match, case-insensitive): `SERIAL`, `ITEM/PN`, `QUANTITY`

**⚠ Do NOT confuse `SERIAL` with `Old Serial 1`, `Old Serial/MAC`, or similar columns.** The exact column header `SERIAL` is the one to extract from. Other columns containing the word "serial" are historical reference data.

**Column mapping (auto-applied):**

| Transfer Input | Column Header | Notes |
|----------------|--------------|-------|
| **Serial** | `SERIAL` | The serial numbers — drives Option 1 flow |
| **Item** | `ITEM/PN` | Informational only — may not match NetSuite SKU exactly (e.g., "Widget Pro HW" in file vs "WIDGET-100" in NetSuite). Do NOT use for item lookup — INV-A1 resolves the correct item from the serial. |
| **Quantity** | `QUANTITY` | Always 1 per row for serialized items — not used directly |

**Other columns present but not used for transfers:** `Invoice #`, `VENDOR`, `Memo`, `Bill Date`, `MAC ID`, `COO`, `Location`, `Component 1/2`, `Public Key`, `Signature`, `ICCID`, `IMEI`, `CARTON #`, `PALLET #`, `Lot Code/LOTCODE`, `Old Serial/MAC` columns.

**When this format is detected:**
1. Extract all non-empty values from the `SERIAL` column
2. Deduplicate (keep unique values only)
3. Report what was found (use the auto-map report template from output-templates.md)
4. Proceed directly to Option 1 flow — no column confirmation needed
5. Destination and transfer date come from the user's message

### Step 2: Fuzzy Matching (Unknown Formats)

If the file does NOT match a known format, use fuzzy column detection.

Match column headers against these patterns. Try exact match first, then partial (contains). Stop at the first match per category.

**⚠ CRITICAL: Column Exclusion List — Do NOT match these as the Serial column:**
Columns containing these terms are historical/reference data, NOT the current serial to transfer:
- `Old Serial`, `Old Serial 1`, `Old Serial 2`, `Previous Serial`, `Former Serial`, `Original Serial`
- `Old MAC`, `Old Serial/MAC`

**Priority rule:** If multiple columns contain the word "serial", always prefer the exact match `SERIAL` (or `Serial`, `serial`) over any compound name like `Old Serial 1`. An exact match on the single word always wins.

| Transfer Input | Match These Column Names (case-insensitive) |
|----------------|---------------------------------------------|
| **Serial** | `serial`, `serial number`, `serial numbers`, `serial #`, `serial no`, `serial no.`, `sn`, `serials` |
| **Item** | `item`, `item number`, `item #`, `item no`, `item name`, `item sku`, `sku`, `part`, `part number`, `part #`, `part no`, `product`, `product number`, `item/pn`, `pn` |
| **Source Location** | `from location`, `source location`, `from`, `source`, `ship from`, `origin`, `from warehouse`, `current location`, `src`, `from loc` |
| **Destination Location** | `to location`, `destination location`, `to`, `destination`, `dest`, `ship to`, `transfer to`, `to warehouse`, `dest location`, `to loc` |
| **Quantity** | `quantity`, `qty`, `count`, `units`, `amount`, `qty to transfer`, `transfer qty` |

**Partial match fallback:** If no exact header match, check if any column header *contains* the key word (`serial`, `item`, `from`, `to`, `qty`). **Exclude columns on the exclusion list above** (e.g., "Old Serial 1" contains "serial" but is NOT the serial column). If multiple non-excluded columns match the same category, ask the user to clarify.

### Step 3: Confirm Column Mapping (Unknown Formats Only)

For unknown formats, present the mapping and wait for confirmation:

```
I found these columns in your file:
• Serial → "{column_name}" ({row_count} values)
• Item → "{column_name}"
• From Location → "{column_name}"
• To Location → "{column_name}"
• Quantity → "{column_name}"

Does this mapping look correct?
```

Only list columns that were actually detected. If the user's message already provides some values (e.g., destination), note those alongside the file columns.

**Known formats skip this step** — they proceed directly to the Option 1/2 flow after the auto-map report.

### No Match for a Column

If a needed column cannot be matched:
- Check whether the user's message provides the value directly (e.g., destination in the prompt text)
- If not, ask: `"I couldn't find a [Serial/Item/Source/Destination/Quantity] column. Which column should I use, or can you provide that value?"`

### Determining the Request Type from File Data

| Columns Found | Request Type | Notes |
|---------------|-------------|-------|
| Serial (+ optional From, To) | **Option 1** — By Serial Number | Source location comes from inventory lookup (INV-A1), not the file |
| Item + Source + Quantity (no Serial) | **Option 2** — By Item + Source Location | Destination and date from user's message |
| Serial + Item (both present) | **Option 1** — By Serial Number | Serial takes precedence; Item column is informational only |
| Only Item (no Source, no Serial) | Ask for source location | Cannot determine Option without source |

### Rules

- **Transfer date and memo are never read from the file.** Always from the user's message.
- **Destination from file is accepted** but must still be resolved via Query 7 if it's a name (not an ID).
- **Empty cells are skipped.** Only non-empty values in the mapped columns are used.
- **Duplicate serials in the file are deduplicated** before processing (keep unique values only).
- After mapping (auto or confirmed), proceed with the normal Option 1 or Option 2 flow — the file is just a source of input data, not a different processing path.

---

## Transfer Buffer Workflow (Per Location Group)

```
1. Discovery query returns item groups
           ↓
1a. [OPTIONS 3–5 ONLY] REVALIDATE each item group against current on-hand:
    ├─ Serialized → INV-B1s at source location → use MIN(T3 count, on-hand)
    ├─ Non-serialized (InvtPart) → INV-B1 at source location → use MIN(T3 qty, on-hand)
    └─ Non-serialized (Assembly) → INV-B1a (ns_getRecord assemblyitem) → verify totalQuantityOnHand ≥ T3 qty
    → Skip items with zero on-hand; note discrepancies in confirmation
           ↓
2. FOR EACH ITEM GROUP (ordered by location, then lines needed DESCENDING):
   > Sort largest items first within each location group. This is bin-packing:
   > large items (many serial lines) get their own transfers, small items
   > pack together in remaining space. Reduces total transfer count.
   ├─ Fetch serials for this item (use cursor if >500) [serialized only]
   ├─ Add to transfer buffer
   ├─ When buffer ≥500 OR line_count ≥5 OR location changes → Create transfer
   └─ Report: "✓ Transfer N (ID: X, Doc: Y)"
           ↓
3. Create final transfer for any remaining serials/items in buffer
           ↓
4. Report completion with grouped links
```

---

## Stream Processing for Serialized Batches (>500 Serials)

When a single item requires more than 500 serials (i.e., multiple transfers), **do NOT fetch all serials upfront**. Fetching thousands of serials into the context window causes memory pressure, duplicate serials across line boundaries, and payload mismatches.

**Use the fetch-create-discard cycle:**

```
// PLANNING PHASE (lightweight — counts only, no serial strings)
Use INV-B1s count or T3 serial_count to determine:
  - total_serials_needed for this item
  - transfers_needed = CEILING(total_serials_needed / 500)

// EXECUTION PHASE (stream — max 500 serial strings in memory at any time)
cursor_id = 0
FOR transfer_n = 1 TO transfers_needed:
    serials_this_batch = MIN(500, remaining_serials)
    
    // FETCH: Get exactly this batch using cursor
    serials = Query INV-B3 with WHERE id > {cursor_id} FETCH FIRST {serials_this_batch} ROWS ONLY
    cursor_id = last id from results
    
    // BUILD: Construct payload directly from fetched serials
    // (apply pre-submission validation)
    
    // CREATE: Submit transfer immediately
    CREATE TRANSFER with these serials
    REPORT "✓ Transfer {N}"
    
    // DISCARD: Do NOT retain serial strings — they are no longer needed
    // The cursor_id is the only state carried forward to the next iteration
```

**Key rules:**
- The **only state** carried between transfers is `cursor_id` (a single integer) and `transfer_num`
- Serial strings from a completed transfer must NOT be referenced or re-read in subsequent iterations
- This applies to all flows: Option 1 (when >500 serials provided), Option 2, and Options 3–5

---

## Detailed Processing Algorithm

```
Initialize:
  transfer_buffer = {
      source_location_id: null,
      items: {},           // {item_id: [serial_ids...]} or {item_id: qty}
      total_count: 0,
      line_count: 0        // Track planned lines — flush at 5
  }
  transfer_num = 1

FOR EACH item_group IN results (ordered by location, then lines_needed DESCENDING):
    // Sort largest items first within each location for optimal bin-packing.
    // lines_needed = CEILING(serial_count / 100) for serialized, 1 for non-serialized.
    
    // STEP A: Handle location changes (flush buffer first)
    IF buffer.source_location_id != item_group.source_location_id:
        IF buffer.total_count > 0:
            CREATE TRANSFER from buffer
            REPORT "✓ Transfer {N} (ID: X, Doc: Y)"
            CLEAR buffer (reset total_count, line_count, items)
            transfer_num++
        buffer.source_location_id = item_group.source_location_id
    
    // STEP B: Fetch serials for this item (serialized items only)
    IF item is serialized:
        IF item_group.serial_count <= 500:
            serials = Fetch all serials (INV-B2)
        ELSE:
            serials = Fetch serials using cursor pagination (INV-B3)
        
        // STEP B1: Calculate how many lines this item needs
        lines_needed = CEILING(serials.length / 100)
        
        // STEP B2: If adding this item would exceed 5 lines, flush buffer first
        IF buffer.line_count + lines_needed > 5 AND buffer.total_count > 0:
            CREATE TRANSFER from buffer
            REPORT "✓ Transfer {N} (ID: X, Doc: Y)"
            CLEAR buffer
            transfer_num++
            buffer.source_location_id = item_group.source_location_id
        
        buffer.items[item_group.item_id] = serials
        buffer.total_count += serials.length
        buffer.line_count += lines_needed
    ELSE:
        // Non-serialized: one line per item, full quantity
        lines_needed = 1
        
        IF buffer.line_count + lines_needed > 5 AND buffer.total_count > 0:
            CREATE TRANSFER from buffer
            REPORT "✓ Transfer {N} (ID: X, Doc: Y)"
            CLEAR buffer
            transfer_num++
            buffer.source_location_id = item_group.source_location_id
        
        buffer.items[item_group.item_id] = item_group.qty_to_transfer
        buffer.total_count += item_group.qty_to_transfer
        buffer.line_count += lines_needed
    
    // STEP C: Create transfers when buffer is full (count OR lines)
    WHILE buffer.total_count >= 500 OR buffer.line_count > 5:
        CREATE TRANSFER with up to 500 units / 5 lines from buffer
        REPORT "✓ Transfer {N} (ID: X, Doc: Y)"
        CLEAR buffer for next batch
        transfer_num++

// STEP D: Final transfer for remaining items
IF buffer.total_count > 0:
    CREATE TRANSFER from buffer
    REPORT "✓ Transfer {N} (ID: X, Doc: Y)"

// STEP E: Output grouped links for all completed transfers
REPORT "Transfer Records:" + link for each transfer
```

> **Key rule:** A transfer is flushed when ANY of these conditions is met: (1) total count reaches 500, (2) line count reaches 5, or (3) source location changes.

---

## Document Numbering During Execution

```
// After creating each transfer:
IF transfer_num == 1:
    response = CREATE TRANSFER (no tranid)
    base_doc_number = READ tranid FROM response
    // If response doesn't include tranid, call ns_getRecord to read it back
ELSE:
    tranid = "{base_doc_number}-{transfer_num}"
    response = CREATE TRANSFER (with tranid)
```

---

## Resume Capability

When user says: "Resume {reference} from transfer {N}"

1. Re-run the discovery query (T3 or INV-A1/B1s) to get item groups
2. Calculate: (N−1) × 500 = items already transferred
3. Skip items/serials until reaching the correct position
4. Continue normal processing

---

## Error Handling

### Error Categories and Detection

#### Discovery Errors (Transaction-Based Options)

| Error | How to Detect | Response |
|-------|---------------|----------|
| No transactions found for date | Discovery Query T-A returns empty | Use exact error template |
| No transactions found for entity | Discovery Query T-B returns empty | Use exact error template |
| Entity not found | Discovery Query T-C returns empty | Use exact error template with suggestions |
| Multiple entity matches | Discovery Query T-C returns multiple | Use exact error template with list |

#### Inventory Lookup Errors (Inventory-First Options)

| Error | How to Detect | Response |
|-------|---------------|----------|
| Serial number not found | INV-A1 returns empty | Use exact error template |
| Item not found | INV-B0 returns empty | Use exact error template |
| Item not at source location (non-serial) | INV-B1 returns empty | Use exact error template |
| Item not at source location (serialized) | INV-B1s returns empty | Use exact error template |
| Insufficient quantity (non-serial) | INV-B1 `quantityavailable` < requested | Use exact error template, ask to proceed |
| Insufficient quantity (serialized) | INV-B1s `quantityonhand` < requested | Use exact error template, ask to proceed |

#### Revalidation Discrepancies (Transaction-Based Options 3, 4, 5)

| Discrepancy | How to Detect | Response |
|-------------|---------------|----------|
| On-hand lower than transaction count (serialized) | INV-B1s returns count < T3 serial_count | Use revalidation discrepancy template (serialized) |
| On-hand lower than transaction count (non-serialized) | INV-B1 returns qty < T3 line_quantity | Use revalidation discrepancy template (non-serialized) |
| Item no longer on hand at source location | INV-B1/INV-B1s returns 0 or empty | Use "no longer on hand" template, skip item |
| All items from transaction unavailable | All items return 0 on-hand | Use "no inventory items currently on hand" template, skip transaction |

> **Handling:** Discrepancy notes appear in the confirmation summary, before the user approves. If some items are still available, proceed with the available quantities. If all items are unavailable, skip the transaction entirely. The adjusted counts (not the transaction counts) are used for transfer planning.

#### Data Not Found

| Error | How to Detect | Response |
|-------|---------------|----------|
| Transaction not found | Query T1 returns empty | Use exact error template |
| No inventory items on transaction | Query T2 returns 0 | Use exact error template |
| Location not found | Query 7 returns empty | Use exact error template with active location list |

#### API and Batch Errors

| Error | How to Detect | Response |
|-------|---------------|----------|
| Rate limit (429) | HTTP 429 response | Wait 5 seconds, retry once |
| Server error (500) | HTTP 500 response | Log error, report to user |
| Transfer fails | API error on `ns_createRecord` | Log error, report to user, **STOP** |
| Cursor returns same data | `last_id` not advancing | Log error, verify query |
| Query returns 0 mid-pagination | Empty result during cursor iteration | Verify with count query, report to user |

---

## Progress Reporting

**Correct (minimal):**
```
Processing BILL-12345 (1,500 serials → 3 transfers)...
✓ Transfer 1 (ID: 16500, Doc: IT-5001) - 500 serials
✓ Transfer 2 (ID: 16501, Doc: IT-5001-2) - 500 serials
✓ Transfer 3 (ID: 16502, Doc: IT-5001-3) - 500 serials

Complete: 1,500 serials in 3 transfers.
```

**Wrong (context bloat — never do this):**
```
Query T3 returned 500 rows:
[500 rows displayed]

Building payload:
[27KB JSON displayed]

API response:
[full response displayed]
```

**Also wrong (narrating internal planning — never do this):**
```
All serials fetched. Now building and creating transfers. Planning the buffer:
* WIDGET-400: 368 serials → 4 lines
* WIDGET-600: 429 serials → 5 lines
Buffer plan: WIDGET-400 (4 lines) + WIDGET-600 would be 5 more lines = 9 total → exceeds 5...
```
Plan internally, then report only the confirmation summary and transfer results.

---

## Context Management — NEVER Do These

| ❌ Don't | Why |
|----------|-----|
| Auto-select AND auto-submit serials without showing the user | Claude CAN query and display available serials to help the user choose. But the user must SEE the list and explicitly confirm before any transfer is created. Showing → confirming → transferring is OK. Silently picking → transferring is not. Options 3–5 exempt (transaction defines the serials). |
| Query more than 500 serials at a time | Exceeds safe limits |
| Create transfers with >500 serials | API timeouts |
| Split non-serialized item qty across multiple lines | Unnecessary — use 1 line per item |
| Create transfers with >5 inventory lines | NetSuite line limit |
| Use literal backslash+n text (`"\\n"`) when building `serialNumbers` — causes double-escaping | Use actual newline chars: `"\n".join(serials)`. MCP tool handles JSON escaping. |
| Narrate buffer planning, line counting, or transfer strategy in chat | Bloats context window |
| Echo query results | Bloats context window |
| Display transfer payload JSON | Bloats context window |
| List serial numbers | Bloats context window |
| Accumulate serials across batches | Memory/context limits |
| Fetch all serials upfront when total >500 | Context overflow causes duplicates |
| Retain serial strings from completed transfers | Discard immediately |
| Skip memo prompt when user didn't provide one | Always ask as FIRST step before any queries |
| Skip INV-A1 verification batches for Option 1 | Must verify ALL user-provided serials before confirmation — do NOT assume unqueried serials match |
| Reconstruct or generate serial strings from memory when building INV-A1 batch queries | Write all serials to disk first, then read each batch from file. Hallucinated serial strings cause false "not found" errors and backtracking. |
| Reconstruct serial strings from memory when building INV-A1 batch queries | Always build IN clauses from the stored serial list using index ranges. Reconstructing from memory causes batch errors and false "not found" results. |
| Use OFFSET with SELECT DISTINCT | Fails silently |
| Use `pageSize` on serial fetch queries | Triggers broken server-side OFFSET |
| Mix source locations in one transfer | Each transfer needs one source |
| Use transaction lookup when serial numbers or item+location are provided | Inventory table is the correct source |
| Trust transaction counts as current on-hand for Options 3–5 | Transactions are discovery-only — inventory may have moved since the transaction was created |
| Create transfers without user confirmation | Confirmation required before any write |
| Ask user to pick an item ID when multiple matches exist | Auto-filter first |
| Add conversational filler | Use exact output templates only |

## Context Management — ALWAYS Do These

| ✅ Do | Why |
|-------|-----|
| Ask for memo as FIRST response if user didn't provide one | Must happen before any discovery queries — wait for response |
| Verify ALL serials via INV-A1 (batched ~500 per query) for Option 1 | Write ALL serials to a file on disk FIRST (e.g., `/home/claude/serials.txt`). Then read batches from that file when building each query. Do NOT reconstruct serial strings from memory — this causes hallucinated serials and false "not found" errors. Use index ranges to slice: `serials[0:500]`, `serials[500:1000]`, etc. |
| For Options 3–5, extract serials from the transaction — this is expected behavior | The "no auto-select" rule only applies to Option 2. Transactions define which serials are involved, so Claude should fetch them. |
| Query inventory tables first for Options 1 & 2 | Master on-hand data is authoritative |
| Run INV-B0 first for Option 2 | Determines serialized vs non-serialized path |
| Run T3 first for transaction-based options | Get item/location groups for planning |
| Revalidate transaction-sourced items against current on-hand (INV-B1/INV-B1s) before creating transfers | Transactions show historical data — on-hand may have changed |
| Present confirmation summary before creating transfers | Prevents unintended writes |
| Process items individually | Proper pagination strategy |
| Auto-filter item matches | Reduces ambiguity without user input |
| Use cursor pagination for >500 serials | `WHERE id > {last_id}` works correctly |
| Flush buffer when location changes | Transfers need single source location |
| Group serials into lines (100 max each) | NetSuite serial line limit |
| Use 1 line per non-serialized item with full quantity | Payloads don't fail at high quantities |
| Count lines before creating transfer; flush at 5 | Exceeding 5 lines causes failures |
| Use stream processing for serialized batches >500 | Prevents context overflow |
| Validate serial count per line matches adjustQtyBy | Catches errors before NetSuite |
| Build serialNumbers with actual newline chars: `"\n".join(serials)` — NOT `"\\n".join()` | MCP tool handles JSON escaping; `"\\n"` causes double-escape bug |
| Report only: "✓ Transfer N (ID: X, Doc: Y)" | Minimal context usage |
| Auto-generate memo | Traceability — always prompt user for memo before confirmation if not provided |
| Use sequential doc numbering for multi-transfer batches | Links related transfers |
| Support resume from transfer N | Error recovery |
| Use exact text from output templates | Consistency |
| Use `inventorynumberlocation` for serialized Assembly items | `inventoryitemlocations` returns zero for Assembly |

---

## Mixing Serialized and Non-Serialized Items

Serialized and non-serialized items **always** go on the same transfer when they share the same source location and total lines ≤ 5. The buffer accumulates both types — the `build_lines` function in transfer-creation.md produces a single `items` array containing both serialized lines (with `serialNumbers`) and non-serialized lines (without). Never split them into separate transfers unless line limits require it.

### Processing Example: Mixed Items Fitting on One Transfer

**Scenario:** Transaction has 2 non-serialized items (40 qty + 30 qty) + 200 serials of item C, all at same location. Lines needed = 1 + 1 + 2 (200 ÷ 100) = **4 lines**, which fits.

**Processing steps:**
1. Process non-serial item 1 (40 qty): buffer = 1 line, 40 count
2. Process non-serial item 2 (30 qty): buffer = 2 lines, 70 count
3. Process item C (200 serials): buffer = 2 + 2 = 4 lines, 270 count → OK
4. End of items → **Transfer 1** (40 non-serial + 30 non-serial + 200 serials = 270 items, 4 lines)

**Result:** 1 transfer with mixed item types. All 3 items share the payload.

### Processing Example: Mixed Items Hitting Line Limit

**Scenario:** Transaction has 1 non-serialized item (50 qty) + 368 serials of item A + 82 serials of item B, all at same location. Total = 500 items, but lines needed = 1 (non-serial) + 4 (368 ÷ 100) + 1 (82) = **6 lines**, which exceeds the 5-line max.

**Processing steps (with line counting):**
1. Process non-serial item (50 qty):
   - lines_needed = 1
   - buffer: 0 + 1 = 1 line, 50 count → OK
2. Process item A (368 serials):
   - lines_needed = CEILING(368/100) = 4
   - buffer: 1 + 4 = 5 lines, 418 count → OK (exactly at limit)
3. Process item B (82 serials):
   - lines_needed = CEILING(82/100) = 1
   - buffer: 5 + 1 = **6 lines → exceeds 5!**
   - **Flush buffer** → **Transfer 1** (50 non-serial + 368 serials = 418 items, 5 lines)
   - Then add item B to fresh buffer: 1 line, 82 count
   - End of items → **Transfer 2** (82 serials, 1 line)

**Result:** 2 transfers instead of 1, even though total count (500) fits in one transfer. The 6 lines required exceed the 5-line max.
