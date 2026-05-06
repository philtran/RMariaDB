# Configurable handling of `INT UNSIGNED` columns

**Date:** 2026-05-06
**Status:** Draft for implementation

## Background

R's native integer type is signed 32-bit (max `2147483647`). MySQL/MariaDB
`INT UNSIGNED` columns have range `0` to `4294967295` — values above `2^31 - 1`
silently overflow into negative numbers when returned through RMariaDB's C++
layer, which always maps `MYSQL_TYPE_LONG` to `MY_INT32`. For example,
`4294967295` is returned to R as `-1`.

This is a long-standing issue (#132, #171, #412). A prior attempt at a fix
(PR #413) silently promoted `INT UNSIGNED` to double. The maintainer (@krlmlr)
declined to merge on the grounds that not all users will want double as the
target type, and that a configurable approach is preferable.

RMariaDB already has prior art for configurable integer-type mapping: the
`bigint` argument to `dbConnect()`, introduced by @krlmlr in 2018, lets users
pick `c("integer64", "integer", "numeric", "character")` for `BIGINT` columns.
This design applies the same pattern to `INT UNSIGNED`.

## Goal

Fix the `INT UNSIGNED` overflow while letting the user choose how unsigned ints
map into R, mirroring the existing `bigint` mechanism as closely as possible.

Non-goals:

- Fixing `BIGINT UNSIGNED` overflow (values > `2^63 - 1`). Currently flows
  through `bigint`; extending `unsigned_int` to cover it would steal columns
  from existing `bigint` configurations and is better done as a follow-up PR.
- Refactoring `convert_bigint` / `convert_unsigned_int` into a single
  conversion pipeline. Possible cleanup PR after this feature lands.

## Design

### Public API

Add a new argument to `dbConnect()`:

```r
unsigned_int = c("integer64", "integer", "numeric", "character")
```

Default (via `match.arg`) is `"integer64"`, mirroring `bigint`.

### Scope

Applies only to `MYSQL_TYPE_LONG` columns with `UNSIGNED_FLAG` set
(i.e. `INT UNSIGNED`). All other types are unchanged:

- `TINYINT/SMALLINT/MEDIUMINT UNSIGNED` fit in signed int32 — unchanged,
  `MY_INT32`.
- `BIGINT` and `BIGINT UNSIGNED` continue to route through `bigint`.
- Signed `INT` continues to use `MY_INT32`.

### C++ layer

- `variable_type_from_field_type(enum_field_types type, bool binary, bool
  length1, bool is_unsigned)` — new `is_unsigned` parameter. For
  `MYSQL_TYPE_LONG + is_unsigned == true`, returns `MY_INT64` (not `MY_INT32`,
  not `MY_DBL`). All other branches unchanged.
- `MariaResultPrep.cpp` reads `fields[i].flags & UNSIGNED_FLAG` and passes it
  to `variable_type_from_field_type`.
- `MariaResult` stores a `std::vector<bool> is_unsigned_int_` parallel to
  `types_`. `is_unsigned_int_[i]` is `true` iff column `i` is
  `MYSQL_TYPE_LONG + UNSIGNED_FLAG`. Any other column (including
  `BIGINT UNSIGNED`) is `false`.
- Expose the vector to R through a dedicated C++ accessor
  (e.g. `result_is_unsigned_int(ptr)` returning a logical vector) rather than
  extending the user-facing `dbColumnInfo` output. `dbFetch_MariaDBResult`
  calls this accessor directly. Keeps the public `dbColumnInfo` schema
  unchanged, which avoids a secondary review thread about exposing new
  metadata fields.

**Why `MY_INT64` in C++ unconditionally:** the C++ layer does not read user
preferences. It produces the safe, lossless representation and lets R downcast
based on `unsigned_int`. Same strategy `bigint` uses.

### R layer

New function in `R/query.R`:

```r
convert_unsigned_int <- function(df, unsigned_int, is_unsigned_int) {
  if (unsigned_int == "integer64") return(df)
  idx <- which(is_unsigned_int)
  if (length(idx) == 0) return(df)
  as_target <- switch(unsigned_int,
    integer = as.integer,
    numeric = as.numeric,
    character = as.character
  )
  df[idx] <- suppressWarnings(lapply(df[idx], as_target))
  df
}
```

Modify `convert_bigint` to exclude unsigned-int-origin columns:

```r
convert_bigint <- function(df, bigint, is_unsigned_int = rep(FALSE, length(df))) {
  if (bigint == "integer64") return(df)
  is_int64 <- which(vlapply(df, inherits, "integer64") & !is_unsigned_int)
  # ...rest unchanged
}
```

Call both from `dbFetch_MariaDBResult` using the `is_unsigned_int` vector
obtained from the result's column metadata.

### Storage and wiring

- `MariaDBConnection` gains an `unsigned_int = "character"` slot.
- `MariaDBResult` gains an `unsigned_int = "character"` slot.
- `dbConnect_MariaDBDriver` `match.arg`s the value and stores it on the
  connection.
- `dbSend()` propagates `unsigned_int = conn@unsigned_int` into the result.

### Data flow

1. User: `dbConnect(..., unsigned_int = "numeric")`.
2. Argument validated by `match.arg`, stored on `MariaDBConnection@unsigned_int`.
3. Query: `dbSendQuery()` → `dbSend()` creates `MariaDBResult` with the same
   value.
4. C++ binds columns: for each `MYSQL_TYPE_LONG + UNSIGNED_FLAG` column, uses
   `MY_INT64` and records `is_unsigned_int_[i] = true`.
5. Fetch: column arrives in R as `integer64`.
6. `dbFetch_MariaDBResult`:
   - `ret <- convert_bigint(ret, res@bigint, is_unsigned_int)`
     — processes only `BIGINT`-origin integer64 columns.
   - `ret <- convert_unsigned_int(ret, res@unsigned_int, is_unsigned_int)`
     — processes only `INT UNSIGNED`-origin columns.
7. Each column type honors its respective user setting independently.

## Error handling

- `match.arg(unsigned_int)` catches invalid values at connection time with a
  standard error.
- `as.integer()` on values > `2^31 - 1` returns `NA` with a warning; we
  suppress the warning inside `convert_unsigned_int` (matches `convert_bigint`
  behavior). Users who choose `"integer"` opt into this.
- No new failure modes in C++.

## Testing

New tests in `tests/testthat/` (new file `test-unsigned-int.R` or extend
`test-queries.R`):

- Round-trip `INT UNSIGNED` column with boundary values `0`, `1`, `2^31-1`,
  `2^31`, `2^32-1`, `NULL`, under each of the four `unsigned_int` settings.
  Assertions:
  - `"integer64"` → class `integer64`, all values exact
  - `"numeric"` → class `numeric`, all values exact (all fit under `2^53`)
  - `"integer"` → values above `2^31-1` become `NA`, not wrapped negatives
  - `"character"` → string representation exact
- Mixed-column test: a query returning `BIGINT` + `INT UNSIGNED` with
  `bigint = "integer"` and `unsigned_int = "numeric"`, confirming each column
  is converted independently. This is the test that exercises the
  `is_unsigned_int` metadata wiring.
- Signed-INT regression: `INT` column still returns plain `integer`, unaffected
  by `unsigned_int` setting.
- Default-behavior test: without specifying `unsigned_int`, `INT UNSIGNED`
  round-trips through `integer64`.

## Compatibility / migration

- Default for `unsigned_int` is `"integer64"`. Previously, `INT UNSIGNED` came
  back as overflowing `integer`. This is a behavior change, but one that fixes
  a data-corruption bug — document as a bug fix in `NEWS.md`.
- Users who relied on the current (broken) behavior can opt back in with
  `unsigned_int = "integer"`.
- `bigint` argument is unchanged. No existing `bigint` configuration changes
  behavior.

## Files touched

**R:**

- `R/dbConnect_MariaDBDriver.R` — add arg, `match.arg`, doc, pass to `new()`.
- `R/MariaDBConnection.R` — add slot.
- `R/MariaDBResult.R` — add slot.
- `R/query.R` — add `convert_unsigned_int`; extend `convert_bigint` with
  optional `is_unsigned_int` param; update `dbSend()` to pass
  `unsigned_int = conn@unsigned_int`.
- `R/dbFetch_MariaDBResult.R` — apply both converters; obtain
  `is_unsigned_int` from the new `result_is_unsigned_int(res@ptr)` accessor.

**C++:**

- `src/MariaTypes.h` / `src/MariaTypes.cpp` — add `is_unsigned` param; return
  `MY_INT64` for unsigned `MYSQL_TYPE_LONG`.
- `src/MariaResult.h` / `.cpp` — store `is_unsigned_int_` vector; expose via
  getter.
- `src/MariaResultPrep.cpp` — compute `is_unsigned`, populate new vector.
  This is the only call site of `variable_type_from_field_type`;
  `MariaResultSimple.cpp` does not have its own type-decision path.
- `src/result.cpp` — new exported accessor `result_is_unsigned_int(ptr)`
  returning a logical vector.

**Docs / other:**

- `man/dbConnect-MariaDBDriver-method.Rd` — regenerated via `devtools::document()`.
- `NEWS.md` — entry under bug fixes / new features.

## Out of scope / future work

- `BIGINT UNSIGNED` overflow above `2^63 - 1`. Requires extending
  `unsigned_int` to also apply to `MYSQL_TYPE_LONGLONG + UNSIGNED_FLAG`, which
  overlaps with `bigint`'s current scope. Breaking change risk — separate PR.
- Consolidating `convert_bigint` + `convert_unsigned_int` into a single
  conversion function. Cleanup PR, after this feature lands.

## Why this approach

- Mirrors the `bigint` pattern, which @krlmlr himself introduced, making the
  pitch "extend the pattern you already designed to the analogous case."
- Default `"integer64"` keeps symmetry with `bigint`'s default — preserves the
  same correctness-over-ergonomics trade-off already accepted for `BIGINT`.
- Addresses the configurability concern from #413 directly.
- Narrow scope (one MySQL type), minimal risk of breaking existing
  configurations.
