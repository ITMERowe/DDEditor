# DSON Format

> **Note**
>
> This document explains how **this project's browser parser** reads, interprets, and reconstructs Darkest Dungeon save files.
>
> The underlying DSON parsing approach is adapted from **robojumper's DarkestDungeonSaveEditor** reverse engineering work. Code snippets shown in this document are from this project's JavaScript browser implementation unless explicitly marked otherwise. Any logic or snippet that directly reflects robojumper's original parsing work should be understood as adapted from robojumper and credited accordingly.

---

# Overview

The parser converts a binary DSON save into editable JSON by processing the file in several stages.

1. Verify the file signature.
2. Read the header.
3. Parse object metadata.
4. Parse field metadata.
5. Read the raw data block.
6. Decode primitive values.
7. Rebuild the object hierarchy.
8. Export JSON.

Each stage produces information required by the next, allowing the parser to reconstruct the save without repeatedly scanning the file.

---

# File Layout

```
┌────────────────────────────────────────────┐
│ Header                                     │
├────────────────────────────────────────────┤
│ Object Metadata (Meta1)                    │
├────────────────────────────────────────────┤
│ Field Metadata (Meta2)                     │
├────────────────────────────────────────────┤
│ Raw Data Block                             │
└────────────────────────────────────────────┘
```

The metadata blocks describe the structure of the save, while the Data block contains field names and values.

---

# Step 1 — Verify the File

The parser first checks that the uploaded file begins with the expected DSON signature.

> This validation logic is part of the DSON parsing behavior adapted from robojumper's reverse engineering work, shown here as implemented in this project's JavaScript parser.

```js
const MAGIC = [0x01, 0xB1, 0x00, 0x00];

const magic = Array.from(bytes.slice(0, 4));

if (magic.join(",") !== MAGIC.join(",")) {
    throw new Error("Invalid DSON file");
}
```

If the signature does not match, parsing stops immediately.

---

# Step 2 — Read the Header

The header provides the offsets required to locate the remaining sections.

> The header layout and interpretation are based on robojumper's DSON reverse engineering. The snippet below is this project's JavaScript implementation of that parsing step.

The implementation reads the header sequentially.

```js
const magicNum = reader.readInt32();
const revision = reader.readInt32();
const headerLength = reader.readInt32();

const meta1Size = reader.readInt32();
const numMeta1 = reader.readInt32();
const meta1Offset = reader.readInt32();

const numMeta2 = reader.readInt32();
const meta2Offset = reader.readInt32();

const dataLength = reader.readInt32();
const dataOffset = reader.readInt32();
```

Instead of searching the file, these offsets allow the parser to jump directly to each section.

---

# Step 3 — Parse Object Metadata

Each Meta1 entry represents one object in the save.

> The meaning of the Meta1 fields comes from robojumper's DSON reverse engineering. The code shown below is this project's JavaScript implementation.

```js
meta1.push({
    parentIndex: reader.readInt32(),
    meta2EntryIdx: reader.readInt32(),
    numDirectChildren: reader.readInt32(),
    numAllChildren: reader.readInt32()
});
```

No values are stored here.

Instead, this table describes how objects relate to one another so the hierarchy can be reconstructed later.

---

# Step 4 — Parse Field Metadata

Each Meta2 entry describes one field.

> The packed `fieldInfo` structure is based on robojumper's DSON reverse engineering. The bit extraction shown below is this project's JavaScript implementation of that known layout.

```js
const nameHash = reader.readInt32();
const offset = reader.readInt32();
const fieldInfo = reader.readInt32();

const isObject = (fieldInfo & 1) === 1;
const nameLen = (fieldInfo & 0b11111111100) >> 2;
const meta1Idx = (fieldInfo & 0b1111111111111111111100000000000) >> 11;
```

After this stage the parser knows:

- where each field begins
- whether it is an object
- how long its name is
- which Meta1 entry it belongs to

The actual field values have not been read yet.

---

# Step 5 — Read the Data Block

The raw data block contains field names followed immediately by their binary values.

```js
const dataBlock = bytes.slice(
    dataOffset,
    dataOffset + dataLength
);
```

Field sizes are not explicitly stored.

The parser determines each field's boundary by finding the next largest metadata offset.

---

# Step 6 — Read Field Names

Each field begins with its UTF-8 name.

```js
const name = readString(
    dataBlock,
    off,
    m.nameLen - 1
);

off += m.nameLen;
```

Everything after the name belongs to the field value.

---

# Step 7 — Decode Values

The parser does not know primitive types ahead of time.

Instead, `parseFieldValue()` progressively attempts to identify the stored value.

The implementation roughly follows this order:

```
Empty field
↓

null

↓

Boolean / Byte

↓

String

↓

Embedded DSON

↓

Integer

↓

Float

↓

Unknown binary
```

If no interpretation is considered reliable, the parser preserves the original bytes.

```js
return `<data:${fieldData.length}b>`;
```

This avoids silently corrupting undocumented structures.

---

# Alignment

Many primitive values begin on a 4-byte boundary.

Rather than assuming the value starts immediately after the field name, the parser calculates the required padding.

```js
function getAlignmentPadding(offset) {
    return (4 - (offset % 4)) % 4;
}

const padding = getAlignmentPadding(dataOffsetInFile);
```

Without respecting alignment, integers and floating-point values would be read incorrectly.

---

# Embedded DSON Files

Some string fields actually contain another complete DSON document.

Before returning a string, the parser checks for another DSON magic number.

```js
const result = decodeDSON(embeddedBuffer);
return result.data;
```

This allows nested save files to appear naturally as nested JSON objects.

---

# Step 8 — Sort Fields

After all fields have been collected, the parser sorts them by their position in the data block.

```js
fields.sort((a, b) => a.dataOffset - b.dataOffset);
```

Sorting ensures the reconstruction phase processes fields in the same order they originally appeared.

---

# Step 9 — Rebuild the Object Tree

Initially every field exists in a flat array.

The parser rebuilds the hierarchy using a stack.

```js
if (field.isObject) {
    field.children = [];
    fieldStack.push(field);
}

...

while (
    fieldStack.length > 0 &&
    fieldStack[fieldStack.length - 1].children.length >=
    fieldStack[fieldStack.length - 1].expectedChildren
) {
    fieldStack.pop();
}
```

Objects are automatically closed once they have received the expected number of children recorded in Meta1.

This allows the hierarchy to be reconstructed without explicit closing markers.

---

# Step 10 — Convert to JSON

Finally, the reconstructed tree is recursively converted into plain JavaScript objects.

```js
function toJsonField(field) {
    if (field.isObject) {
        const obj = {};

        for (const child of field.children) {
            obj[child.name] = toJsonField(child);
        }

        return obj;
    }

    return field.value ?? null;
}
```

The result is standard JSON suitable for viewing, editing, and exporting.

---

# Design Decisions

This implementation intentionally prioritizes correctness.

- Parse each section only once.
- Use header offsets instead of scanning.
- Preserve unknown binary values.
- Respect alignment before decoding primitives.
- Detect embedded DSON files recursively.
- Reconstruct objects using metadata instead of guessing.
- Only convert to JSON after the hierarchy has been rebuilt.

---

# Credits

The understanding of the DSON format and the core parsing approach are based on the reverse engineering work by **robojumper** in the original **DarkestDungeonSaveEditor** project.

The browser interface, drag-and-drop workflow, JSON viewer, and client-side application wrapper were developed for this project.

Where this document shows DSON parsing behavior such as the magic number, header layout, Meta1 structure, Meta2 structure, packed `fieldInfo` decoding, alignment handling, embedded DSON recognition, or object reconstruction, that behavior should be credited to robojumper's reverse engineering work. The snippets themselves are written in this project's JavaScript implementation style unless otherwise stated.
