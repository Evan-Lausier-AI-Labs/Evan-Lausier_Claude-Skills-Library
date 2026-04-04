# Standardized Output Templates

All user-facing output must follow the exact templates below. Do not paraphrase, reorder, or add conversational filler. The only variable parts are the values inside `{braces}`. This ensures identical output across sessions and instances.

---

## File Input Templates (Before Discovery)

**Known format auto-map report (Vendor Bill SN List — no confirmation needed):**
```
Read {row_count} serials from file ({item_context}). Proceeding with Option 1.
```
Examples:
- `Read 568 serials from file (WIDGET-500). Proceeding with Option 1.`
- `Read 90 serials from file (Widget Pro HW). Proceeding with Option 1.`

Then proceed directly to Phase 1 Discovery (INV-A1 lookup).

**Unknown format — column mapping confirmation (wait for user approval):**
```
I found these columns in your file:
• Serial → "{column_name}" ({row_count} values)
• Item → "{column_name}"
• From Location → "{column_name}"
• To Location → "{column_name}"
• Quantity → "{column_name}"

Does this mapping look correct?
```
Only list columns that were detected — omit rows for columns not found.

**Column not found (when needed):**
```
I couldn't find a {column_type} column. Which column should I use, or can you provide that value?
```

---

## Memo Prompt (FIRST Step — Before Any Queries)

**If the user did NOT include a memo in their request, this is the VERY FIRST thing Claude outputs — before any discovery queries, file parsing, or lookups.**

```
Would you like to add a memo to this transfer? If not, I'll use the default: "Claude | Transfer {N} of {total}".
```

Wait for the user to respond:
- If they provide memo text directly (e.g., "yes, use 'Q1 Receipt'" or just "January stock"), use that text as the memo.
- If they say yes without providing memo text (e.g., "yes" or "yes please"), ask: "What memo would you like to use?" and wait for their response.
- If they say no (or equivalent), proceed with the default.

Only AFTER the memo is resolved should Claude begin discovery queries.

**If the user DID include a memo** (e.g., "memo: Q1 Receipt" or "add memo 'January stock'"), skip this prompt and proceed directly to discovery.

---

## Phase 1: Discovery Output

**Option 1 — Serial lookup result (after INV-A1):**
```
Found {serial_count} {serials|serial} for {item_count} {items|item} at {location_count} {locations|location}.
```
Examples:
- `Found 3 serials for 1 item at 1 location.`
- `Found 5 serials for 2 items at 2 locations.`

**Option 2 — Non-serialized item lookup result (after INV-B1):**
```
Found {quantity} units of {item_name} on hand at {source_location}.
```

**Option 2 — Serialized item lookup result (after INV-B1s):**
```
Found {serial_count} serials of {item_name} on hand at {source_location}.
```

**Option 2 — Multiple items lookup result:**
```
Found {item_count} items at {source_location}:
• {item_name_1}: {quantity_or_serial_count} {units|serials}
• {item_name_2}: {quantity_or_serial_count} {units|serials}
```

**Options 3/4/5 — Transaction discovery (after T-A, T-B, or T1):**
```
Found {transaction_count} {transaction_type_plural}: {transaction_list}.
```

**Options 3/4/5 — Item groups on transaction (after T3):**
```
{transaction_number} contains {item_count} inventory {items|item} ({total_count} {serials|items} total) at {location_count} source {locations|location}.
```

---

## Phase 2: Confirmation Summary (Required Before Creating Transfers)

**Always present the confirmation summary in one of the two formats below and wait for explicit user approval before creating any transfer records.**
**Do not proceed without approval.**
**Do not add extra commentary.**
**The memo has already been resolved in the Memo Prompt step above.**

### Format A — Simple Confirmation (Option 1 with few serials only)

Use this format ONLY when the request is Option 1 (by serial number) AND the total serial count is small enough to list individually (roughly 10 or fewer serials). List each serial explicitly.

```
* SKU: {item_name}
* Serial: {serial_1}
* Serial: {serial_2}
* Serial: {serial_3}
* From: {source_location} (ID: {source_location_id})
* To: {destination} (ID: {destination_id})
* Date: {transfer_date}
* Memo: {memo_text}
* Transfers: {transfer_count} {transfers|transfer}, {serial_count} {serials|serial}

Submit?
```

If multiple SKUs are present, group serials under each SKU:
```
* SKU: {item_name_1}
  * Serial: {serial_1}
  * Serial: {serial_2}
* SKU: {item_name_2}
  * Serial: {serial_3}
* From: {source_location} (ID: {source_location_id})
* To: {destination} (ID: {destination_id})
* Date: {transfer_date}
* Memo: {memo_text}
* Transfers: {transfer_count} {transfers|transfer}, {serial_count} {serials|serial}

Submit?
```

If serials come from multiple source locations, list each From separately:
```
* SKU: {item_name}
  * Serial: {serial_1} — From: {source_location_1} (ID: {id_1})
  * Serial: {serial_2} — From: {source_location_2} (ID: {id_2})
* To: {destination} (ID: {destination_id})
* Date: {transfer_date}
* Memo: {memo_text}
* Transfers: {transfer_count} {transfers|transfer}, {serial_count} {serials|serial}

Submit?
```

### Format B — Full Table Confirmation (All other cases)

Use this format for ALL cases except Format A above. This includes:
- Option 1 with many serials (too many to list individually)
- Option 2 (all variants — serialized, non-serialized, single, multiple)
- Option 3 (specific transactions)
- Option 4 (transactions by date)
- Option 5 (transactions by entity and date)

**If a field is not applicable (e.g., Transaction for Options 1 & 2, or Serials for non-serialized items), leave it blank.**

```
{transfer_count} {transfers|transfer} will be created to {destination}.
{item_count} {items|item}, {total_count} total {serials|units}, across {location_count} source {locations|location}.

| Transaction | Source | Item | Type | Count | Count Type | Serials | Transfers |
|-------------|--------|------|------|-------|------------|---------|-----------|
| {transaction_number_or_blank} | {source_location} | {item_name} | {serialized|non-serialized} | {count} | {serials|units} | {serial_list_or_blank} | {transfer_row_count} |

Transfer date: {transfer_date}
Memo: {memo_text}
Submit?
```

**Column definitions:**
- **Transaction**: The source transaction number (BILL-12345, PO-123, etc.). Blank for Options 1 & 2.
- **Source**: The source location name.
- **Item**: The item SKU/name.
- **Type**: "serialized" or "non-serialized".
- **Count**: Number of serials or units for this row.
- **Count Type**: "serials" or "units" matching the Type.
- **Serials**: Individual serial numbers if count is small enough to list; blank for non-serialized or large serialized batches.
- **Transfers**: Number of transfers this row will produce.

**Examples:**

Option 2 — non-serialized, single item:
```
1 transfer will be created to WH-CENTRAL.
1 item, 250 total units, across 1 source location.

| Transaction | Source | Item | Type | Count | Count Type | Serials | Transfers |
|-------------|--------|------|------|-------|------------|---------|-----------|
| | WH-EAST | ITEM-D | non-serialized | 250 | units | | 1 |

Transfer date: 2026-02-01
Memo: Claude | Transfer 1 of 1
Submit?
```

Option 3 — mixed serialized and non-serialized on one transfer:
```
1 transfer will be created to Main Warehouse.
3 items, 290 total items, across 1 source location.

| Transaction | Source | Item | Type | Count | Count Type | Serials | Transfers |
|-------------|--------|------|------|-------|------------|---------|-----------|
| BILL-12345 | TRANSIT | ITEM-A | non-serialized | 40 | units | | 1 |
| BILL-12345 | TRANSIT | ITEM-E | non-serialized | 50 | units | | 1 |
| BILL-12345 | TRANSIT | WIDGET-200 | serialized | 200 | serials | | 1 |

Transfer date: 2026-01-15
Memo: Claude | Transfer 1 of 1
Submit?
```
(200 serials = 2 lines + 2 non-serialized items = 2 lines → 4 lines total, fits on 1 transfer.)

Option 3 — mixed serialized and non-serialized, exceeds line limit (2 transfers):
```
2 transfers will be created to Main Warehouse.
2 items, 750 total items, across 1 source location.

| Transaction | Source | Item | Type | Count | Count Type | Serials | Transfers |
|-------------|--------|------|------|-------|------------|---------|-----------|
| BILL-12345 | TRANSIT | Widget Pro | serialized | 500 | serials | | 1 |
| BILL-12345 | TRANSIT | ITEM-A | non-serialized | 250 | units | | 1 |

Transfer date: 2026-01-15
Memo: Claude | Transfer 1 of 2
Submit?
```
(500 serials = 5 lines + 1 non-serialized = 6 lines → exceeds 5-line limit, requires 2 transfers.)

Option 4 — multiple transactions by date:
```
2 transfers will be created to Main Warehouse.
2 items, 800 total units, across 1 source location.

| Transaction | Source | Item | Type | Count | Count Type | Serials | Transfers |
|-------------|--------|------|------|-------|------------|---------|-----------|
| BILL-12345 | TRANSIT | Widget A | non-serialized | 500 | units | | 1 |
| BILL-12346 | TRANSIT | Widget B | non-serialized | 300 | units | | 1 |

Transfer date: 2026-01-15
Memo: Claude | Transfer 1 of 2
Submit?
```

---

## Phase 3: Execution Output

**Processing header (single item/source, Options 1 & 2):**
```
Processing {item_name_or_serial_list} from {source_location} ({count} {serials|items} → {transfer_count} {transfers|transfer})...
```

**Processing header (multiple items, Option 2):**
```
Processing {item_count} items from {source_location} ({total_count} {serials|items} → {transfer_count} {transfers|transfer})...
```

**Processing header (transaction-based, Options 3/4/5):**
```
Processing {transaction_number} ({count} {serials|items} → {transfer_count} {transfers|transfer})...
```

**Processing header (multiple transactions):**
```
Processing {transaction_count} transactions...
```
Then for each transaction:
```
{transaction_number} ({count} {serials|items} → {transfer_count} {transfers|transfer})...
```

**Per-transfer result line (success):**
```
✓ Transfer {N} (ID: {internal_id}, Doc: {doc_number}) - {count} {serials|items}
```

**With item detail (multiple items on same transfer):**
```
✓ Transfer {N} (ID: {internal_id}, Doc: {doc_number}) - {count} {serials|items} ({item_name_1}: {count_1}, {item_name_2}: {count_2})
```

**With item name (multiple items across transfers):**
```
✓ Transfer {N} (ID: {internal_id}, Doc: {doc_number}) - {count} {serials|items} ({item_name})
```

**Per-transfer result line (failure):**
```
✗ Transfer {N} FAILED: {error_message}
```

---

## Phase 4: Completion Output

**Single transfer — success:**
```
Complete: {count} {serials|items} in 1 transfer.
→ https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={internal_id}&whence=
```

**Multiple transfers, single source — success:**
```
Complete: {count} {serials|items} in {transfer_count} transfers.

Transfer Records:
• {doc_1} → https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={id_1}&whence=
• {doc_2} → https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={id_2}&whence=
```

**Multiple items processed — success:**
```
Complete: {item_count} items processed, {total_count} {serials|items} in {transfer_count} transfers.

Transfer Records:
• {doc_1} → https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={id_1}&whence=
```

**Multiple items from named source — success:**
```
Complete: {item_count} items processed from {source_location}, {total_count} {serials|items} in {transfer_count} transfers.

Transfer Records:
• {doc_1} → https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={id_1}&whence=
```

**Multiple transactions — success:**
```
Complete: {transaction_count} transactions processed, {total_count} {serials|items} in {transfer_count} transfers.

Transfer Records:
• {doc_1} → https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={id_1}&whence=
```

**Failure — stopped mid-batch (serialized):**
```
Stopped. {completed_count}/{total_count} serials transferred.
✓ {doc_number} → https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={id}&whence=
Resume with: "{resume_command}"
```

**Failure — stopped mid-batch (non-serialized):**
```
Stopped. {completed_count}/{total_count} items transferred.
Resume with: "{resume_command}"
```

**Failure — stopped mid-batch (multiple transactions):**
```
Stopped. {completed_count}/{total_count} items transferred ({completed_txn} complete, {failed_txn} failed).
✓ {doc_number} → https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={id}&whence=
Resume with: "{resume_command}"
```

---

## Error Messages (Use Exact Text)

**Serial not found:**
```
Serial '{serial_number}' not found or has zero quantity on hand.
```

**Partial serials found:**
```
Found {found_count} of {total_count} serials. Serial '{missing_serial}' not found or has zero quantity on hand.
Proceed with the {found_count} found serials?
```

**Item not found:**
```
Item '{item_sku}' not found (or is inactive).
```

**Item not found at source location (non-serialized):**
```
Item '{item_sku}' not found at {source_location} or has zero quantity on hand.
```

**Item not found at source location (serialized):**
```
Item '{item_sku}' not found at {source_location} or has zero serialized quantity on hand.
```

**Insufficient quantity (non-serialized):**
```
Only {available} units available at {source_location}; {requested} requested. Proceed with {available}?
```

**Insufficient quantity (serialized):**
```
Only {available} serials available at {source_location}; {requested} requested. Proceed with {available}?
```

**Transaction not found:**
```
Transaction '{transaction_number}' not found.
```

**No inventory items on transaction:**
```
No inventory items found on {transaction_number}.
```

**Location not found:**
```
Location '{search_term}' not found. Active locations: [{location_list}].
```

**No transactions found for date:**
```
No transactions found for {date} with inventory items.
```

**No transactions found for entity:**
```
No transactions found from {entity_name} in that date range with inventory items.
```

**Entity not found:**
```
Vendor/Customer '{entity_name}' not found. Did you mean: [{suggestion_list}]?
```

**Multiple entity matches:**
```
Multiple entities match '{search_term}': {entity_list}. Please specify which one.
```

**Missing required inputs:**
```
To process this request, I need: {missing_input_list}.
```

**Serialized item — user did not provide specific serial numbers:**
```
{item_name} is a serialized item — I need to know which specific serials to transfer.

There are {on_hand_count} serials of {item_name} on hand at {source_location}. I can pull up a list so you can pick from it. How many would you like to see?

You can also type specific serial numbers or upload a file.
```

**User asks Claude to "just pick" or "choose random" serials:**
```
I can't auto-select serials, but I can make this easy — here are {requested_count} serials of {item_name} currently on hand at {source_location}:

{serial_list}

Would you like me to transfer these {requested_count}? Or pick different ones from the list?
```

> **⚠ RULE: Claude must NEVER auto-select AND auto-submit serials in one step.** The user must always see the specific serial numbers and explicitly confirm before Claude creates any transfer. Claude CAN query and display available serials to help the user choose — but the user must approve the shown list. This is not the same as auto-selecting: showing → confirming → transferring is safe. Silently picking → transferring without showing is not.
>
> **Workflow when user asks Claude to pick:**
> 1. Query INV-B2 (or INV-B1s) to get available serials at the source location
> 2. Display the requested count of serials to the user
> 3. Ask: "Would you like me to transfer these? Or pick different ones?"
> 4. WAIT for explicit confirmation
> 5. Only after "yes" → proceed with those serials as Option 1
>
> **Options 3–5 are exempt** — the transaction defines which serials are involved, so Claude fetches and uses them directly.

#### Revalidation Discrepancy Messages (Options 3, 4, 5 Only)

**Item on-hand lower than transaction count (serialized):**
```
⚠ {item_name}: transaction shows {transaction_count} serials at {source_location}, but only {on_hand_count} currently on hand. Using {on_hand_count}.
```

**Item on-hand lower than transaction count (non-serialized):**
```
⚠ {item_name}: transaction shows {transaction_qty} units at {source_location}, but only {on_hand_qty} currently on hand. Using {on_hand_qty}.
```

**Item no longer on hand at source location:**
```
⚠ {item_name}: no longer on hand at {source_location}. Skipping.
```

**All items from transaction no longer on hand:**
```
{transaction_number}: no inventory items currently on hand at the source {locations|location}. Nothing to transfer.
```

---

## Terminology Rules

| Data Scenario | Use This Word |
|---------------|---------------|
| All items are serialized | "serials" |
| All items are non-serialized | "items" |
| Mix of serialized and non-serialized | "items" |
| Singular (1 serial) | "serial" |
| Singular (1 item) | "item" |
| Singular (1 transfer) | "transfer" |
| Singular (1 location) | "location" |
| Singular (1 transaction) | "transaction" |
| Plural (2+) | add "s" |

## Output Rules

1. **No conversational filler.** Do not add "Let me look that up," "Here's what I found," "Great, I'll process that now," or similar.
2. **No internal narration.** Do not describe buffer planning, line counting, query strategy, or processing decisions.
3. **No query results.** Do not echo SuiteQL output, row counts, or raw data.
4. **No payload display.** Do not show JSON payloads.
5. **No serial listing.** Do not list individual serial numbers.
6. **Phases flow directly.** Discovery → Confirmation → (user approves) → Execution → Completion. No extra text between phases.
7. **Two confirmation formats.** Use Format A (bullet-point with listed serials) for Option 1 with few serials. Use Format B (full table) for everything else.
8. **Numbers over 999 use commas.** Example: `1,500` not `1500`.
