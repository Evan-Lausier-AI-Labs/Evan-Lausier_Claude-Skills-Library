---
name: netsuite-inventory-transfer
description: "Use this skill whenever the user wants to transfer inventory in NetSuite — moving serialized or non-serialized items between warehouse locations by creating Inventory Transfer records. Triggers include: any mention of 'transfer', 'move inventory', 'move items', 'move serials', serial numbers, item SKUs with source/destination locations, vendor bill transfers, transaction-based transfers, or references to NetSuite inventory operations. Also triggers when the user uploads a spreadsheet (.xlsx, .csv, .tsv) containing serial numbers or item data for transfer, references specific transaction numbers (BILL-*, PO-*, IT-*), asks to find items by date or vendor for transfer, or says 'resume' a previous transfer batch. Use this skill even if the user doesn't say 'inventory transfer' explicitly — any request to relocate stock between NetSuite locations qualifies. Do NOT use for general NetSuite queries, reporting, or non-inventory record operations."
---

# NetSuite Inventory Transfer Skill

## Overview

This skill creates Inventory Transfer records in NetSuite to move serialized and non-serialized items between warehouse locations. It uses the NetSuite MCP integration (SuiteQL queries via `ns_runCustomSuiteQL` and record creation via `ns_createRecord`).

**Before doing anything else**, read the appropriate reference file(s) from `references/` based on what the user is asking for. The reference files contain the exact SuiteQL queries, payload structures, output templates, and processing rules that must be followed precisely.

## Reference Files — Read These First

| File | What It Contains | When to Read |
|------|-----------------|--------------|
| `references/queries.md` | All SuiteQL queries (INV-A1, INV-B0, INV-B1, INV-B1a, INV-B1s, INV-B2, INV-B3, T-A, T-B, T-C, T1, T2, T3, T4, T5, T6, Query 7, Query 8), cursor pagination patterns, item resolution | **Always** — read before running any query |
| `references/transfer-creation.md` | Transfer record JSON structure, serialNumbers delimiter rules, field reference, payload sizing, document numbering, pre-submission validation, line-building logic | **Always** — read before creating any transfer record |
| `references/output-templates.md` | Mandatory user-facing output text for all 4 phases (Discovery, Confirmation, Execution, Completion), error messages, terminology rules | **Always** — read before producing any user-facing output |
| `references/processing-rules.md` | Transfer buffer workflow, stream processing for >500 serials, detailed processing algorithm, line-count flushing, context management rules, resume capability, NEVER/ALWAYS rules | **Always** — read before processing any transfer batch |

**Read all four reference files before beginning work on any transfer request.** They define the exact queries, payload formats, output text, and processing logic. Do not improvise or paraphrase — follow them precisely.

## Five Request Types

| # | Request Type | User Provides | Primary Query Path |
|---|-------------|---------------|-------------------|
| 1 | By Serial Number | Serial number(s), Destination, Date | INV-A1 → group by location/item → transfer buffer |
| 2 | By Item + Source Location | Item SKU(s), Source, Qty, Destination, Date | INV-B0 → INV-B1 or INV-B1s → INV-B2/B3 → transfer buffer |
| 3 | Specific Transactions | Transaction number(s), Destination, Date | T1 → T3 → **revalidate (INV-B1/INV-B1s)** → **INV-B2/B3** (serialized) → transfer buffer |
| 4 | Transactions by Date | Date range, Destination, Transfer Date | T-A → T3 → **revalidate (INV-B1/INV-B1s)** → **INV-B2/B3** (serialized) → transfer buffer |
| 5 | Transactions by Entity+Date | Entity name, Date range, Destination, Transfer Date | T-B → T3 → **revalidate (INV-B1/INV-B1s)** → **INV-B2/B3** (serialized) → transfer buffer |

**Default lookup path:** Always query the master inventory record first (Options 1 & 2). Only use transaction-based queries (Options 3–5) when the user explicitly references a transaction number, date range for transactions, or vendor/customer name.

### Mandatory: Inventory Revalidation for Transaction-Sourced Items (Options 3, 4, 5)

> **Hard Rule:** When a request starts from a transaction, date range, or vendor/customer search, use the transaction only to identify candidate items. Before creating any inventory transfer, revalidate each item against current on-hand inventory at the source location.

**Why:** A bill, PO, or receipt records where inventory originally came from, but does not guarantee inventory is still there now. Items may have been transferred, consumed, or adjusted since the transaction was created.

**How revalidation works:**
1. Use transaction queries (T-A, T-B, T1, T3) to identify candidate items and source locations.
2. Before presenting the confirmation summary, verify current on-hand for each candidate:
   - Serialized (any item type) → Run INV-B1s; use MIN(T3 count, on-hand count)
   - Non-serialized, InvtPart → Run INV-B1; use MIN(T3 qty, on-hand qty)
   - Non-serialized, Assembly → Run INV-B1a (`ns_getRecord` assemblyitem); verify totalQuantityOnHand ≥ T3 qty (per-location not available for non-serialized Assembly — total on-hand is a safety guard)
3. If on-hand is lower than transaction count, include a discrepancy note in the confirmation.
4. If an item has zero on-hand, exclude it and report as unavailable.
5. **Fetch serials after revalidation (serialized items only):** Use INV-B2 (≤500) or INV-B3 (>500) to fetch serial strings currently on-hand at the source location. Cap fetch at MIN(T3 count, revalidated on-hand count). Do NOT use T4/T5/T6 — transaction serials may have moved since the bill/PO was created.
6. Revalidation is silent when counts match — no extra output needed.

## Required Inputs by Request Type

**Option 1 — By Serial Number:**
- Serial number(s) ✓ (comma-separated list, or uploaded file with Serial column), Destination ✓, Transfer Date ✓, Memo (optional)

**Option 2 — By Item + Source Location:**
- Item SKU(s) ✓, Source Location ✓, Quantity ✓ ("all" or numeric), Destination ✓, Transfer Date ✓, Memo (optional)
- **⚠ Serialized items:** If INV-B0 shows `isserialitem = 'T'`, Claude must NOT auto-submit serials without showing them first. Offer to pull up available serials so the user can confirm. The user must see and approve the specific serials before any transfer is created.

**Option 3 — Specific Transactions:**
- Transaction number(s) ✓, Destination ✓, Transfer Date ✓, Item Filter (optional), Quantity per Item (optional), Memo (optional)

**Option 4 — Transactions by Date:**
- Date range/cutoff ✓, Destination ✓, Transfer Date ✓, Transaction Type (optional), Item Filter (optional), Memo (optional)

**Option 5 — Transactions by Entity+Date:**
- Entity name ✓, Date range ✓, Destination ✓, Transfer Date ✓, Transaction Type (optional), Memo (optional)

If inputs are missing, respond: `"To process this request, I need: [list missing items]"`

If the user did NOT include a memo in their request, ask immediately (before any discovery queries) if they'd like to add one. If they say yes without providing memo text, ask what the memo should say — do NOT assume the default. If they provide memo text (in the same message or a follow-up), use it. If they decline, use the auto-generated default. This must be the FIRST response after receiving the transfer request.

## System Limits

These limits exist because exceeding them causes API timeouts, transaction failures, or processing errors. The values represent tested safe maximums.

| Parameter | Limit | Applies To |
|-----------|-------|-----------|
| Serials per line | 100 max | Serialized items only |
| Lines per transfer | **5 max (hard limit)** | All items |
| Serials per transfer | 500 max (250–500 recommended) | Serialized items only |
| Non-serialized qty per line | No limit (full qty on 1 line) | Non-serialized items |
| Non-serialized qty per transfer | No limit | Non-serialized items |
| Query batch size | 500 rows | All queries |

**Transfer count formulas:**
- Serialized: `CEILING(serial_count / 500)` — but may be higher if line-count limit (5) triggers first
- Non-serialized: `CEILING(distinct_item_count / 5)` — 1 line per item, full quantity
- **Mixed (serialized + non-serialized):** Calculate total lines needed across ALL items: `SUM(CEILING(serial_count / 100) for each serialized item) + COUNT(non-serialized items)`. Flush at 5 lines. Do NOT split serialized and non-serialized into separate transfers — they go on the same transfer when total lines ≤ 5 and they share the same source location.

## High-Level Processing Flow

```
1. Parse user request → identify request type (1–5)
2. IF user did not provide a memo → ask immediately: "Would you like to add a memo?"
   Wait for response before proceeding.
3. Validate inputs → resolve locations (Query 7 if text names provided)
4. Run discovery queries → find items/serials/transactions
   ⚠ For Option 1 with many serials: verify ALL serials via INV-A1 (batched).
   Do NOT skip batches or assume unqueried serials match.
5. Present confirmation summary → WAIT for user approval
6. Execute transfers using buffer workflow (see references/processing-rules.md)
7. Report results with NetSuite links
```

**The memo prompt and confirmation step are both mandatory.** Never skip the memo question. Never create transfer records without explicit user approval.

## Input Parsing Rules

| Input | How to Parse |
|-------|-------------|
| Serial Numbers | Split by comma, trim whitespace |
| Item SKUs | Split by comma, trim whitespace |
| Transaction Numbers | Split by comma, trim whitespace |
| Location (text) | Run Query 7 to resolve to ID |
| Location (numeric) | Use as ID directly |
| Transfer Date | Convert any format to YYYY-MM-DD |
| Transaction Type | "bill"→`VendBill`, "PO"→`PurchOrd`, "receipt"→`ItemRcpt` |
| Quantity | "all"→ full on-hand qty; numeric → that qty |
| Memo | Auto-generate: `"Claude | Transfer {N} of {total}"`. Append user memo if provided: `"Claude | Transfer {N} of {total} | {user_memo}"` |

## File-Based Input (Uploaded Spreadsheets)

When a user uploads a `.xlsx`, `.csv`, or `.tsv` file, read it and map columns to transfer inputs. See `references/processing-rules.md` for the full file handling logic.

**Two modes:**

1. **Known format (Vendor Bill SN List):** If headers include `SERIAL`, `ITEM/PN`, and `QUANTITY` → auto-map the `SERIAL` column, report what was found, and proceed directly to Option 1. No column confirmation needed. The `ITEM/PN` column is informational only (may not match NetSuite SKU exactly).

2. **Unknown format:** Use fuzzy column matching (case-insensitive). Present detected mapping to user and wait for confirmation before proceeding.

**For both modes:**
- Serial column found → **Option 1** (by serial number)
- Item + Source + Quantity found (no serials) → **Option 2** (by item + source location)
- Transfer date, destination, and memo always come from the user's message, not the file
- Empty cells are skipped; duplicate serials are deduplicated

## Item Resolution (Multiple Matches)

When an item SKU matches multiple items, auto-filter in this order — do NOT ask the user to pick an ID:

1. Active only (`isinactive = 'F'`)
2. Inventory-tracked only (`itemtype IN ('InvtPart', 'Assembly')`)
3. On-hand quantity > 0 at relevant location
4. Exact match over partial match on `itemid`

After filtering: 1 result → use it. 2+ results → present filtered list with distinguishing info. 0 results → report not found.

## Location Resolution

Claude resolves all location names to IDs dynamically using Query 7 at runtime. There is no hardcoded location table — this ensures the solution works across environments (sandbox, production) and automatically picks up new locations without documentation updates.

Users can provide either a location name (e.g., "HQ") or a numeric ID (e.g., "4"). If a name is provided, Claude runs Query 7 to resolve it. If the name is not found, Claude shows the list of active locations.

## Environment Configuration

These values are environment-specific. Update them when switching between sandbox and production.

| Parameter | How to Set |
|-----------|-----------|
| **NetSuite Base URL** | Set in the Project Instructions. Claude reads it from there to construct transfer record links. |
| **Subsidiary** | Determined dynamically from the item record (`i.subsidiary` returned by SuiteQL queries). Do NOT hardcode. |
| **MCP Connector** | Connected in the Claude project settings. Must point to the correct NetSuite environment. |

**Transfer record URL pattern:**
```
https://{netsuite_base_url}/app/accounting/transactions/invtrnfr.nl?id={internal_id}&whence=
```

## MCP Tools Used

| Task | Tool |
|------|------|
| SuiteQL queries (all discovery/lookup) | `ns_runCustomSuiteQL` |
| Create transfer records | `ns_createRecord` (recordType: `inventorytransfer`) |
| Read back document number (1st transfer in batch) | `ns_getRecord` |
| Verify non-serialized Assembly on-hand (INV-B1a) | `ns_getRecord` (recordType: `assemblyitem`) |
| Update existing records | `ns_updateRecord` |

## Critical Rules (Quick Reference)

### Output Discipline (Enforced — No Exceptions)

Claude's output to the user follows ONLY the four-phase pattern: Discovery → Confirmation → Execution → Completion. Nothing else.

- **Do NOT narrate internal planning.** No buffer math ("4 + 4 = 8 lines > 5"), no transfer strategy, no line counting, no revalidation calculations ("transaction shows 400, on-hand = 799 → use 400").
- **Do NOT echo query results.** No SuiteQL output, row counts, or raw data.
- **Do NOT show JSON payloads.** No transfer record structures.
- **Do NOT list serial numbers.** Report counts only.
- **Revalidation is silent when counts match.** Only show discrepancy notes when on-hand differs from transaction count.
- **Use exact text from output templates** (`references/output-templates.md`) for all user-facing output.

### Assembly Item Handling (Critical — `inventoryitemlocations` Does Not Work for Assembly)

When an item has `itemtype = 'Assembly'`:
- **Serialized Assembly** → Use INV-B1s (queries `inventorynumberlocation`) — works correctly
- **Non-serialized Assembly** → Use **INV-B1a** (NOT INV-B1). INV-B1 queries `inventoryitemlocations` which returns zero for Assembly items.
- **INV-B1a method:** Call `ns_getRecord` with `recordType: "assemblyitem"`, read `totalQuantityOnHand`. This gives total on-hand across all locations (per-location is not available for non-serialized Assembly via SuiteQL).
- **During revalidation (Options 3–5):** If `totalQuantityOnHand ≥ T3 qty`, proceed with T3 qty. If less, use `totalQuantityOnHand`. If zero, skip the item.

### NEVER:
- Auto-select AND auto-submit serials in one step — user must always SEE the specific serials and explicitly confirm before transfer. Claude CAN query and display available serials to help the user choose, then wait for confirmation. Showing → confirming → transferring is OK. Silently picking → transferring is not. Options 3–5 are exempt (transaction defines the serials).
- Query more than 500 serials at a time
- Create transfers with >500 serials or >5 lines
- Split non-serialized item qty across multiple lines
- Split serialized and non-serialized items into separate transfers when they share the same source location and total lines ≤ 5 — they go on the same transfer
- Use literal backslash+n text (`"\\n"`) when building `serialNumbers` — causes double-escaping. Use actual newline chars: `"\n".join(serials)` (Python) or `chunk.join("\n")` (JS)
- Use `\\n` double backslash (literal text, not a delimiter) — the MCP tool handles JSON escaping automatically
- Use OFFSET with SELECT DISTINCT
- Mix source locations in one transfer
- Narrate buffer planning, line counting, revalidation math, or transfer strategy to the user
- Echo query results, JSON payloads, or serial number lists
- Show revalidation calculations (only show discrepancy notes when counts differ)
- Create transfers without user confirmation
- Skip memo prompt — always ask as the FIRST step if user didn't provide one
- Skip INV-A1 verification batches — must verify ALL user-provided serials before confirmation, not just a sample
- Reconstruct or generate serial strings from memory when building INV-A1 queries — this causes hallucinated serials and false "not found" errors
- Fetch all serials upfront when total >500 (use stream processing)
- Use `pageSize` parameter on `ns_runCustomSuiteQL` for `SELECT DISTINCT` serial fetch queries
- Use `inventoryitemlocations` for ANY Assembly item — it returns zero
- Trust transaction counts as current on-hand for Options 3–5 (transactions are discovery-only)
- Use T4/T5/T6 to fetch serial strings for transfer creation (Options 3–5) — transaction serials may have moved. Use INV-B2/B3 (current on-hand at source location) instead

### ALWAYS:
- Read all reference files before starting
- Ask for memo as the FIRST response after receiving a transfer request (if user didn't provide one). Wait for response before running any queries.
- Query inventory tables first for Options 1 & 2
- For Option 1 with many serials: write ALL serials to a file on disk FIRST (e.g., `/home/claude/serials.txt`). Run ALL INV-A1 batches by reading from that file. Do NOT assume unqueried serials will match.
- Build INV-A1 batch IN clauses by reading from the saved file using index ranges (e.g., `serials[0:400]`, `serials[400:800]`). Do NOT reconstruct serial strings from memory — hallucinated serials cause false "not found" errors and wasted backtracking.
- For Options 3–5 (transaction/date/vendor): Use the transaction to identify candidate items and counts (T3), but fetch actual serial strings from current on-hand at the source location (INV-B2/B3). The "no auto-select" rule does NOT apply here — the transaction defines which items are involved, and INV-B2/B3 provides the serials currently available.
- Run INV-B0 first for Option 2 (determines serialized vs non-serialized AND InvtPart vs Assembly)
- Check `itemtype` from INV-B0 or T3 — if Assembly + non-serialized, use INV-B1a (not INV-B1)
- Run Query T3 first for transaction-based options (Options 3–5)
- Revalidate transaction-sourced items against current on-hand before creating transfers:
  - Serialized → INV-B1s
  - Non-serialized InvtPart → INV-B1
  - Non-serialized Assembly → INV-B1a (`ns_getRecord` assemblyitem → `totalQuantityOnHand`)
- After revalidation, fetch serial strings for serialized items using INV-B2 (≤500) or INV-B3 (>500) at the source location. Cap at MIN(T3 count, revalidated on-hand count). Never use T4/T5/T6 for this.
- Present confirmation summary and wait for approval before creating transfers
- Use cursor pagination (`WHERE id > {last_id}`) for >500 serials
- Validate each line's serial count matches `adjustQtyBy` before submitting
- Use stream processing (fetch→create→discard) for serialized batches >500
- Count lines before creating transfer; flush buffer at 5 lines
- Combine serialized and non-serialized items on the same transfer when they share the same source location and total lines ≤ 5. The buffer accumulates both types — use the combined `build_lines` function to produce a single `items` array.
- Use sequential doc numbering for multi-transfer batches (first = NS auto, 2+ = base-N)
- Use `inventorynumberlocation` for serialized Assembly items
- Use exact text from output templates (references/output-templates.md)
- Use i.subsidiary from item query results for the subsidiary field — never hardcode
