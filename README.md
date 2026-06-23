# Darkest Dungeon Save Editor

A browser-based save editor for **Darkest Dungeon** that reads the game's binary DSON save format and presents it as editable JSON.

The application runs entirely in the browser, with no installation or backend required.

## Features

- Browser-based save editor
- Drag & drop save files
- 100% client-side processing
- View save data as formatted JSON
- Export edited JSON
- No external dependencies

---

## How It Works

Darkest Dungeon stores its save files using **DSON**, a proprietary binary serialization format.

Although the application refers to this process as **"decoding"**, the save file is **not encrypted**. Instead, the editor parses the binary DSON structure and reconstructs it into a JSON object.

In other words, this project **decodes** the save file in the practical sense, but technically it is **parsing and deserializing** the DSON binary format rather than decrypting it.

---

### 1. Verify the DSON Header

Every DSON file begins with a fixed magic number.

```js
const MAGIC = [0x01, 0xB1, 0x00, 0x00];

const magic = Array.from(bytes.slice(0, 4));

if (magic.join(",") !== MAGIC.join(",")) {
    throw new Error("Invalid DSON file");
}
```

If the magic number does not match, the parser immediately rejects the file.

---

### 2. Read the Header

The header contains the offsets and sizes for every section of the file.

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

Rather than scanning the entire file, the parser uses these offsets to jump directly to the object metadata, field metadata, and raw data sections.

---

### 3. Parse Object Metadata

The first metadata table describes the hierarchy of every object stored in the save.

```js
meta1.push({
    parentIndex: reader.readInt32(),
    meta2EntryIdx: reader.readInt32(),
    numDirectChildren: reader.readInt32(),
    numAllChildren: reader.readInt32()
});
```

This information doesn't contain any values—it simply describes how objects relate to one another so the hierarchy can be reconstructed later.

---

### 4. Parse Field Metadata

The second metadata table describes every field stored in the save.

```js
const nameHash = reader.readInt32();
const offset = reader.readInt32();
const fieldInfo = reader.readInt32();

const isObject = (fieldInfo & 1) === 1;
const nameLength = (fieldInfo >> 2) & 0x1FF;
const meta1Index = fieldInfo >> 11;
```

The `fieldInfo` integer packs several values into a single 32-bit number.

```
bit 0       → object flag
bits 2–10   → field name length
bits 11–31  → object metadata index
```

This compact representation is one of the reasons DSON files are unreadable without a parser.

---

### 5. Read the Data Block

The data block contains the actual field names and binary values.

```js
const dataBlock = bytes.slice(
    dataOffset,
    dataOffset + dataLength
);
```

Metadata tells the parser **where** everything is.

The data block contains **what** everything is.

---

### 6. Read Field Names and Values

Each metadata entry points to a location inside the data block.

The parser first reads the field name, then everything after that becomes the field's raw value until the next field begins.

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

After this step, the parser has a flat list of fields.

---

### 7. Infer Primitive Types

Unlike formats such as JSON, DSON does not explicitly store every value's type.

Instead, the parser infers primitive values from their binary representation.

```js
if (fieldData.length === 0) {
    return null;
}

if (fieldData.length === 1) {
    const byte = fieldData[0];

    if (byte === 0 || byte === 1)
        return byte === 1;

    return byte;
}

if (fieldData.length === 4) {
    return view.getInt32(0, true);
}
```

Depending on the binary data, values may become:

- Strings
- Integers
- Floats
- Booleans
- Nested DSON objects

If the parser cannot confidently identify a value, it preserves the original bytes instead of guessing.

```json
{
    "affliction_type_id": "<data:6b>",
    "virtue_type_id": "<data:6b>",
    "dungeon_history": "<data:19b>"
}
```

This ensures that unknown structures are never silently corrupted.

---

### 8. Rebuild the Object Tree

After parsing every field, the save still exists as a flat list.

The object metadata is then used to reconstruct the original hierarchy.

```js
if (field.isObject) {
    field.children = [];
    field.expectedChildren =
        meta1[field.meta1Idx]?.numDirectChildren ?? 0;
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
    fieldStack.length &&
    fieldStack[fieldStack.length - 1].children.length >=
    fieldStack[fieldStack.length - 1].expectedChildren
) {
    fieldStack.pop();
}
```

An actual decoded save is reconstructed into a hierarchy like this:

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
                    ├── trinkets
                    └── item_tracking
```

---

### 9. Export as JSON

Finally, the reconstructed object tree is converted into standard JSON.

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

Example output from a real save file:

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
              "roster.status": 0,
              "roster.building_name": "tavern",
              "actor": {
                "name": "Reynauld",
                "combat_ready": false
              },
              "heroClass": "crusader",
              "resolveXp": 40,
              "weapon_rank": 4,
              "armour_rank": 3
            }
          }
        }
      }
    }
  }
}
```

The result is a human-readable representation of the original DSON save file that can be viewed, edited, and exported.

---

## Credits

The DSON parsing logic is adapted from the reverse engineering work by **robojumper** in the original project:

https://github.com/robojumper/DarkestDungeonSaveEditor

This repository does **not** attempt to reverse engineer the DSON format independently. Instead, it adapts the existing parsing logic into a modern browser application with a graphical interface and client-side workflow.

The browser interface, drag-and-drop workflow, JSON viewer, and client-side application wrapper were developed for this project.

---

## Usage

1. Open the webpage.
2. Drag a Darkest Dungeon save file onto the page.
3. The save is parsed automatically.
4. View or edit the JSON representation.
5. Copy or export the result.

---

## License

Please refer to the original project's license regarding the underlying DSON parsing implementation.

The browser interface, UI, and application wrapper in this repository are original additions built around that parser.
