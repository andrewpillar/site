---
title: 'mgrt: Simple SQL migrations'
index: true
layout: post
createdAt: 2019-01-27T22:40
updatedAt: 2019-01-27T22:40
---
As mentioned in my previous [post](/2019/01/24/an-update), I have been working on a tool for handling SQL migrations. Well, that tool is now ready to debut, and is called [mgrt](https://github.com/andrewpillar/mgrt). Much like [jrnl](https://github.com/andrewpillar/jrnl), I decided to take the Welsh approach when it can to naming it.

mgrt aims to be simple and transparent in how it operates. It does not integrate with an ORM, instead it simply reads in SQL scripts, known as revisions, and runs them against the database. Revisions in mgrt contain special directives, formatted as SQL comments, that tell mgrt about the revision, such as how it should be run, and reset. Each revision that is performed will be logged to the database, and be given a hash to ensure that the revision cannot run if it has been modified. This is a deliberate design choice, as revisions should be immutable.

The usage of mgrt is straight forward, and somewhat similar to [jrnl](https://github.com/andrewpillar/jrnl).

```
$ mgrt init
$ mgrt add -c "Create users table"
```

Once initialized we can add revisions with `mgrt add`, the `-c` flag is optional, and it there for providing a descriptive comment about the revision. When we have a new revision added we can write up the necessary SQL.

```sql
-- mgrt: revision: 1136214245: Create users table
-- mgrt: up

CREATE TABLE users (
    email    TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL
);

-- mgrt: down

DROP TABLE users;
```

Then all we need to do is configure the database connection details via the `mgrt.yml` file.

```yaml
...
type: sqlite3

address: db.sqlite
...
```

And we can kick everything off with `mgrt run`.

```
$ mgrt run
up - performed revision: 1136214245: Create users table
```

This is just a brief introduction to the tool. Of course for a more detailed guide then refer to the project [readme](https://github.com/andrewpillar/mgrt).
