# Darkest Dungeon Save Editor

A browser-based save editor for **Darkest Dungeon** that reads the game's binary DSON save format and presents it as editable JSON.

The application runs entirely in the browser—no installation or backend required.

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
