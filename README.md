# Shelvd reading data format, version 1

Shelvd built this JSON format to back up and restore a reader's library. We share it as a reference for a common export format between reading apps. Use any part that helps.

The schemas here are the ones Shelvd uses, as of November 2026. The example data is made up, and its identifiers are fake.

## Files

| File | One record is |
| --- | --- |
| `books.schema.json` | An edition: title, authors, identifiers, format, length |
| `entries.schema.json` | A book in the reader's library: shelf, current progress, rating, review |
| `attempts.schema.json` | One read of a book. A reread is a new attempt |
| `progress.schema.json` | One progress point: percent, page or audio position, time |
| `sessions.schema.json` | One reading session: start, end, time spent, pages |
| `notes.schema.json` | One note, attached to a progress point |
| `shelves.schema.json` | One shelf. Reading status comes from the shelf |
| `example.json` | Two books that use every record type above |

The schemas use JSON Schema draft-07. You can check a record with any draft-07 validator.

A Shelvd backup is a ZIP file. Each record type is a JSON Lines file in the ZIP, with one record on each line. `example.json` puts all the record types in one file, so you can read it from top to bottom.

## How the records connect

- A **book** is one edition of a work.
- An **entry** puts a book in the reader's library. It points to one book and one shelf.
- An **attempt** is one read of that book. It points to its entry.
- A **progress point** records where the reader was. It points to its entry and, when known, its attempt.
- A **note** points to one progress point. The progress point gives the note its place in the book and its time.
- A **session** is one sitting. It points to its entry and, when known, its attempt.

IDs are strings with no meaning of their own. They link records inside one export only. The same book can get a different ID in the next export.

## Start dates, rereads and notes

A Goodreads CSV export has no start date, only a count of rereads, and one notes field for each book. This is how our format handles each one.

**Start dates.** Each attempt has its own `startedAt` and `finishedAt`. Each one can be `null` when the date is not known. In the example, the first read of *Dom Casmurro* came from a Goodreads import. It has a finish date and no start date.

**Rereads.** Each read is a separate attempt. `sequence` gives the order: `1` is the first read, `2` is the first reread. A reread does not change the attempts before it. When a reader starts a reread, Shelvd does three things:

1. It adds a new attempt.
2. It points the entry's `currentAttemptId` to the new attempt.
3. It sets the entry's `finishedAt` back to `null`.

The first finish date stays on attempt 1.

**Notes.** A note is attached to a progress point, so it keeps its position in the book and its time. In Shelvd, a note is plain text, and a progress point has one note or none.

## Dates and unknown values

- An exact time is UTC in ISO 8601 format and ends in `Z`. Example: `2026-11-02T21:15:00Z`.
- A date with no time is `YYYY-MM-DD`. Shelvd does not add a midnight or a time zone to it.
- An unknown value is `null`. Shelvd never writes `0` for an unknown value.
- A percent is a number from 0 to 100.
- A rating has a `scale` of `5` and a `value` that keeps its precision. Example: `4.5`.

## Reading status

The entry has no status field. Status comes from the entry's shelf. Every Shelvd library has four default shelves:

| Shelf `slug` | Status |
| --- | --- |
| `want-to-read` | Not started |
| `currently-reading` | Reading now |
| `completed` | Finished |
| `set-aside` | Stopped before the end |

Readers can also make their own shelves. A book on one of these shelves has no default status. For that book, use the `status` of its current attempt: `active`, `completed` or `stopped`.

## Fields to know

- `entries.addedAt`: when the book first came into the library.
- `entries.progressPercent` and `entries.finishedAt`: the current read only.
- `entries.deletedAt` and `attempts.deletedAt`: the reader removed the record. A backup keeps removed records. Skip them if you only want the current library.
- `progress.recordedAt`: when the reader was at this position.
- `progress.syncAt`: when Shelvd saved the progress point.
- `progress.pagePosition` and `progress.measuredPageCount`: the page and the page total that the source reported. They are `null` when the source gave only a percent.
- `progress.audioPositionSeconds` and `progress.playbackSpeed`: for audiobooks.
- `sessions.durationSeconds`: time spent reading. It can be less than the time from `startedAt` to `endedAt`.
- `sessions.pagesRead`: the number the source reported. Shelvd does not calculate it from the start page and the end page.
- `startedSource`, `finishedSource` and `source`: short text that tells where a value came from. Examples: `manual`, `import_history`.

## Fields that only Shelvd uses

You can ignore these fields:

- `books.catalogId`: the book's ID in the Shelvd catalog.
- `books.provenance`: where Shelvd got each fact about the book.
- `entries.completionOverride`: a mark the reader sets in Shelvd. `1` means finished, `-1` means not finished.
- `entries.origin`: how the book came into the library. Examples: `goodreads-import`, `kindle`.
- `entries.mergedAt`: Shelvd merged this entry into another entry.

## What we would change

We built this format for backup and restore inside Shelvd. For a format that many apps share, we would change these things:

1. **Open identifiers.** `identifiers` has a fixed list of eight keys. A new catalog, such as Skoob, needs a new version of the schema. A list of `{ "scheme": "isbn13", "value": "..." }` pairs stays open.
2. **Status on the entry.** Today you must read the shelf to get the status. A `status` field on the entry is easier to read and easier to map between apps.
3. **Notes in the core.** Notes are outside the core record types in our backup. Quotes and highlights have no place yet.
4. **App fields in a separate block.** The fields that only Shelvd uses sit next to the shared fields. A separate block for each app keeps the shared part small.
5. **Room for extra data.** Our schemas refuse every field they do not list (`additionalProperties: false`). This keeps a restore strict, but another app cannot add its own data.
6. **Clear time names.** `recordedAt` and `syncAt` are easy to confuse. Names such as `readAt` and `savedAt` say more.
7. **True and false.** `shelves.isDefault` uses `1` and `0`. A JSON boolean is clearer.

## License

You can copy, change and use every file here, with no credit needed. The files are dedicated to the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/). The full text is in `LICENSE`.

Questions or ideas: open an issue in this repository, or write to hello@shelvd.io.
