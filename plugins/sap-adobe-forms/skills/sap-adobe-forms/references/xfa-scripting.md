# XFA Scripting Reference

## Script Languages

| Language | Best For |
|---|---|
| JavaScript | Complex logic, loops, DOM traversal |
| FormCalc | Page functions, math, simpler expressions |

Set per-field in Script Editor → Language dropdown.

## Field Events

| Event | Fires When |
|---|---|
| `Initialize` | Field/subform first loads |
| `Calculate` | Any data change that may affect value |
| `Validate` | Field loses focus or form submits |
| `Change` | User edits field value |
| `Click` | Button clicked |
| `Enter` / `Exit` | Focus gained / lost |

## Initialize Event (Conditional Visibility)

```javascript
// Subform Initialize — show/hide based on header flag
if (IS_HEADER.HAZMAT_FLAG.rawValue == "X") {
    this.presence = "visible";
} else {
    this.presence = "hidden";
}
```

```javascript
// Multi-condition visibility
var copyNo = xfa.form.form1.IS_HEADER.COPY_NO.rawValue;
var docType = xfa.form.form1.IS_HEADER.DOC_TYPE.rawValue;

if (copyNo > 1 && docType == "COPY") {
    this.presence = "visible";
} else {
    this.presence = "hidden";
}
```

## Calculate Event (Running Totals)

```javascript
// Sum all quantity fields in IT_ITEMS repeating subform
var total = 0;
var items = xfa.resolveNodes("$record.IT_ITEMS[*].QUANTITY");
for (var i = 0; i < items.length; i++) {
    var val = parseFloat(items.item(i).rawValue) || 0;
    total += val;
}
this.rawValue = total;
```

```javascript
// Count non-empty rows
var count = 0;
var items = xfa.resolveNodes("$record.IT_ITEMS[*].ITEM_NO");
for (var i = 0; i < items.length; i++) {
    if (items.item(i).rawValue != null && items.item(i).rawValue != "") {
        count++;
    }
}
this.rawValue = count;
```

## FormCalc — Page Functions

```formcalc
" Page X of Y (in footer text field — Calculate event):
"Page " & Page() & " of " & Count(PageAll())

" Total line items (in footer):
Count(IT_ITEMS[*].ITEM_NO)

" Sum quantities:
Sum(IT_ITEMS[*].QUANTITY)

" Conditional value:
If (IS_HEADER.COPY_NO > 1, "COPY", "ORIGINAL")
```

## Navigating the XFA DOM

```javascript
// Absolute path from form root
xfa.form.form1.IS_HEADER.SHIP_TO_NAME.rawValue

// Relative path from current field (use $ for self)
this.rawValue

// Parent subform
this.parent.rawValue

// Sibling field
this.parent.ANOTHER_FIELD.rawValue

// Resolve multiple nodes (returns XFANodeList)
var nodes = xfa.resolveNodes("$record.IT_ITEMS[*].HU_ID");
nodes.length   // count
nodes.item(i)  // access by index
```

## Presence Values

```javascript
this.presence = "visible";    // shown, takes layout space
this.presence = "hidden";     // not shown, still takes space
this.presence = "invisible";  // not shown, no layout space
this.presence = "inactive";   // not shown, excluded from data
```

## Dynamic Field Values

```javascript
// Set field value from another field
this.rawValue = xfa.form.form1.IS_HEADER.DELIVERY_NO.rawValue;

// Concatenate multiple fields
this.rawValue = xfa.form.form1.IS_HEADER.CITY.rawValue
              + ", "
              + xfa.form.form1.IS_HEADER.COUNTRY.rawValue;

// Format a date value (stored as YYYYMMDD from ABAP)
var raw = xfa.form.form1.IS_HEADER.SHIP_DATE.rawValue || "";
if (raw.length == 8) {
    this.rawValue = raw.substr(6,2) + "." + raw.substr(4,2) + "." + raw.substr(0,4);
}
```

## Page Break Control

```javascript
// Force page break before this subform — set in Pagination tab, not script
// Subform → Object → Pagination → Before: Start new page

// Trigger relayout after conditional changes:
xfa.layout.relayout();
```

## Validate Event

```javascript
// Validate that a required field has a value
if (this.rawValue == null || this.rawValue == "") {
    app.alert("Handling Unit ID is required.");
    return false;  // blocks form submission
}
return true;
```

## Repeating Subform Index

```javascript
// Get the current row index in a repeating subform
var idx = this.parent.instanceIndex;

// Access the same index in another table
var matchNode = xfa.resolveNode("$record.IT_TOTALS[" + idx + "].SUBTOTAL");
if (matchNode) {
    this.rawValue = matchNode.rawValue;
}
```

## Barcode Value Construction

```javascript
// Build barcode value from multiple fields in Initialize event
var dlvNo  = xfa.form.form1.IS_HEADER.DELIVERY_NO.rawValue || "";
var itemNo = this.parent.ITEM_NO.rawValue || "";
var huId   = this.parent.HU_ID.rawValue || "";

this.rawValue = dlvNo + "^" + itemNo + "^" + huId;
```

## Debugging

```javascript
// Write to console (visible in Adobe LiveCycle preview):
xfa.host.messageBox("Value is: " + this.rawValue);

// Check if node resolved
var node = xfa.resolveNode("$record.IS_HEADER.DELIVERY_NO");
if (!node) {
    xfa.host.messageBox("Node not found — check data binding");
}
```
