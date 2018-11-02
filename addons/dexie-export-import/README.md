# Export and Import IndexedDB Database

Export / Import IndexedDB <---> Blob

This module extends 'dexie' module with new methods for importing / exporting databases to / from blobs.

# Install
```
npm install dexie
npm install dexie-export-import
```

# Features

* Export of IndexedDB Database to JSON Blob.
* Import from Blob back to IndexedDB Database.
* Chunk-wise / Streaming - does not read the entire DB into RAM
* Progress callback (typically for showing progress bar)
* Optional filter allows to import/export subset of data
* Support for all structured clonable exotic types (Date, ArrayBuffer, Blob, etc) except CryptoKeys (which by design cannot be exported)
* Atomic - import / export within one database transaction (optional)
* Export speed: Using getAll() in chunks rather than openCursor().
* Import speed: Using bulkPut() in chunks rather than put().
* Can also export IndexedDB databases that was not created with Dexie.

# Similar Libraries
## [indexeddb-export-import](https://github.com/Polarisation/indexeddb-export-import)
 
Much smaller in size, but also much lighter than dexie-export-import.

[Indexeddb-export-import](https://github.com/Polarisation/indexeddb-export-import) can be better choice if...

* ...your data contains no Dates, ArrayBuffers, TypedArrays or Blobs (only objects, strings, numbers, booleans and arrays).
* ...your database is small enough to fit in RAM on your target devices.

Dexie-export-import tries to scale and support

# Usage

Here's the basic usage. There's a lot you can do by supplying optional `[options]` arguments. The available options are described later on in this README (See Typescript interfaces below).

*NOTE:* Typescript users using dexie@2.x will get compilation errors if using the static import method `Dexie.import()`. 

```js
import Dexie from "dexie";
import "dexie-export-import";

//
// Import from Blob or File to Dexie instance:
//
const db = await Dexie.import(blob, [options]);

//
// Export to Blob
//
const blob = await db.export([options]);

//
// Import from Blob or File to existing Dexie instance
//
await db.import(blob, [options]);

```

# Extended Dexie Interface

Importing this module will extend Dexie and Dexie.prototype as follows.
Even though this is conceptually a Dexie.js addon, there is no addon instance.
Extended interface is done into Dexie and Dexie.prototype as a side effect when
importing the module.

```ts
//
// Extend Dexie interface (typescript-wise)
//
declare module 'dexie' {
  // Extend methods on db
  interface Dexie {
    export(options?: ExportOptions): Promise<Blob>;
    import(blob: Blob, options?: ImportOptions): Promise<void>;
  }
  interface DexieConstructor {
    import(blob: Blob, options?: StaticImportOptions): Promise<Dexie>;
  }
}
```

# Compatibility

| Product | Required version          |
| ------- | ------------------------- |
| dexie   | ^2.0.4 and ^3.0.0-alpha.5 |
| Safari  | ^10.1                     |
| IE      | ^11                       |
| Chrome  | any version               |
| FF      | any version               |


## Import Options

```ts
export interface StaticImportOptions {
  noTransaction?: boolean;
  chunkSizeBytes?: number; // Default: DEFAULT_KILOBYTES_PER_CHUNK ( 1MB )
  filter?: (table: string, value: any, key?: any) => boolean;
  progressCallback?: (progress: ImportProgress) => boolean;
}

export interface ImportOptions extends StaticImportOptions {
  acceptMissingTables?: boolean;
  acceptVersionDiff?: boolean;
  acceptNameDiff?: boolean;
  acceptChangedPrimaryKey?: boolean;
  overwriteValues?: boolean;
  clearTablesBeforeImport?: boolean;
  noTransaction?: boolean;
  chunkSizeBytes?: number; // Default: DEFAULT_KILOBYTES_PER_CHUNK ( 1MB )
  filter?: (table: string, value: any, key?: any) => boolean;
  progressCallback?: (progress: ImportProgress) => boolean;
}

```

## Import Progress

```ts
export interface ImportProgress {
  totalTables: number;
  completedTables: number;
  totalRows: number;
  completedRows: number;
  done: boolean;
}
``` 

## Export Options

```ts
export interface ExportOptions {
  noTransaction?: boolean;
  numRowsPerChunk?: number;
  prettyJson?: boolean;
  filter?: (table: string, value: any, key?: any) => boolean;
  progressCallback?: (progress: ExportProgress) => boolean;
}
```

## Export Progress

```ts
export interface ExportProgress {
  totalTables: number;
  completedTables: number;
  totalRows: number;
  completedRows: number;
  done: boolean;
}
```

## Defaults
```ts
const DEFAULT_KILOBYTES_PER_CHUNK = 1024; // When importing blob
const DEFAULT_ROWS_PER_CHUNK = 2000; // When exporting db
```

# JSON Format

The JSON format is described in the Typescript interface below. This JSON format is streamable as it is generated
in a streaming fashion, and imported also using a streaming fashion. Therefore, it is important that the data come
last in the file.

```ts
export interface DexieExportJsonStructure {
  formatName: 'dexie';
  formatVersion: 1;
  data: {
    databaseName: string;
    databaseVersion: number;
    tables: Array<{
      name: string;
      schema: string; // '++id,name,age'
      rowCount: number;
    }>;
    data: Array<{ // This property must be last (for streaming purpose)
      tableName: string;
      inbound: boolean;
      rows: any[]; // This property must be last (for streaming purpose)
    }>;
  }
}
```

# Sample

This sample shows a download link and a drop area for importing files back into the database.

## NPM

```npm
npm install dexie
npm install dexie-export-import
npm install downloadjs
```

## CSS

```css
#dropzone {
  width: 600px;
  height: 100px;
  border: 2px dotted #bbb;
  border-radius: 10px;
  padding: 35px;
  color: #bbb;
  text-align: center;
}
```

## HTML

```html
<p>
  <a id="exportLink" href="#">Click here to export the database</a>
</p>
<div id="dropzone">
  Drop dexie export JSON file here
</div>
```

## Javascript

```js
import Dexie from 'dexie';
import 'dexie-export-import';
import download from 'downloadjs';

//
// Declare Database and pre-populate it
//
const db = new Dexie('exportSample');
db.version(1).stores({
  foos: 'id'
});
db.on('populate', ()=>{
  return db.foos.bulkAdd([
    {
      id: 1,
      foo: 'foo',
      date: new Date(), // Dates, Blobs, ArrayBuffers, etc are supported
    },{
      id: 2,
      foo: 'bar',
    }
  ]);
});

//
// When document is ready, bind export/import funktions to HTML elements
//
document.addEventListener('DOMContentLoaded', ()=>{
  const dropZoneDiv = document.getElementById('drop-zone');
  const exportLink = document.getElementById('exportLink');

  // Configure exportLink
  exportLink.onclick = async ()=>{
    try {
      const blob = await db.export({prettyJson: true});
      download(blob, "dexie-export.json", "application/json");
    } catch (error) {
      console.error(error);
    }
  };

  // Configure dropZoneDiv
  dropZoneDiv.ondragover = event => {
    event.stopPropagation();
    event.preventDefault();
    event.dataTransfer.dropEffect = 'copy';
  };

  // Handle file drop:
  dropZoneDiv.ondragover = async event => {
    event.stopPropagation();
    event.preventDefault();

    // Pick the File from the drop event (a File is also a Blob):
    const file = ev.dataTransfer.files[0];
    try {
      if (!file) throw new Error(`Only files can be dropped here`);
      await db.import(file);
      console.log("Import complete");
    } catch (error) {
      console.error(error);
    }
  }
});

```

# Exporting IndexedDB Databases that wasn't generated with Dexie
As Dexie can dynamically open non-Dexie IndexedDB databases, this is not an issue.
Sample provided here:

```js
import Dexie from 'dexie';
import 'dexie-export-import';

async function exportDatabase(databaseName) {
  const db = await new Dexie(databaseName).open();
  const blob = await db.export();
  return blob;
}

async function importDatabase(file) {
  const db = await Dexie.import(file);
  return db.backendDB();
}
```


## Background / Why

This feature has been asked for a lot:

* https://github.com/dfahlander/Dexie.js/issues/391
* https://github.com/dfahlander/Dexie.js/issues/99
* https://stackoverflow.com/questions/46025699/dumping-indexeddb-data
* https://feathub.com/dfahlander/Dexie.js/+9

My simple answer initially was this:

```js
function export(db) {
    return db.transaction('r', db.tables, ()=>{
        return Promise.all(
            db.tables.map(table => table.toArray()
                .then(rows => ({table: table.name, rows: rows})));
    });
}

function import(data, db) {
    return db.transaction('rw', db.tables, () => {
        return Promise.all(data.map (t =>
            db.table(t.table).clear()
              .then(()=>db.table(t.table).bulkAdd(t.rows)));
    });
}
```

Looks simple!

But:

1. The whole database has to fit in RAM. Can be issue on small devices.
2. If using JSON.stringify() / JSON.parse() on the data, we won't support exotic types (Dates, Blobs, ArrayBuffers, etc)
3. Not possible to show a progress while importing.

This addon solves these issues, and some more, with the help of some libraries.

## Libraries Used
To accomplish a streamable export/import, and allow exotic types, I use the libraries listed below. Note that these libraries are listed as devDependencies because they are bundles using rollupjs - so there's no real dependency from the library user persective.

### [typeson](https://www.npmjs.com/package/typeson) and [typeson-registry](https://www.npmjs.com/package/typeson-registry)
These modules enables something similar as JSON.stringify() / JSON.parse() for exotic or custom types.

### [clarinet](https://www.npmjs.com/package/clarinet)
This module allow to read JSON in a streaming fashion

## Streaming JSON
I must admit that I had to do some research before I understood how to accomplish streaming JSON from client-side Javascript (both reading / writing). It is really not obvious that this would even be possible. Looking at the Blob interface, it does not provide any way of either reading or writing in a streamable fashion.

What I found though (after some googling) was that it is indeed possible to do that based on the current DOM platform (including IE11 !).

### Reading JSON in Chunks

A File or Blob represents something that can lie on a disk file and not yet be in RAM. So how do we read the first 100 bytes from a Blob without reading it all?

```js
const firstPart = blob.slice(0,100);
```
Ok, and in the next step we use a FileReader to really read this sliced Blob into memory.

```ts

const first100Chars = await readBlob(firstPart);

function readBlob(blob: Blob): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onabort = ev => reject(new Error("file read aborted"));
    reader.onerror = ev => reject((ev.target as any).error);
    reader.onload = ev => resolve((ev.target as any).result);
    reader.readAsText(blob);
  });
}
```
Voila!

But! How can we keep transactions alive when calling this non-indexedDB async call?

I use two different solutions for this:
1. If we are in a Worker, I use `new FileReaderSync()` instead of `new FileReader()`.
2. If in the main thread, I use `Dexie.waitFor()` to while reading this short elapsed chunk, keeping the transaction alive still.

Ok, fine, but how do we parse the chunk then? Cannot use JSON.parse(firstPart) because it will most defenitely be incomplete.

[Clarinet](https://www.npmjs.com/package/clarinet) to the rescue. This library can read JSON and callback whenever JSON tokens come in.

### Writing JSON in Chunks

Writing JSON is solved more easily. As the BlobBuilder interface was deprecated from the DOM, I firstly found this task impossible. But after digging around, I found that also this SHOULD be possible if browers implement the Blob interface correctly.

Blobs can be constructeed from an array of other Blobs. This is the key.

1. Let's say we generate 1000 Blobs of 1MB each on a device with 512 MB RAM. If the browser does its job well, it will allow the first 200 blobs or so to reside in RAM. But then, it should start putting the remanding blobs onto temporary files.
2. We put all these 1000 blobs into an array and generate a final Blob from that array.

And that's pretty much it.