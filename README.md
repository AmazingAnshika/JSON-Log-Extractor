# JSON Log Extractor

Converts a JSON-based log file into **columnar format** — one file per field.

## The problem

Logs are written as one JSON document per line. Reading a single field means parsing
every line in full, which is slow when files are 4–48 GB. Splitting the data into one
file per field lets a reader touch only the columns it cares about.

Given this input:

```
{"timestamp": "2022-01-14T00:12:21.000", "Field1": 10, "Field_Doc": {"f1": "xyz"}}
{"timestamp": "2022-01-18T00:15:51.000", "Field_Doc": {"f1": "abc", "f2": 1.7}}
```

the tool produces four files in `output_columns/`:

| File | Contents |
| --- | --- |
| `timestamp.column` | `2022-01-14T00:12:21.000`, `2022-01-18T00:15:51.000` |
| `Field1.column` | `10` |
| `Field_Doc.f1.column` | `xyz`, `abc` |
| `Field_Doc.f2.column` | `1.7` |

Nested objects are flattened into dotted names (`Field_Doc.f1`), recursing to any depth.
Field names are discovered from the data, so nothing has to be known in advance.

### Column file format

- One value per line.
- Strings are written as-is; values never contain newlines.
- `null` and empty strings are written as an empty line.

## Build and run

Requires a C++11 compiler. [RapidJSON](https://rapidjson.org/) is vendored in
`rapidjson/`, so there is nothing to install.

```bash
g++ -std=c++11 -O2 -pthread newa.cpp -o extractor
./extractor
```

Both commands print nothing on success. Results land in `output_columns/`:

```bash
ls output_columns/
cat output_columns/timestamp.column
```

Delete `output_columns/` between runs — the program opens column files in append mode,
so a second run stacks new values underneath the old ones.

## Layout

| Path | Purpose |
| --- | --- |
| `newa.cpp` | The working implementation. Reads all lines, splits them across 4 threads. |
| `a.cpp` | Earlier attempt. All threads share one file handle and rewind it, so they race over the same data. |
| `1111.cpp` | Windows variant using `windows.h` and `CRITICAL_SECTION`. Does not build on macOS or Linux. |
| `dummy.log` | Two-line sample from the spec. |
| `dummy1.log` | Nine-line sample with deeper nesting, nulls, and empty strings. |
| `rapidjson/` | Vendored JSON parser. |

The input path and output directory are hardcoded in `main()` in `newa.cpp`; change them
there and recompile to process a different file.

## Known limitations

- **Rows are not aligned.** A field missing from a line is skipped rather than written as
  a blank line, so column files end up with different lengths and a row can no longer be
  reassembled by line number. Visible with `dummy1.log`.
- **Writes are not synchronized.** A mutex is declared but never locked, so threads
  appending to the same column file can interleave, and concurrent appends mean output
  order does not follow input order.
- **The whole file is read into memory** before processing, which does not hold for the
  4–48 GB target. A streaming single pass is needed instead.
- One `ofstream` is opened and closed per value, making syscalls the dominant cost.
