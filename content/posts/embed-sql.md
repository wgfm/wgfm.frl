---
title: "Embedding SQL queries in Go"
author: "Werner Hofstra"
date: 2021-11-12T21:33:23+01:00
draft: false
---

Go 1.16 introduced the [embed](https://pkg.go.dev/embed) package to the
standard library. This library allows us to embed static content within your
compiled executable program. This is amazing, because we only have to ship one
binary, even when we want to include an HTML page, for example.[^1]

One application for embedding that I personally find very exciting, is SQL.
Call me old fashioned, but I love me some plain old SQL queries. I think
they're easier to read, reason about, debug, and maintain than anything an ORM
or SQL statement builder library can provide.

Without embedding, we could use string literals to represent queries:

```go
package store

const getAsset = `
  SELECT
    a.id,
    a.publication_reference,
    a.purpose,
    mt.media_type
  FROM assets a
  JOIN media_types mt ON a.media_type_id = mt.id
  WHERE publication_reference = $1;
`
```

Unfortunately, many editors will not recognize the contents of the `getAsset`
string as SQL code. As such, the string will be shown as a string, without any
syntax highlighting. Gofmt will not format the query either. On top of that, to
make the query line out nicely, we may want to add some padding to the query.
How much padding? The answer will depend on your team's styling conventions, or
even on the opinion of the individual writing the code.

Wouldn't it be nice to move these queries to their own files? I'd like to think
so, and Go provides us a great way to do just that. This is how I currently
embed SQL queries:

```sql
-- path/to/project/store/sql/get_asset.sql
SELECT
  a.id,
  a.publication_reference,
  a.purpose,
  mt.media_type
FROM assets a
JOIN media_types mt ON a.media_type_id = mt.id
WHERE publication_reference = $1;
```

```go
// path/to/project/store/queries.go
package store

import _ "embed"

//go:embed sql/get_asset.sql
var getAsset string
```

The embedded file in the `//go:embed` directive is always relative to the
location of the source file that contains the directive. In other words, if our
embedding Go file resides at `store/queries.go` and we have a directive such as
`//go:embed sql/get_asset.sql`, the Go compiler will try to read a file at
`store/sql/get_asset.sql`.

Embedding SQL queries has a couple of benefits. The first, and perhaps most
obvious, is that storing SQL queries in `.sql` files allows your editor to
apply syntax highlighting, formatting[^2], and all of that other cool stuff
editors do when they know the type of file they're editing. When a query is
hidden in a string in a `.go` file, there's no easy way for the editor to know
that the string is in fact an SQL query.

Another emergent benefit of embedding is that it becomes very easy to find out
which tables and columns are used where. Recently, I was cleaning up some dead
code. There was one column, `purpose`, which had served its purpose (pun
intended). Its value could easily be derived, so it was due for removal.

If my queries wouldn't have been stored in `.sql` files, I would have had to
grep the entire codebase for `purpose`, and I would have had to deal with a lot
of noise from the surrounding Go code, which was still using the concept of
'purpose'.

Instead, since my queries were stored in `.sql` files, finding all usages of
this column was as easy as:

```bash
find . -name '*.sql' | xargs grep purpose
```

As with anything in the field of software engineering, embedding SQL files is
not a silver bullet. Sometimes you need to dynamically construct queries.
Sometimes your queries are so simple, it doesn't make sense to put them in
separate files.

It has, however, made our code base more consistent. Separating Go from SQL
is already paying dividends.

[^1]: I have to mention that there were some open source packages that offered
  similar functionality, such as [statik](https://github.com/rakyll/statik),
  before Go 1.16.

[^2]: Despite being one of the oldest popular languages, it is surprisingly
  hard to find an SQL formatter that works well across multiple editors.
