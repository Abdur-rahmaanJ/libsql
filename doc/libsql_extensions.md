# libSQL extensions

This document describes extensions to the library provided by libSQL, not available in upstream SQLite at the time of writing.

## RANDOM ROWID

Regular tables use an implicitly defined, unique, 64-bit rowid column as its primary key.
If rowid value is not specified during insertion, it's auto-generated with the following heuristics:
 1. Find the current max rowid value.
 2. If max value is less than i64::max, use the next available value
 3. If max value is i64::max:
     a. pick a random value
     b. if it's not taken, use it
     c. if it's taken, go to (a.), rinse, repeat

Based on this algorithm, the following trick can be used to trick libSQL into generating random rowid values instead of consecutive ones - simply insert a sentinel row with `rowid = i64::max`.

The newly introduced `RANDOM ROWID` option can be used to explicitly state that the table generates random rowid values on insertions, without having to insert a dummy row with special rowid value, or manually trying to generate a random unique rowid, which some user applications may find problematic.

### Usage

`RANDOM ROWID` keywords can be used during table creation, in a manner similar to its syntactic cousin, `WITHOUT ROWID`:
```sql
CREATE TABLE shopping_list(item text, quantity int) RANDOM ROWID;
```

On insertion, pseudorandom rowid values will be generated:
```sql
CREATE TABLE shopping_list(item text, quantity int) RANDOM ROWID;
INSERT INTO shopping_list(item, quantity) VALUES ('bread', 2);
INSERT INTO shopping_list(item, quantity) VALUES ('butter', 1);
.mode column
SELECT rowid, * FROM shopping_list;
rowid                item    quantity
-------------------  ------  --------
1177193729061749947  bread   2       
4433412906245401374  butter  1  
```

### Restrictions

`RANDOM ROWID` is mutually exclusive with `WITHOUT ROWID` option, and cannot be used with tables having an `AUTOINCREMENT` primary key.


## WebAssembly-based user-defined functions (experimental)

In addition to being able to define functions via the C API (http://www.sqlite.org/c3ref/create_function.html), it's possible to enable experimental support for `CREATE FUNCTION` syntax allowing users to dynamically register functions coded in WebAssembly.

Once enabled, `CREATE FUNCTION` and `DROP FUNCTION` are available in SQL. They act as syntactic sugar for managing data stored in a special internal table: `libsql_wasm_func_table(name TEXT, body TEXT)`. This table can also be inspected with regular tools - e.g. to see which functions are registered and what's their source code.

### How to enable

This feature is experimental and opt-in, and can be enabled by the following configure:
```sh
./configure --enable-wasm-runtime
```

Then, in your source code, the internal table for storing WebAssembly source code can be created via `libsql_try_initialize_wasm_func_table(sqlite3 *db)` function.

#### shell support
In order to initialize the internal WebAssembly function lookup table in libsql shell (sqlite3 binary), one can use the `.init_wasm_func_table` command. This command is safe to be called multiple times, even if the internal table already exists.

### CREATE FUNCTION

Creating a function requires providing its name and WebAssembly source code (in WebAssembly text format). The ABI for translating between WebAssembly types and libSQL types is to be standardized soon.

Example SQL:
```sql
CREATE FUNCTION IF NOT EXISTS fib LANGUAGE wasm AS '
(module 
 (type (;0;) (func (param i64) (result i64))) 
 (func $fib (type 0) (param i64) (result i64) 
 (local i64) 
 i64.const 0 
 local.set 1 
 block ;; label = @1 
 local.get 0 
 i64.const 2 
 i64.lt_u 
 br_if 0 (;@1;) 
 i64.const 0 
 local.set 1 
 loop ;; label = @2 
 local.get 0 
 i64.const -1 
 i64.add 
 call $fib 
 local.get 1 
 i64.add 
 local.set 1 
 local.get 0 
 i64.const -2 
 i64.add 
 local.tee 0 
 i64.const 1 
 i64.gt_u 
 br_if 0 (;@2;) 
 end 
 end 
 local.get 0 
 local.get 1 
 i64.add) 
 (memory (;0;) 16) 
 (global $__stack_pointer (mut i32) (i32.const 1048576)) 
 (global (;1;) i32 (i32.const 1048576)) 
 (global (;2;) i32 (i32.const 1048576)) 
 (export "memory" (memory 0)) 
 (export "fib" (func $fib)))
';
```
[1] WebAssembly source: https://github.com/psarna/libsql_bindgen/blob/55b69d8d08fc0e6e096b37467c05c5dd10398eb7/src/lib.rs#L68-L75 .

### Drop function

Dropping a dynamically created function can be done via a `DROP FUNCTION` statement.

Example:
```sql
DROP FUNCTION IF EXISTS fib;
```

### How to implement user-defined functions in WebAssembly

TODO