# Darkest Dungeon Save Editor

A browser-based save editor for **Darkest Dungeon** that reads the game's binary DSON save format and presents it as editable JSON.

The application runs entirely in the browser, no installation or backend required.

## Features

* Browser-based save editor
* Drag & drop save files
* 100% client-side processing
* View save data as formatted JSON
* Export edited JSON
* No external dependencies

## How It Works

Darkest Dungeon stores its save files using **DSON**, a proprietary binary serialization format.

Rather than decrypting the file, the editor:

1. Reads the DSON header.
2. Parses the metadata tables describing every object and field.
3. Reconstructs the original object hierarchy.
4. Interprets each binary value as its corresponding data type (string, integer, float, boolean, etc.).
5. Represents the reconstructed structure as human-readable JSON for viewing and editing.

In other words, this project **deserializes** the game's binary save format into a structured JSON representation.

A simplified version of the parsing flow looks like this:

```js
function parseDSON(buffer) {
  const reader = new BinaryReader(buffer);

  // 1. Verify this is a DSON file
  const magic = reader.readBytes(4);
  if (!isValidMagic(magic)) {
    throw new Error("Invalid DSON file");
  }

  // 2. Read header information
  const header = readHeader(reader);

  // 3. Read metadata tables
  const meta1 = readObjectMetadata(reader, header.meta1Offset, header.numMeta1);
  const meta2 = readFieldMetadata(reader, header.meta2Offset, header.numMeta2);

  // 4. Read the raw data block
  const dataBlock = readBytesAt(buffer, header.dataOffset, header.dataLength);

  // 5. Convert metadata + raw bytes into flat fields
  const fields = meta2.map(entry => {
    const name = readFieldName(dataBlock, entry.offset, entry.nameLength);
    const rawValue = readFieldValueBytes(dataBlock, entry);

    return {
      name,
      isObject: entry.isObject,
      meta1Index: entry.meta1Index,
      value: entry.isObject ? null : parseValue(rawValue)
    };
  });

  // 6. Rebuild the nested save structure
  const root = rebuildObjectTree(fields, meta1);

  // 7. Convert to readable JSON
  return JSON.stringify(root, null, 2);
}
```

In short:

```text
DSON binary save
      ↓
read header
      ↓
parse metadata
      ↓
read binary values
      ↓
rebuild object tree
      ↓
JSON output
```

So this project “decodes” the save file in the practical sense, but technically it is **parsing and deserializing** the DSON binary format rather than decrypting it.


## Credits

The DSON parsing implementation is based on the excellent reverse engineering work by **robojumper** in the original project:

https://github.com/robojumper/DarkestDungeonSaveEditor

This repository does **not** attempt to reverse engineer the DSON format independently. Instead, it adapts the existing parsing logic into a modern browser application with a graphical interface and client-side workflow.

## Usage

1. Open the webpage.
2. Drag a Darkest Dungeon save file onto the page.
3. The save is parsed automatically.
4. View or edit the JSON representation.
5. Copy or export the result.

## License

Please refer to the original project's license regarding the underlying DSON parsing implementation.

The browser interface, UI, and application wrapper in this repository are original additions built around that parser.
