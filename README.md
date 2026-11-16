# Verona CSV

A UTF-8 comma-separated field parser and writer written in Verona, using Base
and Foundation allocators and byte buffers.

## Build and test

From this package directory, using the repository toolchain:

```sh
../../build/verona build csv ../../build/formats
../../build/verona build csv-tests ../../build/formats
../../build/formats/csv-tests
```

Consumers add `src`, `../base/src`, and `../foundation/src` to their module
paths and import `csv`. The static archive is a build artifact; Verona
consumers use source modules. Tests also use `../testing/src`.

## Shared contracts

- `result` contains `status: error:status` and `offset: u64`. Success is `(ok)`
  with offset equal to the input byte length. Syntax errors are `(invalid-input)`
  with a zero-based byte offset (EOF may equal length). Invalid UTF-8 is detected
  before parsing and reports offset zero; INI's NUL precheck also reports zero.
- Input is a borrowed `data:view`, not a NUL-terminated string. The caller keeps
  its bytes alive and unchanged throughout parsing and callbacks.
- `visitor` carries an opaque `(pointer void)` context and a typed callback.
  There is no global parser state. Callbacks return `unit`; cancellation is not
  supported. Events already delivered are not rolled back on a later error.
- Writers append to an initialized `data:data`, using its chosen allocator.
  They validate input and reserve a checked worst-case capacity before appending,
  so validation, overflow, or allocation failure preserves existing output.
  Source bytes must not alias output storage. Check every returned status.
- No implicit file I/O, document ownership, or reference counting. Arena reset
  or destruction invalidates all associated buffers. The algorithms use
  recursion; unbounded input lengths are not a stack-safety guarantee.

## API

| Export | Contract |
| --- | --- |
| `visitor(context, emit)` | `emit(context, row: u64, column: u64, field: data:view) -> unit` |
| `parse(source, allocator:backend, visitor)` | Decode fields using one reusable scratch buffer; returns `result` |
| `write-field(source, out)` | Append a quoted field, doubling embedded quotes; returns `error:status` |
| `write-separator(out)` | Append a comma; returns `error:status` |
| `write-record-end(out)` | Append CRLF; returns `error:status` |

Rows and columns start at zero. **Field views are valid only during the callback**;
copy them to retain values. Scratch storage is released through the supplied
allocator on success and every error, including failed growth. With an arena,
release is a no-op and the arena retains that storage until reset/destruction.
Allocation failures retain their `allocation-error` cause and report the byte
being processed. Memory use follows the largest decoded field, not the document.

Comma is the only separator. LF and CRLF terminate records; a bare CR outside
quotes is rejected. A quoted field may contain commas, CR/LF, and doubled quotes.
Quotes must enclose the entire field; whitespace after a closing quote is an
error. Unquoted spaces are preserved. UTF-8 (including embedded NUL) is accepted;
there is no BOM stripping or type inference.

An empty input has zero records. A blank line is one empty field. A trailing comma
creates a final empty field; a final record newline creates no extra record.
Ragged rows are accepted and headers are ordinary fields. Callers enforce shape
or header rules as needed. The parser takes a complete memory view; incremental
chunk input and table/DOM construction are outside this initial API.

Writers always quote fields. Add separators between fields and a record ending
after the last field, checking each status. Individual operations preserve output
on failure; a sequence of operations is not a transaction.

## Example

```lisp
(import base)
(import csv)
(import data)
(import allocator)

(%function on-field
  ((context (pointer void)) (row u64) (column u64) (value data:view)) unit
  unit)

(%function inspect-csv ((backend allocator:backend)) csv:result
  (csv:parse (data:view "name,language\nbruno,verona\n" 27) backend
    (csv:visitor (cast (pointer void) "") on-field)))
```

Tests cover quoted delimiters/newlines/quotes, empty and trailing fields, Unicode,
row/column coordinates, malformed syntax, invalid UTF-8, system and arena
allocation, and allocation failure with scratch cleanup and preserved output.
