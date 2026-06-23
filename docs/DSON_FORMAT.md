# DSON Format Overview

Darkest Dungeon stores its save files using **DSON**, a proprietary binary serialization format.

Although the application refers to this process as **decoding**, the save files are **not encrypted**. The editor parses and deserializes the binary data into a JSON representation.

## File Structure

A DSON save consists of four major sections:

1. Header
2. Object Metadata
3. Field Metadata
4. Data Block

The metadata describes the structure of the save, while the data block stores field names and values.

---

## 1. Verify the Header

Every DSON file begins with a fixed magic number.

```js
const MAGIC = [0x01, 0xB1, 0x00, 0x00];
```

Files with an invalid header are rejected immediately.

---

## 2. Read the Header

The header contains offsets and sizes for each section of the file.

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

These offsets allow the parser to jump directly to each section.

---

## 3. Parse Object Metadata

The first metadata table defines the hierarchy of objects.

```js
meta1.push({
    parentIndex: reader.readInt32(),
    meta2EntryIdx: reader.readInt32(),
    numDirectChildren: reader.readInt32(),
    numAllChildren: reader.readInt32()
});
```

No actual values are stored here—only structural relationships.

---

## 4. Parse Field Metadata

Each field is described by an entry containing:

- field name hash
- offset into the data block
- packed metadata

```js
const nameHash = reader.readInt32();
const offset = reader.readInt32();
const fieldInfo = reader.readInt32();
```

The packed `fieldInfo` stores several values inside one integer:

```
bit 0       object flag
bits 2–10   field name length
bits 11–31  object metadata index
```

---

## 5. Read the Data Block

The data block contains field names followed by their binary values.

```js
const dataBlock = bytes.slice(
    dataOffset,
    dataOffset + dataLength
);
```

---

## 6. Read Fields

Each metadata entry points to a location inside the data block.

The parser reads the field name first, then interprets the remaining bytes as the field value.

```js
const name = readString(
    dataBlock,
    offset,
    nameLength - 1
);

offset += nameLength;

const rawValue = dataBlock.slice(
    offset,
    nextOffset
);
```

At this stage, the parser has a flat list of fields.

---

## 7. Infer Primitive Types

DSON does not explicitly store primitive types.

Instead, the parser infers them from the binary data.

```js
if (fieldData.length === 1) {
    const byte = fieldData[0];

    if (byte === 0 || byte === 1)
        return byte === 1;

    return byte;
}
```

Values may become:

- strings
- integers
- floats
- booleans
- nested objects

Unknown structures are preserved as raw binary placeholders rather than guessed.

Example:

```json
{
    "affliction_type_id": "<data:6b>",
    "virtue_type_id": "<data:6b>"
}
```

---

## 8. Rebuild the Object Hierarchy

The metadata tables are used to reconstruct the original object tree.

```js
if (field.isObject) {
    field.children = [];
}
```

Example hierarchy:

```
base_root
└── heroes
    └── "1"
        └── hero_file_data
            └── raw_data
                └── base_root
                    ├── actor
                    ├── heroClass
                    ├── quirks
                    ├── skills
                    └── trinkets
```

---

## 9. Export as JSON

The reconstructed tree is recursively converted into standard JSON.

```js
function toJsonField(field) {
    if (field.isObject) {
        const object = {};

        for (const child of field.children || []) {
            object[child.name] = toJsonField(child);
        }

        return object;
    }

    return field.value ?? null;
}
```

Example output:

```json
{
  "base_root": {
    "version": 513,
    "heroes": {
      "1": {
        "hero_file_data": {
          "raw_data": {
            "base_root": {
              "actor": {
                "name": "Reynauld"
              }
            }
          }
        }
      }
    }
  }
}
```

## Credits

The parsing approach is adapted from the reverse engineering work by **robojumper** in the original Darkest Dungeon Save Editor project.