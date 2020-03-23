---
title: Introducing jrnl
index: true
layout: post
createdAt: 2018-10-07T08:35
updatedAt: 2019-01-24T21:09
---
[jrnl](https://github.com/andrewpillar/jrnl) is a simple static site generator
written in Go. It takes posts written in Markdown, and converts them to HTML.
The generated HTML files are then copied to a remote destination, this can
either be a location on disk, or to a remote server. Unlike most other static
site generators jrnl does not serve the content that is generated. This post
shall serve as a brief introduction to jrnl, for more in depth usage refer to
the readme on the project's GitHub.

Below is what it looks like to use jrnl.

```
$ jrnl init blog
journal initialized, set the title with 'jrnl title'
$ cd blog
blog $ jrnl title "My Blog"
blog $ jrnl post "Penguin One, Us Zero" -c "TV Shows/The Leftovers"
```

First we initialise a new journal with `jrnl init`. If given an argument this
will create a new journal in the respectively named directory, otherwise it will
create a new journal in the current directory. We then change into the newly
created directory, and set the journal's title with `jrnl title`.

Next we create a post with `jrnl post`, giving it the title of the post as the
first, and only argument to that command. We also set the `-c` flag on the
command telling journal that this new post will belong in the given category, if
the given category does not exist then jrnl will create it. The above example
demonstrates how sub-categories can be created through jrnl simply by using a
`/` as the delimiter.

`jrnl post` will drop you into the editor you have specified in the `$EDITOR`
environment variable.

```
---
title: Penguin One, Us Zero
index: true
layout: post
createdAt: 2018-11-10T22:55
updatedAt: 2018-11-10T22:55
---
Season 1, episode 2 of The Leftovers...
```

A block of YAML, called front matter, sits at the top of the file containing
meta-data about the new post, such as the title, whether or not to index it, the
layout to use during rendering of the post, and the timestamps. Here we change
the layout from `""`, its default value, to `post`.

```
new post added tv-shows/the-leftovers/penguin-one-us-zero
blog $ jrnl remote-set me@andrewpillar.com:/var/www/andrewpillar.com
```

Once created jrnl will print out the ID of the newly created post, we can use
this later for editing, or removing the post. After this we call the
`jrnl remote-set` command to set a new remote for our journal.

```
blog $ jrnl publish
jrnl: open _layouts/post: no such file or directory
```

Finally we try and publish our journal with `jrnl publish`, however this fails
as it's trying to open a non-existent layout file to use during the rendering
of the posts. We can fix this however by creating a `post` file in the
`_layouts` directory.

```html
<!DOCTYPE HTML>
<html lang="en">
    <head>
        <meta charset="utf-8"/>
        <title>{{.Site.Title}}</title>
    </head>
    <body>{{.Post.Body}}</body>
</html>
```

Now that we have a basic layout defined we can publish without any errors.

```
blog $ jrnl publish
```

My aim with this project was to create a simple static site generator that stays
out of your way as much as possible. The above walkthrough is a very haste one,
and should hopefully get across how to use jrnl. For more thorough documentation
then please refer to the project's [readme](https://github.com/andrewpillar/jrnl).
