---
title: 'mgrt: Simple SQL migrations'
index: true
layout: post
createdAt: 2019-01-27T22:40
updatedAt: 2019-03-31T23:19
---
As mentioned in my previous [post](/2019/01/24/an-update), I have been working on a tool for handling SQL migrations. Well, that tool is now ready to debug, and is called [mgrt](https://github.com/andrewpillar/mgrt). Much like [jrnl](https://github.com/andrewpilar/jrnl), I decided to take the Welsh approach when it came to naming it.

mgrt aims to be simple and transparen in how it operates. It does not integrate with an ORM, instead it simply reads in SQL Scripts, known as revisions, and runs them against the database. Revisions in mgrt contain special directives, formatted as SQL comments, that tell mgrt about the revision, such as who authored it, how it should be run, and how it should be reset. Each revision that is performed will be logged to the database, and be given a hash to ensure that the revision cannot be run again if it has been modified.

The usage of mgrt is straight forward.

```
$ mgrt init
```

Initialise with `mgrt init`, set the author details in the `mgrt.yml` file, and begin writing revisions with `mgrt add`.

```
$ mgrt add -m "Create users table"
added new revision at:
  revisions/1136214245/up.sql
  revisions/1136214245/down.sql
```

A revision to mgrt is nothing more than a directory containing three files. A `_message` file which holds the message we gave the `mgrt add` command via `-m`, and two SQL files `up.sql`, and `down.sql`. If no message is given via the `-m` flag then mgrt will drop you into the editor given via `$EDITOR` for writing up the revision's message.

We can now write up the logic of the revision by editing these newly created SQL files.

`up.sql` file:

```sql
CREATE TABLE users (
    email    TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL
);
```

`down.sql` file:

```sql
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

**EDIT:** Yes, I nuked the unit tests, then decided to rewrite them. Nuking them in the first place was a stupid and naive thing for me to do. Also, yes I edited this fairly old blog post introducing mgrt to, well re-introduce it since I added the revision authorship feature. ~~I now deem this tool feature complete for the most part in my mind, until other people start using it -- ha~~.

**EDIT 2 Electric Boogaloo:** Now I deem this tool feature complete, for real this time, look I even [tagged](https://github.com/andrewpillar/mgrt/releases/tag/v1.0.0) it in Git. Still need to implement support for SSL connectivity. Lack of this feature isn't something that is breaking, but is a nice to have. Also, note to self: stop updating these so late at night, you make typos like a dummy...
