# DSON Format

This document describes the **DSON** (Darkest Serialized Object Notation) format used by **Darkest Dungeon**, and explains how this project parses and reconstructs save files.

Unlike JSON, DSON is a compact binary serialization format optimized for fast loading by the game engine. Object names, metadata, and values are stored separately, allowing the game to reconstruct complex object graphs without repeatedly storing structural information.

This document describes how the browser parser converts a DSON file into editable JSON.

---

# File Layout

A DSON save file is divided into four primary sections.

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

Each section serves a different purpose.

| Section | Purpose |
|----------|---------|
| Header | Describes where every section is located. |
| Meta1 | Defines the object hierarchy. |
| Meta2 | Defines every field and where its data lives. |
| Data | Stores field names and binary values. |

The parser reads these sections independently before reconstructing the original object tree.

---

# Step 1 — Verify the File

Every DSON file begins with the same four-byte magic number.

```js
const MAGIC = [0x01, 0xB1, 0x00, 0x00];
```

The parser validates these bytes before attempting to read anything else.

If the magic number does not match, the file is rejected immediately.

This prevents accidentally interpreting unrelated binary files as DSON saves.

---

# Step 2 — Read the Header

The first 64 bytes form the DSON header.

The header does **not** contain gameplay data.

Instead, it describes where the remaining sections begin.

```js
const revision = reader.readInt32();
const headerLength = reader.readInt32();

const meta1Offset = reader.readInt32();
const meta2Offset = reader.readInt32();

const dataOffset = reader.readInt32();
const dataLength = reader.readInt32();
```

Rather than scanning the file sequentially, the parser jumps directly to each section using these offsets.

The revision field also contains the game's build number.

```
buildNumber = revision >> 16
```

---

# Step 3 — Parse Object Metadata (Meta1)

Meta1 contains one entry for every object in the save.

Each entry stores only structural information.

```js
{
    parentIndex,
    meta2EntryIdx,
    numDirectChildren,
    numAllChildren
}
```

Meaning:

| Field | Description |
|------|-------------|
| parentIndex | Parent object index |
| meta2EntryIdx | Corresponding Meta2 entry |
| numDirectChildren | Immediate child count |
| numAllChildren | Total descendants |

Notice that **no values are stored here**.

Meta1 only defines relationships between objects.

---

# Step 4 — Parse Field Metadata (Meta2)

Meta2 contains one entry for every field.

```js
{
    nameHash,
    offset,
    fieldInfo
}
```

The interesting part is `fieldInfo`.

Instead of storing several integers separately, DSON packs multiple values into one 32-bit integer.

```
31                             11 10      2 1 0
+--------------------------------+---------+-+
| Meta1 Index                    | NameLen |O|
+--------------------------------+---------+-+
```

Where:

- bit 0 = object flag
- bits 2–10 = field name length
- bits 11–31 = Meta1 index

The parser extracts these values using bit operations.

```js
const isObject = (fieldInfo & 1) === 1;
const nameLength = (fieldInfo >> 2) & 0x1FF;
const meta1Index = fieldInfo >> 11;
```

Packing several values into one integer reduces file size while remaining quick to decode.

---

# Step 5 — Read the Data Block

The data block stores the actual bytes.

Unlike Meta1 and Meta2, it contains:

- field names
- strings
- integers
- floats
- nested DSON blobs
- unknown binary structures

```js
const dataBlock = bytes.slice(
    dataOffset,
    dataOffset + dataLength
);
```

Metadata answers **where** data lives.

The data block contains **what** the data actually is.

---

# Step 6 — Read Field Names

Each Meta2 entry contains an offset into the data block.

At that offset, the parser reads:

```
Field Name
↓

heroClass\0

Immediately followed by

↓

binary value
```

```js
const name = readString(
    dataBlock,
    offset,
    nameLength - 1
);
```

The next metadata entry determines where the current field ends.

Everything between those offsets belongs to the current field.

---

# Step 7 — Alignment

Not every value begins immediately after the field name.

Many primitive values are aligned to 4-byte boundaries.

The parser calculates the required padding before interpreting data.

```js
const padding = (4 - (offset % 4)) % 4;
```

Without respecting alignment, integers and floats would be read incorrectly.

---

# Step 8 — Decode Primitive Values

Unlike JSON, DSON does **not** explicitly record the type of every value.

Instead, the parser infers types from the binary layout.

Examples include:

### Boolean

```text
00 = false
01 = true
```

### Integer

```
4 bytes
Little Endian
```

### Float

```
IEEE-754
Little Endian
```

### String

```
[length][UTF-8 bytes][null]
```

The parser attempts each interpretation in order until one matches.

---

# Step 9 — Embedded DSON Objects

Some values are not primitive types.

Instead, they contain an entire DSON document embedded inside another one.

The parser detects this by checking for another DSON magic number.

```
01 B1 00 00
```

When found, it recursively calls the decoder.

```js
decodeDSON(embeddedBuffer)
```

This allows structures such as hero data to contain complete nested save objects.

---

# Step 10 — Unknown Values

Not every binary structure has been reverse engineered.

Rather than guessing, the parser preserves unknown data.

Example:

```json
{
    "affliction_type_id":"<data:6b>"
}
```

This guarantees the editor never silently corrupts values it does not understand.

---

# Step 11 — Rebuild the Object Tree

At this point every field exists in a flat array.

Meta1 tells the parser which fields belong inside which objects.

The parser reconstructs the hierarchy using a stack.

```
base_root
└── heroes
    └── "1"
        └── hero_file_data
            └── raw_data
```

Objects are pushed onto the stack when entered and removed once their expected number of children has been processed.

This avoids recursive metadata lookups while rebuilding the tree.

---

# Step 12 — Convert to JSON

Finally, every object is recursively converted into a standard JavaScript object.

```js
function toJsonField(field) {

    if(field.isObject){

        const object = {};

        for(const child of field.children){
            object[child.name] = toJsonField(child);
        }

        return object;
    }

    return field.value;
}
```

The result is ordinary JSON that can be viewed, edited, exported, and re-encoded if desired.

---

# Design Notes

Several design decisions were made to preserve save integrity.

- Unknown binary structures are never guessed.
- Alignment is respected before reading primitive values.
- Embedded DSON documents are decoded recursively.
- Metadata is parsed before values to avoid unnecessary file scans.
- Object hierarchies are reconstructed from metadata rather than inferred.

These choices prioritize correctness over aggressive interpretation.

---

# Credits

The DSON parsing logic used by this project is adapted from the reverse engineering work by **robojumper**.

This document explains how the adapted browser implementation reconstructs DSON saves into editable JSON.