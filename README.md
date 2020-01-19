# Simple Google Spreadsheet Access (node.js)

[![NPM version](https://badge.fury.io/js/google-spreadsheet.png)](http://badge.fury.io/js/google-spreadsheet)

[![CircleCI](https://circleci.com/gh/theoephraim/node-google-spreadsheet.svg?style=svg)](https://circleci.com/gh/theoephraim/node-google-spreadsheet)

Node.js module to make interacting with Google Spreadsheets super easy (_uses v4 sheets API_)

- with or without auth
- cell-based API - read, write, bulk-updates, formatting
- row-based API - read, update, delete (based on the old v3 row-based calls)
- managing worksheets - add, remove, resize, change title, formatting

## Why?

While the v4 sheets api is much easier to use than v3 was, the official google module is a giant meta-tool that handles all of their APIs. The module and the API itself are still awkward and the docs are still terrible.
This module tries to provide simple and logical methods to handle the interactions most projects would need.

## Installation

[![NPM Info](https://nodei.co/npm/google-spreadsheet.png?downloads=true&downloadRank=true&stars=true)](https://www.npmjs.org/package/google-spreadsheet)

## Basic Usage

_This example is simply meant to show some of the things you can do._

Check out the examples folder for more options.

```javascript
const GoogleSpreadsheet = require('google-spreadsheet');
const creds = require('./service-account-creds-from-google.json');

// spreadsheet key is the long id in the sheets URL
const doc = new GoogleSpreadsheet('<the sheet ID from the url>');

(async function main() { // to allow async/await at root level
  // use API key only for read-only access to public sheets
  doc.useApiKey('YOUR-API-KEY');
  // OR more commonly, use a service account for full access
  await doc.useServiceAccountAuth(creds);

  await doc.getInfo(); // load basic doc properties, sheet info
  console.log(doc.title);
  const sheet = doc.sheetsByIndex[0];
  await doc.updateProperties({ title: 'rename this document' })

  // interact with rows, ie first row of sheet is treated as column names / property keys
  const rows = await sheet.getRows({ limit: 10 });
  console.log(rows[0].firstColumn);
  await sheet.addRow({ firstColumn: 'foo', secondColumn: 'bar' });
  rows[0].firstColumn = 'newVal'
  await rows[0].save();
  await rows[0].delete();

  // interact with cells
  await sheet.loadCells('A1:E10'); // loads that range of cells;
  const a1 = sheet.getCell(0, 0); // access cells using a zero-based index
  const c6 = sheet.getCellByA1('C6'); // or A1 style notation
  console.log(a1.formattedValue); // access everything about the cell
  console.log(a1.value);
  console.log(a1.formula);
  a1.value = 123.456; // update value
  c6.formula = '=A1'; // update formula
  a1.format = { bold: true }; // modify formatting
  await sheet.saveUpdatedCells(); // save updates in one call

  // manage worksheets
  const newSheet = await doc.addSheet({ title: 'hot new sheet!' });
  await newSheet.resize({ rowCount: 10, columnCount: 100 });
  await newSheet.delete();
})();
```

## Authentication

IMPORTANT: Google recently deprecated their ClientLogin (username+password)
access, so things are slightly more complicated now. Older versions of this
module supported it, so just be aware that things changed.

### Unauthenticated access (read-only access on public docs)

By default, this module makes unauthenticated requests and can therefore
only access spreadsheets that are "public".

The Google Spreadsheets Data API reference and developers guide is a little
ambiguous about how you access a "published" public Spreadsheet.

If you wish to work with a Google Spreadsheet without authenticating, not only
must the Spreadsheet in question be visible to the web, but it must also have
been explicitly published using "File > Publish to the web" menu option in
the google spreadsheets GUI.

Many seemingly "public" sheets have not also been "published" so this may
cause some confusion.

*Unauthenticated requests allow reading, but not writing to sheets. To write on a sheet, you must authenticate.*


### Service Account (recommended method)

This is a 2-legged oauth method and designed to be "an account that belongs to your application instead of to an individual end user".
Use this for an app that needs to access a set of documents that you have full access to.
([read more](https://developers.google.com/identity/protocols/OAuth2ServiceAccount))

__Setup Instructions__

1. Go to the [Google Developers Console](https://console.developers.google.com/project)
2. Select your project or create a new one (and then select it)
3. Enable the Drive API for your project
  - In the sidebar on the left, expand __APIs & auth__ > __APIs__
  - Search for "drive"
  - Click on "Drive API"
  - click the blue "Enable API" button
4. Create a service account for your project
  - In the sidebar on the left, expand __APIs & auth__ > __Credentials__
  - Click blue "Add credentials" button
  - Select the "Service account" option
  - Select "Furnish a new private key" checkbox
  - Select the "JSON" key type option
  - Click blue "Create" button
  - your JSON key file is generated and downloaded to your machine (__it is the only copy!__)
  - note your service account's email address (also available in the JSON key file)
5. Share the doc (or docs) with your service account using the email noted above


## Google's API Limitations

Google's API is somewhat limiting. Calls are made to two differently designed APIs, one made to deal with cells, and one to deal with rows. These APIs will let you manage the data in your sheets, but you cannot make any modifications to the formatting of the cells.

### Row-Based API Limitations

The row-based API assumes that the "header row" (first row) of your sheet is set. They have limitations on the column names they will accept - all lowercase with no symbols or spaces. If the values in your sheet do not follow their rules, their API will adapt the key it actually returns to you. I recommend just following their rules to avoid confusion.

You _can_ set a formula value into a cell using the row-based API, but when reading rows, you cannot access the formula, or even be aware that there is one in the cell. Any cells with formulas will return the calculated value of the formula. If you try to update a row, the cell with a formula will be overwritten to its calculated value.

**IMPORTANT** The row-based API also assumes there are no empty rows in your sheet. If any row is completely empty, you will not be able to access any rows after the empty row using the row-based API.

-----------------------------------------

## API

This module follows "normal" node callback conventions:

- Every method that takes a callback takes it as its last param
- Every callback will be called with the error (or null) as first param
- Some methods have optional params

### `GoogleSpreadsheet`

The main class that represents an entire spreadsheet.


#### `new GoogleSpreadsheet(sheet_id, [auth], [options])`

Create a new google spreadsheet object.

- `sheet_id` -- the ID of the spreadsheet (from its URL)
- `auth` - (optional) an existing auth token
- `options` - (optional)
  - `visibility` - defaults to `public` if anonymous
  - `projection` - defaults to `values` if anonymous



#### `GoogleSpreadsheet.useServiceAccountAuth(account_info, callback)`

Uses a service account email and public/private key to create a token to use to authenticated requests.
Normally you would just pass in the result of requiring the json file that google generates for you when you create a service account.

See the "Authentication" section for more info.

If you are using heroku or another environment where you cannot save a local file, you may just pass in an object with
- `client_email` -- your service account's email address
- `private_key` -- the private key found in the JSON file

Internally, this uses a JWT client to generate a new auth token for your service account that is valid for 1 hour. The token will be automatically regenerated when it expires.

**SPECIAL NOTE FOR HEROKU USERS**

1. Save your private key to a text file
2. Replace `\n` with actual line breaks
3. Replace `\u003d` with `=`
4. heroku config:add GOOGLE_PRIVATE_KEY="$(cat yourfile.txt)"



#### `GoogleSpreadsheet.setAuthToken(id)`

Use an already created auth token for all future requets.



#### `GoogleSpreadsheet.getInfo(callback)`

Get information about the spreadsheet. Calls callback passing an object that contains:

- `id` - the URL/id as returned from google
- `title` - the title of the document
- `updated` - last updated timestamp
- `author` - auth info in an object
  - `name` - author name
  - `email` - author email
- `worksheets` - an array of `SpreadsheetWorksheet` objects (see below)



#### `GoogleSpreadsheet.getRows(worksheet_id, options, callback)`

Get an array of row objects from the sheet.

- `worksheet_id` - the index of the sheet to read from (index starts at 1)
- `options` (optional)
  - `offset` - start reading from row #
  - `limit` - max # of rows to read at once
  - `orderby` - column key to order by
  - `reverse` - reverse results
  - `query` - send a structured query for rows ([more info](https://developers.google.com/google-apps/spreadsheets/data#send_a_structured_query_for_rows))
- `callback(err, rows)` - will be called with an array of SpreadsheetRow objects (see below)

*NOTE* The `reverse` option only works in conjunction with `orderby`. It will not work to reverse the default ordering. This is a known bug in Google's API.

#### `GoogleSpreadsheet.addRow(worksheet_id, new_row, callback)`

Add a single row to the sheet.

- `worksheet_id` - the index of the sheet to add to (index starts at 1)
- `new_row` - key-value object to add - keys must match the header row on your sheet
- `callback(err, row)` - will be called with the new SpreadsheetRow (see below)

#### `GoogleSpreadsheet.getCells(worksheet_id, options, callback)`

Get an array of cell objects.

- `worksheet_id` - the index of the sheet to add to (index starts at 1)
- `options` (optional)
  - `min-row` - row range min (uses #s visible on the left)
  - `max-row` - row range max
  - `min-col` - column range min (uses numbers, not letters!)
  - `max-col` - column range max
  - `return-empty` - include empty cells (boolean)


#### `GoogleSpreadsheet.addWorksheet(options, callback)`

Add a new worksheet to the doc.

- `options` (optional)
  - `title` - title for the new sheet, must be unique in the doc (default = 'Worksheet {timestamp}')
  - `rowCount` - number of rows (default = 50)
  - `colCount` - number of columns (default = 20)
  - `headers` - array of string keys to put in the first row

#### `GoogleSpreadsheet.removeWorksheet(sheet, callback)`

Remove a worksheet from the doc - by id, index, or the SpreadsheetWorksheet object

- `sheet` - can be a SpreadsheetWorksheet object, the id of the sheet, or the index (starts at 1)

----------------------------------

### `SpreadsheetWorksheet`

Represents a single "sheet" from the spreadsheet. These are the different tabs/pages visible at the bottom of the Google Sheets interface.

These are the sheet objects returned as `worksheets` when calling `GoogleSpreadsheet.getInfo`. Many of the calls are accessible from the main Spreadsheet object by passing in a sheet ID (see above), but some functionality is only available on the Worksheet object because it requires various URLs only known after fetching the sheets for making requests.

__Properties:__
- `url` - the URL for the sheet
- `id` - the ID of the sheet
- `title` - the title (visible on the tabs in google's interface)
- `rowCount` - number of rows
- `colCount` - number of columns

### `SpreadsheetWorksheet.getRows(options, callback)`
See above.

### `SpreadsheetWorksheet.getCells(options, callback)`
See above.

### `SpreadsheetWorksheet.addRow(new_row, callback)`
See above.

#### `GoogleSpreadsheet.bulkUpdateCells(cells, callback)`
Do a bulk update on cells.

- `cells` - an array of SpreadsheetCell objects to save

### `SpreadsheetWorksheet.del(callback)`
Remove this sheet from the doc.

#### `SpreadsheetWorksheet.setHeaderRow(values, callback)`
Set the first row of the sheet

- `values` - array of string values to put in the first row of the sheet

#### `SpreadsheetWorksheet.clear(callback)`
Clears the entire sheet's contents

#### `SpreadsheetWorksheet.resize(options, callback)`
Set the dimensions of the sheet

- `options`
  - `rowCount` - number of rows
  - `colCount` - number of columns

#### `SpreadsheetWorksheet.setTitle(title, callback)`
Set the title of the sheet

- `title` - new title for the worksheet



----------------------------------

### `SpreadsheetRow`
Represents a single row from a sheet.

These are returned from calling `GoogleSpreadsheet.getRows` and `SpreadsheetWorksheet.getRows`.

You can treat the row as a normal javascript object. Object keys will be from the header row of your sheet, however the google API mangles the names a bit to make them simpler. It's easiest if you just use all lowercase keys to begin with.

See limitations above for notes about Google's row-based API!

#### `SpreadsheetRow.save( callback )`
Saves any changes made to the row's values.

#### `SpreadsheetRow.del( callback )`
Deletes the row from the sheet.

----------------------------------

### `SpreadsheetCell`
Represents a single cell from the sheet.
Using cells is the only way to read and modify the formulas in your sheet.

__Properties:__
- `id` - the ID of the cell
- `row` - the row this cell is in
- `col` - the column this cell is in
- `value` - the value of the cell as a string
- `formula` - the formula present in the cell, for example `=SUM(A3:B3)` (if applicable)
- `numericValue` - the value of the cell as a number (if applicable)

__IMPORTANT__:
- You can modify `value`, `numericValue`, or `formula`, and things will work as expected
- You can save a cell by either calling `save` or doing a bulk update.
- Setting `value` or `numericValue` on a cell that contains a formula will clear the formula
- Setting a `formula` value will clear the `value` and `numericValue` and after saving the values will be updated

#### `SpreadsheetCell.save(callback)`
Saves the current value or formula

#### `SpreadsheetCell.del(callback)`
Clear the cell -- internally just calls `.setValue('', callback)`

#### `SpreadsheetCell.setValue(val, callback)`
Sets the value and saves it (Just for convenience)

----------------------------------

## Further possibilities & to-do
- getting list of available spreadsheets for an authenticated user
- more authentication options

## Links
- <https://developers.google.com/google-apps/spreadsheets/>

## License
node-google-spreadsheets is free and unencumbered public domain software. For more information, see the accompanying UNLICENSE file.
