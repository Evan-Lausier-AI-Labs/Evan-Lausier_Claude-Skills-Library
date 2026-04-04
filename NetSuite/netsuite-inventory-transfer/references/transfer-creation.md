# Transfer Record Creation Reference

## serialNumbers Delimiter — Use Actual Newline Characters

The `serialNumbers` value passed to `ns_createRecord` must contain **actual newline characters** (the byte `0x0A`) between serial numbers. The MCP tool handles JSON serialization internally — it will escape those real newlines to `\n` in the JSON wire format automatically.

### How to Build the String in Code

**Python:**
```python
serial_str = "\n".join(serials)   # "\n" in Python = actual newline byte — CORRECT
# Do NOT use "\\n".join(serials)  # "\\n" in Python = literal backslash+n text — WRONG
```

**JavaScript:**
```javascript
serialNumbers: chunk.join("\n")   // "\n" in JS = actual newline byte — CORRECT
// Do NOT use chunk.join("\\n")   // "\\n" in JS = literal backslash+n text — WRONG
```

### Why This Matters

When the MCP tool (or `json.dumps()` / `JSON.stringify()`) serializes the payload:
- **Actual newline byte** → becomes `\n` in JSON → ✅ NetSuite reads it correctly
- **Literal backslash+n text** (`"\\n"`) → becomes `\\n` in JSON → ❌ NetSuite sees literal text, not a delimiter (double-escape bug)

### What the Final JSON Looks Like (for reference only)

In the JSON that goes over the wire, the serialNumbers value will show `\n` between serials:
```
"serialNumbers": "SN-0001-AAAA\nSN-0002-BBBB\nSN-0003-CCCC"
```

You do NOT need to manually produce this — the JSON serializer does it for you when you pass a string containing actual newline characters.

### Common Mistake: Double-Escaping

If you see `\\n` in the JSON output (or in error logs), the serial string was built with literal backslash+n text instead of actual newline characters. Fix by using `"\n".join(serials)` (Python) or `chunk.join("\n")` (JS) — NOT `"\\n".join(serials)` or `chunk.join("\\n")`.

---

## Transfer Record Structure — Serialized Items

```json
{
  "recordType": "inventorytransfer",
  "data": {
    "trandate": "2026-01-15",
    "memo": "Claude | Transfer 1 of 3",
    "subsidiary": "{i.subsidiary}",
    "location": "10",
    "transferlocation": "11",
    "inventory": {
      "items": [
        {
          "item": "570",
          "adjustQtyBy": 3,
          "serialNumbers": "SN-0001-AAAA\nSN-0002-BBBB\nSN-0003-CCCC"
        }
      ]
    }
  }
}
```

> The `serialNumbers` field takes a newline-delimited string of human-readable serial numbers. Build it with actual newline characters: `"\n".join(serials)` (Python) or `chunk.join("\n")` (JS). The MCP tool handles JSON escaping — you do NOT need to manually escape. Each serial string must match the `inventorynumber.inventorynumber` value from SuiteQL queries, NOT the internal numeric ID. The `adjustQtyBy` value must equal the number of serial numbers provided.

## Transfer Record Structure — Non-Serialized Items

```json
{
  "recordType": "inventorytransfer",
  "data": {
    "trandate": "2026-01-15",
    "memo": "Claude | Transfer 1 of 1",
    "subsidiary": "{i.subsidiary}",
    "location": "10",
    "transferlocation": "11",
    "inventory": {
      "items": [
        {
          "item": "571",
          "adjustQtyBy": 2520
        },
        {
          "item": "572",
          "adjustQtyBy": 840
        }
      ]
    }
  }
}
```

> Non-serialized items use 1 line per item with the full quantity — no splitting across multiple lines. The only limit is 5 lines (i.e., 5 distinct items) per transfer.

## Transfer Record Structure — Mixed (Serialized + Non-Serialized)

Serialized and non-serialized items **can and should** be combined on the same transfer when total lines ≤ 5 and they share the same source location. Do NOT split them into separate transfers unless line limits require it.

```json
{
  "recordType": "inventorytransfer",
  "data": {
    "trandate": "2026-01-15",
    "memo": "Claude | Transfer 1 of 1",
    "subsidiary": "{i.subsidiary}",
    "location": "10",
    "transferlocation": "11",
    "inventory": {
      "items": [
        {
          "item": "571",
          "adjustQtyBy": 250
        },
        {
          "item": "570",
          "adjustQtyBy": 100,
          "serialNumbers": "SN-1001-XXXX\nSN-1002-YYYY\n...100 serials..."
        },
        {
          "item": "570",
          "adjustQtyBy": 50,
          "serialNumbers": "SN-1101-ZZZZ\n...50 serials..."
        }
      ]
    }
  }
}
```

> In this example: 1 non-serialized line (250 units) + 2 serialized lines (150 serials across 2 chunks of 100 and 50) = 3 lines total, well within the 5-line limit. These belong on one transfer.

---

## Field Reference

| Field | Source | Notes |
|-------|--------|-------|
| tranid | Auto-generated (transfers 2+) | Omit for first transfer in batch; for subsequent transfers use `{base_doc_number}-{N}` |
| trandate | User input | YYYY-MM-DD format |
| memo | Auto-generated | `"Claude | Transfer {N} of {total}"` (user memo appended if provided: `"Claude | Transfer {N} of {total} | {user_memo}"`) |
| subsidiary | Query result from `item` table (`i.subsidiary`) | Determined dynamically from item records |
| location | source_location_id from query | FROM location |
| transferlocation | User input (resolved via Query 7 if text) | TO location (destination) |
| inventory.items[].item | item_id from query | Item being transferred |
| inventory.items[].adjustQtyBy | chunk size (1–100 for serialized) or full qty (non-serialized) | Number of serials or qty units in this line |
| inventory.items[].serialNumbers | Newline-delimited serial strings from query | Serialized items only — build with actual newline chars: `"\n".join(serials)`. Do NOT use `"\\n".join()` (causes double-escape). MCP tool handles JSON escaping. |

---

## Pre-Submission Validation (Required before every `ns_createRecord` call)

Before submitting any transfer payload, verify each inventory line:

1. **Serial count check:** Count the actual newline characters in the `serialNumbers` string — the count + 1 must equal `adjustQtyBy`. In Python: `serial_str.count("\n") + 1 == adjustQtyBy`
2. **Duplicate check:** Verify no duplicate serial strings exist within the same transfer (across all lines)
3. **Line count check:** Verify total lines ≤ 5

If any check fails, rebuild the payload from the current query results — do NOT attempt to patch or deduplicate from memory.

---

## Document Numbering for Multi-Transfer Batches

When a batch requires multiple transfers (>500 serials/items or >5 lines), use sequential document numbering to link related transfers.

**Rules:**
1. **First transfer in a batch:** Do NOT pass `tranid` — let NetSuite auto-assign the document number
2. **Read back the document number** from the `ns_createRecord` response (or call `ns_getRecord` with the returned internal ID if the create response does not include `tranid`)
3. **Subsequent transfers in the same batch:** Pass `tranid` with the base number plus a hyphenated suffix: `-2`, `-3`, etc.
4. **Single-transfer batches:** No change — let NetSuite assign normally

**Example sequence for a 1,500-serial batch:**

| Transfer | `tranid` in Payload | How Assigned |
|----------|---------------------|--------------|
| Transfer 1 | *(omitted)* | NetSuite auto-assigns → e.g. `IT-5001` |
| Transfer 2 | `"IT-5001-2"` | Claude appends `-2` to base |
| Transfer 3 | `"IT-5001-3"` | Claude appends `-3` to base |

**Payload example — Transfer 2 of a batch:**
```json
{
  "recordType": "inventorytransfer",
  "data": {
    "tranid": "IT-5001-2",
    "trandate": "2026-01-15",
    "memo": "Claude | Transfer 2 of 3",
    "subsidiary": "{i.subsidiary}",
    "location": "10",
    "transferlocation": "11",
    "inventory": {
      "items": [
        {
          "item": "570",
          "adjustQtyBy": 100,
          "serialNumbers": "SN-1001-XXXX\n...100 serials..."
        }
      ]
    }
  }
}
```

The `ns_getRecord` call to read back the document number is only needed once per batch (after the first transfer).

---

## Building Transfer Lines from Buffer

The buffer may contain a mix of serialized and non-serialized items. Build all lines from the buffer into a single `items` array for one transfer. Do NOT split serialized and non-serialized into separate transfers — they go on the same transfer as long as total lines ≤ 5.

```
FUNCTION build_lines(buffer):
    lines = []
    
    FOR EACH item_id, value IN buffer.items:
        IF value is an array (serialized — contains serial strings):
            // Split into chunks of 100 serials (max per line)
            WHILE value.length > 0:
                chunk = value.splice(0, 100)
                lines.append({
                    item: item_id,
                    adjustQtyBy: chunk.length,
                    // CRITICAL: Join with actual newline character — "\n" in code IS a newline byte
                    // Do NOT use "\\n" — that produces literal backslash+n text (double-escape bug)
                    // The MCP tool / JSON serializer handles escaping to \n in the JSON wire format
                    serialNumbers: chunk.join("\n")
                })
        ELSE (non-serialized — value is a quantity number):
            // Non-serialized: one line per item with full quantity
            lines.append({
                item: item_id,
                adjustQtyBy: value
            })
    
    RETURN lines  // Maximum 5 lines total per transfer
```

> **Key:** The buffer stores serialized items as `{item_id: [serial_strings...]}` and non-serialized items as `{item_id: qty_number}`. The `build_lines` function handles both in a single pass and produces a single `items` array for the transfer payload.

---

## Payload Size Estimate

| Component | Size |
|-----------|------|
| 1 serial in serialNumbers string | ~20 bytes |
| 100 serials (1 line) | ~2.5 KB |
| 500 serials (5 lines) | ~13 KB |
| Headers + structure | ~2 KB |
| **Total per transfer** | **~15 KB** |

---

## Troubleshooting

### "Invalid JSON format for parameter 'data': Unexpected end of JSON input"
- **Cause:** The `serialNumbers` string was built incorrectly — either with literal backslash+n text (`"\\n"`) that got double-escaped, or with some other malformed content.
- **Fix:** Build the serial string using actual newline characters: `"\n".join(serials)` (Python) or `chunk.join("\n")` (JS). Do NOT use `"\\n".join(serials)` or `chunk.join("\\n")` — those produce literal backslash+n text that double-escapes in JSON.
- **Verification:** In Python, check `repr(serial_str)` — you should see `\n` (single backslash), NOT `\\n` (double backslash).

### "Field 'receiptinventorynumber' was not found"
- **Fix:** Use `ia.inventorynumber` NOT `ia.receiptinventorynumber`

### "The number of serial numbers entered (N) is not equal to the item quantity (M)"
- **Cause:** A duplicate serial string was included in the same inventory line, making the actual serial count exceed the declared `adjustQtyBy`. This typically happens when all serials are fetched upfront and a serial is accidentally placed in two different line chunks.
- **Fix:** Use stream processing (fetch→create→discard per transfer). Always run pre-submission validation.
- **If error occurs mid-batch:** Re-fetch the current batch of serials from the cursor position and rebuild the payload from scratch.

### OFFSET + DISTINCT returns duplicate data
- **Problem:** SuiteQL applies OFFSET before DISTINCT, causing silent failures
- **Fix:** Use cursor-based pagination: `WHERE inv.id > {last_id}`

### INV-B1 returns no results for a known item
- **Check:** Is the item an Assembly type? Assembly items do not appear in `inventoryitemlocations`. Use INV-B1s instead.
- **Check:** Confirm the source location ID is correct (run Query 7)
- **Check:** Confirm `quantityonhand > 0`

### Diagnostic Queries (When Results Are Unexpected)

**Check 1 — Is the item inventory-tracked?**
```sql
SELECT id, itemid, isserialitem, itemtype, isinactive
FROM item
WHERE itemid = '{item_sku}'
```

**Check 2 — Does the item have on-hand quantity at the specified location?**
```sql
SELECT il.quantityonhand, il.quantityavailable
FROM inventoryitemlocations il
WHERE il.item = {item_id}
AND il.location = {location_id}
```

**Check 3 — Do inventory assignments exist on the transaction?**
```sql
SELECT COUNT(*) FROM inventoryassignment WHERE transaction = {transaction_id}
```

**Check 4 — Is the transaction ID correct?**
```sql
SELECT id, tranid, type FROM transaction WHERE tranid = '{transaction_number}'
```

### Transfer Fails: Invalid Inventory Number

**Cause:** Serial not at source location or already transferred

**Verify current location:**
```sql
SELECT 
    nib.location,
    l.name,
    nib.quantityonhand
FROM inventorynumberlocation nib
INNER JOIN location l ON l.id = nib.location
WHERE nib.inventorynumber = {inventory_number_id}
AND nib.quantityonhand > 0
```

### Transfer Fails: Payload Error — Prevention

- The 500-serial-per-transfer and 5-line-per-transfer limits are designed to prevent size issues. If errors still occur, reduce to 250 serials per transfer (lower end of safe range).
- Check for special characters in serial numbers that may inflate payload size.
- **Most common cause:** Double-escaped newlines in `serialNumbers` — using `"\\n".join()` instead of `"\n".join()`. The serial string must contain actual newline bytes, not literal backslash+n text. The MCP tool handles JSON escaping automatically.
- **Second most common cause:** Exceeding 5 lines per transfer when mixing serialized and non-serialized items. Always count planned lines before building the payload and flush the buffer at 5 lines.
