# [vertica-query-analyser](https://github.com/lukas-mi/vertica-query-analyser)

SQL analyser for Vertica. A wrapper of [uber/queryparser](https://github.com/uber/queryparser).

Features:
* Provides the following information (check out [uber/queryparser](https://github.com/uber/queryparser]) for more detail):
    * What columns were accessed and in what clauses (`SORT`, `GROUP`, etc.)
    * What are the join columns (`JOIN ON`)
    * What table were accessed
    * Table lineage
* Built in catalog usage
* Simple json output

Build:
* `stack setup`
* `stack build`

Docker:
* `docker pull lukasmi/vertica-query-parser:latest` ([lukasmi/vertica-query-analyser](https://hub.docker.com/r/lukasmi/vertica-query-analyser))

Usage:
* `vq-analyser catalog_file [query_file]` - specify catalog file and optionally query file, if query file is not specified then stdin is used.
* `vq-analyser catalog_file -d directory` - specify catalog file and directory in which all `*.sql` files will be analysed. Will create files after the original ones: `<original>.sql.json` on analysis success and `<original>.sql.txt` on failure.

Examples:

`catalog.sql`:
```sql
CREATE SCHEMA demo;
CREATE TABLE demo.foo (a INT, b INT);
CREATE TABLE demo.bar AS SELECT * FROM demo.foo;
```
Command: `vq-analyser catalog.sql queries.sql` or `docker run -i -v /host/path:/container/path /container/path/catalog.sql /container/path/queries.sql`
1. Column resolving
    * `queries.sql`:
    ```sql
    SELECT * FROM demo.bar WHERE a IS NOT NULL ORDER BY a;
    
    ```
    * output: 
    ```json
    [
      {
        "statement": "SELECT",
        "lineage": [],
        "columns": [
          {
            "clause": "ORDER",
            "column": "demo.bar.a"
          },
          {
            "clause": "SELECT",
            "column": "demo.bar.a"
          },
          {
            "clause": "WHERE",
            "column": "demo.bar.a"
          },
          {
            "clause": "SELECT",
            "column": "demo.bar.b"
          }
        ],
        "tables": [
          "demo.bar"
        ],
        "joins": []
      }
    ]
    ```
2. Joins
    * `queries.sql`:
    ```sql
    SELECT foo.a, foo.b
    FROM demo.foo foo
    JOIN demo.bar bar
    ON foo.a = bar.a;
    ```
    * output:
    ```json
    [
      {
        "statement": "SELECT",
        "lineage": [],
        "columns": [
          {
            "clause": "JOIN",
            "column": "demo.bar.a"
          },
          {
            "clause": "JOIN",
            "column": "demo.foo.a"
          },
          {
            "clause": "SELECT",
            "column": "demo.foo.a"
          },
          {
            "clause": "SELECT",
            "column": "demo.foo.b"
          }
        ],
        "tables": [
          "demo.bar",
          "demo.foo"
        ],
        "joins": [
          {
            "left": "demo.bar.a",
            "right": "demo.foo.a"
          }
        ]
      }
    ]
    ```
3. Lineage
    * `queries.sql`:
    ```sql
    CREATE TABLE demo.baz AS
        SELECT foo.a, foo.b
        FROM demo.foo foo
        JOIN demo.bar bar
        ON foo.a = bar.a;
    ```
    * output
    ```json
    [
      {
        "statement": "CREATE_TABLE",
        "lineage": [
          {
            "decedent": "demo.baz",
            "ancestors": [
              "demo.bar",
              "demo.foo"
            ]
          }
        ],
        "columns": [
          {
            "clause": "JOIN",
            "column": "demo.bar.a"
          },
          {
            "clause": "JOIN",
            "column": "demo.foo.a"
          },
          {
            "clause": "SELECT",
            "column": "demo.foo.a"
          },
          {
            "clause": "SELECT",
            "column": "demo.foo.b"
          }
        ],
        "tables": [
          "demo.bar",
          "demo.baz",
          "demo.foo"
        ],
        "joins": [
          {
            "left": "demo.bar.a",
            "right": "demo.foo.a"
          }
        ]
      }
    ]
    ```
