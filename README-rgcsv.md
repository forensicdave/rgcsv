# rgcsv

Grep a CSV without losing the header.

`rgcsv` is a small wrapper around [ripgrep](https://github.com/BurntSushi/ripgrep)
for searching data files. When the target is a CSV or TSV file it prints the
header line before the matches, and it matches on whole logical records (even
ones that span multiple lines inside quoted fields). The result is that the
output is itself valid CSV, so you can pipe it straight into a CSV tool such
as [VisiData](https://www.visidata.org/) and still see the column names:

```sh
rgcsv -i widget sales.csv | vd -f csv -
```

For any other kind of file it behaves exactly like `rg PATTERN FILE`.

To search *many* CSVs with differing schemas at once, see the sibling tool
[`rgjsonl`](README-rgjsonl.md), which emits JSONL instead.

## Install

```sh
chmod +x rgcsv
cp rgcsv ~/bin/        # or anywhere on your PATH
```

Requires bash, `rg`, and `awk` (the stock macOS/BSD awk is fine).

## Usage

```
rgcsv [rg options] PATTERN FILE
```

The file is always the **last** argument; everything before it is passed to
ripgrep unchanged, so the usual flags work:

```sh
rgcsv EMEA sales.csv                        # plain search, header kept
rgcsv -i widget sales.csv                   # case-insensitive
rgcsv -w 100 sales.csv                      # whole-word match
rgcsv -e '-5' sales.csv                     # pattern starting with a dash
rgcsv -i widget data.tsv | vd -f tsv -      # TSV works too
rgcsv error app.log                         # non-CSV: identical to plain rg
```

The field delimiter is irrelevant to `rgcsv` — only the `"` quoting rules
matter, and those are the same in `;`-delimited files — so semicolon (or any
other) CSVs need no flag here. Just tell the downstream tool:

```sh
rgcsv -i widget euro.csv | vd -f csv --csv-delimiter=';' -
```

## How it handles multi-line records

CSV records can contain newlines inside quoted fields:

```csv
id,name,notes
2,Gadget,"line one
line two"
```

ripgrep is line-based, so searching this file directly would return partial
records and break downstream parsing. `rgcsv` instead:

1. **Folds** the file into one physical line per logical record. A record is
   complete once its cumulative count of `"` characters is even (escaped
   `""` quotes add two, so they don't affect this). Embedded newlines are
   temporarily replaced with the ASCII record-separator byte (`0x1e`).
2. **Searches** the folded records with `rg`, skipping the first record
   (the header, which is printed separately — it may itself span lines).
3. **Unfolds** the matches, restoring the original newlines.

So a match *anywhere* in a record prints the *whole* record, exactly as it
appears in the file.

## Behavior notes

- **File type is detected by extension** (`.csv` / `.tsv`, case-insensitive).
  Anything else gets plain `rg` behavior — there is no content sniffing.
- **Exit codes** follow ripgrep's convention: `0` if something matched, `1`
  if nothing did (the header is still printed), `2` on usage errors.
- **The header is always printed** in CSV mode, even with zero matches, and
  is never duplicated if the pattern happens to match it.
- **Line numbers (`-n`) are not meaningful** in CSV mode: rg sees folded
  records on stdin, so numbers count body records, not file lines.
- **Patterns can't span newlines**, but they *can* match across a fold
  boundary via wildcards, since the embedded newline is a single `0x1e` byte
  during the search.
- **The `0x1e` byte is reserved**: data that genuinely contains ASCII
  record-separator bytes would be corrupted by the fold/unfold round trip.
- **Malformed files** with an unterminated quote at EOF are still searched;
  the dangling partial record is treated as complete.
