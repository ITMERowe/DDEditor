# Darkest Dungeon Save Editor

A browser-based save editor for **Darkest Dungeon** that reads the game's binary DSON save format and presents it as editable JSON.

Everything runs entirely in your browser—no installation or backend required.

## Features

- Browser-based save editor
- Drag & drop save files
- 100% client-side
- View and edit saves as JSON
- Export modified saves

---

## How It Works

Darkest Dungeon stores save files in **DSON**, a binary serialization format. The editor parses the file, reconstructs its object hierarchy, and converts it into editable JSON.

For a detailed breakdown of the format, see **[docs/DSON_FORMAT.md](docs/DSON_FORMAT.md)**.

### 1. Verify the file

Every DSON file begins with a fixed magic number.

```js
const MAGIC = [0x01, 0xB1, 0x00, 0x00];
```

---

### 2. Read the header

The header contains the offsets and sizes of every section in the file.

```js
const headerLength = reader.readInt32();
const dataOffset = reader.readInt32();
```

---

### 3. Parse metadata

Object and field metadata describe the structure of the save without storing actual values.

```js
const parentIndex = reader.readInt32();
const fieldInfo = reader.readInt32();
```

---

### 4. Read the data block

Using the metadata offsets, the parser extracts the raw data section.

```js
const dataBlock = bytes.slice(dataOffset, dataOffset + dataLength);
```

---

### 5. Decode fields

Each metadata entry points to a field name followed by its binary value.

```js
const name = readString(dataBlock, offset, nameLength - 1);
```

---

### 6. Infer primitive values

Primitive types are inferred from the binary representation.

```js
if (fieldData.length === 4)
    return view.getInt32(0, true);
```

Unknown values are preserved as raw binary instead of guessed.

---

### 7. Rebuild the hierarchy

The metadata is used to reconstruct the original nested object tree.

```js
field.parent.children.push(field);
```

---

### 8. Export JSON

The reconstructed tree is recursively converted into standard JSON.

```js
return field.value ?? null;
```

Example:

```json
{
  "base_root": {
    "version": 513,
    "heroes": {
      "1": {
        "heroClass": "crusader"
      }
    }
  }
}
```

---

## Credits

The DSON parsing logic is adapted from the reverse engineering work by **robojumper**:

https://github.com/robojumper/DarkestDungeonSaveEditor

This project packages that parser into a modern browser-based editor with a graphical interface and client-side workflow.

## License

Please refer to the original project's license regarding the underlying DSON parsing implementation.

The browser interface, UI, and application wrapper are original additions built around that parser.