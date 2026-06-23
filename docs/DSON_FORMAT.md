# DSON Format

> **Note**
>
> This document explains how **this project's browser parser** reads, interprets, and reconstructs Darkest Dungeon save files.
>
> The underlying DSON parsing approach is adapted from **[robojumper's DarkestDungeonSaveEditor](https://github.com/robojumper/DarkestDungeonSaveEditor)** reverse engineering work. Code snippets shown in this document are from this project's JavaScript browser implementation unless explicitly marked otherwise. Any logic or snippet that directly reflects **[robojumper's DarkestDungeonSaveEditor](https://github.com/robojumper/DarkestDungeonSaveEditor)** original parsing work should be understood as adapted from **[robojumper](https://github.com/robojumper/DarkestDungeonSaveEditor)** and credited accordingly.

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

The metadata blocks describe the structure of the save, while the data block contains field names followed by the corresponding field values.

---

# Step 1 — Verify the File

The parser first checks that the uploaded file begins with the expected DSON signature.

> This validation logic is part of the DSON parsing behavior adapted from **[robojumper's DarkestDungeonSaveEditor](https://github.com/robojumper/DarkestDungeonSaveEditor)** reverse engineering work, shown here as implemented in this project's JavaScript parser.

```js
const MAGIC = [0x01, 0xB1, 0x00, 0x00];
const magic = Array.from(bytes.slice(0, 4));

if (magic.join(',') !== MAGIC.join(',')) {
    throw new Error(
        `Invalid magic number: ${magic
            .map(x => x.toString(16).padStart(2, '0'))
            .join(' ')}`
    );
}
```

If the signature does not match, parsing stops immediately.

---

# Step 2 — Read the Header

The header provides the offsets required to locate the remaining sections.

> The header layout and interpretation are based on **[robojumper's DarkestDungeonSaveEditor](https://github.com/robojumper/DarkestDungeonSaveEditor)** DSON reverse engineering. The snippet below is this project's JavaScript implementation of that parsing step.

The browser parser reads the full 64-byte header sequentially. Some reserved or unused fields are read only to advance the reader offset correctly.

```js
reader.seek(0);

const magicNum = reader.readInt32();
const revision = reader.readInt32();
const headerLength = reader.readInt32();
const _zeroes1 = reader.readInt32();
const meta1Size = reader.readInt32();
const numMeta1 = reader.readInt32();
const meta1Offset = reader.readInt32();
const _zeroes2a = reader.readInt32();
const _zeroes2b = reader.readInt32();
const _zeroes2c = reader.readInt32();
const _zeroes2d = reader.readInt32();
const numMeta2 = reader.readInt32();
const meta2Offset = reader.readInt32();
const _zeroes4 = reader.readInt32();
const dataLength = reader.readInt32();
const dataOffset = reader.readInt32();

if (headerLength !== 64) {
    throw new Error(`Invalid header length: ${headerLength}`);
}

const buildNumber = (revision >> 16) & 0xFFFF;
```

Instead of searching the file, these offsets allow the parser to jump directly to each section.

---

# Step 3 — Parse Object Metadata

Each Meta1 entry represents one object in the save.

> The meaning of the Meta1 fields comes from **[robojumper's DarkestDungeonSaveEditor](https://github.com/robojumper/DarkestDungeonSaveEditor)** DSON reverse engineering. The code shown below is this project's JavaScript implementation.

```js
const meta1 = [];
reader.seek(meta1Offset);

for (let i = 0; i < numMeta1; i++) {
    meta1.push({
        parentIndex: reader.readInt32(),
        meta2EntryIdx: reader.readInt32(),
        numDirectChildren: reader.readInt32(),
        numAllChildren: reader.readInt32()
    });
}
```

No primitive field values are stored here. This table describes object relationships so the hierarchy can be reconstructed later.

---

# Step 4 — Parse Field Metadata

Each Meta2 entry describes one field.

> The packed `fieldInfo` structure is based on **[robojumper's DarkestDungeonSaveEditor](https://github.com/robojumper/DarkestDungeonSaveEditor)** DSON reverse engineering. The bit extraction shown below is this project's JavaScript implementation of that known layout.

```js
const meta2 = [];
reader.seek(meta2Offset);

for (let i = 0; i < numMeta2; i++) {
    const nameHash = reader.readInt32();
    const offset = reader.readInt32();
    const fieldInfo = reader.readInt32();
    const isObject = (fieldInfo & 1) === 1;
    const nameLen = (fieldInfo & 0b11111111100) >> 2;
    const meta1Idx =
        (fieldInfo & 0b1111111111111111111100000000000) >> 11;

    meta2.push({
        nameHash,
        offset,
        fieldInfo,
        isObject,
        nameLen,
        meta1Idx
    });
}
```

After this stage the parser knows:

- where each field begins
- whether it is an object
- how long its name is
- which Meta1 entry it belongs to

The actual field values have not been interpreted yet.

---

# Step 5 — Read the Data Block

The raw data block contains field names followed immediately by their binary values.

```js
const dataBlock = bytes.slice(dataOffset, dataOffset + dataLength);
```

Field sizes are not explicitly stored. The parser determines each field boundary by finding the next largest metadata offset.

```js
let nextOffset = dataBlock.length;

for (let j = 0; j < meta2.length; j++) {
    if (meta2[j].offset > m.offset && meta2[j].offset < nextOffset) {
        nextOffset = meta2[j].offset;
    }
}
```

---

# Step 6 — Read Field Names

Each field begins with its UTF-8 name. `nameLen` includes the terminating null byte, so the parser reads `nameLen - 1` visible characters and then advances by the full `nameLen`.

```js
let off = m.offset;

const name = readString(dataBlock, off, m.nameLen - 1);
off += m.nameLen;
```

Everything after the name, up to the next field offset, belongs to that field's raw value.

```js
const rawData = dataBlock.slice(off, nextOffset);
```

---

# Step 7 — Decode Values

The parser does not know primitive types ahead of time. Instead, `parseFieldValue()` attempts a small set of interpretations in a fixed order.

The current implementation follows this order:

```
Empty field
↓
null
↓
Boolean / single byte / single ASCII character
↓
String
↓
Embedded DSON
↓
Integer
↓
Float
↓
Unknown binary placeholder
```

If no interpretation is considered reliable, the parser preserves the field as a placeholder that records the byte count.

```js
return `<data:${fieldData.length}b>`;
```

This avoids silently inventing a value for undocumented binary structures.

---

# Alignment

Many primitive values begin on a 4-byte boundary. Rather than assuming the value starts immediately after the field name, the parser calculates the padding required from the value's starting offset inside the data block.

```js
function getAlignmentPadding(offset) {
    return (4 - (offset % 4)) % 4;
}

const padding = getAlignmentPadding(dataOffsetInFile);
```

When a value is interpreted as a 32-bit integer or 32-bit float, the parser reads it after that padding.

```js
const view = new DataView(
    fieldData.buffer,
    fieldData.byteOffset + padding,
    4
);

const intVal = view.getInt32(0, true);
```

Without respecting alignment, integers and floating-point values can be read from the wrong byte position.

---

# Embedded DSON Files

Some string-like fields contain another complete DSON document. Before returning a string, the parser checks whether the string payload begins with the DSON magic number.

```js
const stringDataBytes = fieldData.slice(
    padding + 4,
    padding + 4 + Math.min(4, strlen)
);
const magic = Array.from(stringDataBytes);

if (magic.length === 4 && magic.join(',') === MAGIC.join(',')) {
    const embeddedBuffer = fieldData.buffer.slice(
        fieldData.byteOffset + padding + 4,
        fieldData.byteOffset + padding + 4 + strlen - 1
    );

    const result = decodeDSON(embeddedBuffer);
    return result.data;
}
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

Initially every field exists in a flat array. The parser rebuilds the hierarchy using a stack and the direct-child counts from Meta1.

```js
const root = [];
const fieldStack = [];

for (const field of fields) {
    if (field.isObject) {
        field.children = [];
        const expected =
            (meta1[field.meta1Idx] &&
                meta1[field.meta1Idx].numDirectChildren) || 0;
        field.expectedChildren = expected;
    }

    if (fieldStack.length === 0) {
        root.push(field);
    } else {
        field.parent = fieldStack[fieldStack.length - 1];
        field.parent.children.push(field);
    }

    if (field.isObject) {
        fieldStack.push(field);
    }

    while (
        fieldStack.length > 0 &&
        fieldStack[fieldStack.length - 1].children.length >=
            fieldStack[fieldStack.length - 1].expectedChildren
    ) {
        fieldStack.pop();
    }
}
```

Objects are automatically closed once they have received the expected number of children recorded in Meta1. This allows the hierarchy to be reconstructed without explicit closing markers.

---

# Step 10 — Convert to JSON

Finally, the reconstructed tree is recursively converted into plain JavaScript objects.

```js
function toJsonField(f) {
    if (f.isObject) {
        const obj = {};
        if (f.children) {
            for (const c of f.children) {
                obj[c.name] = toJsonField(c);
            }
        }
        return obj;
    }

    return f.value !== undefined ? f.value : null;
}

const result = {};
for (const r of root) {
    result[r.name] = toJsonField(r);
}
```

The result is standard JSON suitable for viewing, editing, and exporting.

---

# Example Output Snippet

A decoded save can contain a top-level `base_root`, global save values, and nested hero records. The snippet below is shortened from an actual decoded output.

```json
{
  "base_root": {
    "version": 513,
    "nextGuid": 784,
    "dismissed_hero_count": 20,
    "heroes": {
      "1": {
        "hero_file_data": {
          "raw_data": {
            "base_root": {
              "actor": {
                "name": "Reynauld",
                "current_hp": 1113063424,
                "stunned": 0,
                "combat_ready": false,
                "damage_source_type": 5,
                "damage_type": 5,
                "colour_variation": 0
              },
              "heroClass": "crusader",
              "resolveXp": 40,
              "skills": {
                "selected_combat_skills": {
                  "smite": 0,
                  "stunning_blow": 0,
                  "bulwark_of_faith": 0,
                  "inspiring_cry": 0
                },
                "selected_camping_skills": {
                  "encourage": 0,
                  "stand_tall": 0,
                  "zealous_speech": 0
                }
              }
            }
          }
        }
      }
    },
    "highest_resolve_xp": 40
  }
}
```

Unknown binary values are represented as placeholders such as `"<data:6b>"`, meaning the parser preserved a 6-byte field instead of guessing the wrong type.

---

# Design Decisions

This implementation intentionally prioritizes cautious decoding.

- Parse each section only once.
- Use header offsets instead of scanning.
- Preserve unknown binary values as byte-count placeholders.
- Respect alignment before decoding primitives.
- Detect embedded DSON files recursively.
- Reconstruct objects using metadata instead of guessing.
- Only convert to JSON after the hierarchy has been rebuilt.

---

# Credits

The understanding of the DSON format and the core parsing approach are based on the reverse engineering work by **[robojumper](https://github.com/robojumper/DarkestDungeonSaveEditor)** in the original **[DarkestDungeonSaveEditor](https://github.com/robojumper/DarkestDungeonSaveEditor)** project.

The browser interface, drag-and-drop workflow, JSON viewer, and client-side application wrapper were developed for this project.

Where this document shows DSON parsing behavior such as the magic number, header layout, Meta1 structure, Meta2 structure, packed `fieldInfo` decoding, alignment handling, embedded DSON recognition, or object reconstruction, that behavior should be credited to **[robojumper's DarkestDungeonSaveEditor](https://github.com/robojumper/DarkestDungeonSaveEditor)** reverse engineering work. The snippets themselves are written in this project's JavaScript implementation style unless otherwise stated.
