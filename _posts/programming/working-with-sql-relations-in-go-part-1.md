---
title: Working with SQL Relations in Go - Part 1
layout: post
index: true
createdAt: 2020-04-07T18:33
updatedAt: 2020-04-07T18:33
---
In my [last post](/programming/2019/07/13/orms-and-query-building-in-go) I
touched on the idea of using query builders for building up somewhat complex
SQL queries in Go. Well, in this post I'd like to go further, and explore how
this can be used to idiomatically build out a simple system for modelling
relationships in Go. This post going forward will assume that you have read the
aforementioned post.

* [Preface](#preface)
* [Setting the Stage](#setting-the-stage)
* [The Initial Models](#the-initial-models)
* [Abstracting Commonalities](#abstracting-commonalities)
* [Entity Stores](#entity-stores)
* [A Barebones Handler](#a-barebones-handler)
* [Implementing Custom Query Options](#implementing-custom-query-options)
* [Conclusion](#conclusion)

## Preface {#preface}

This post is split up into multiple parts simply due to how long it is going to
be. In this first part we will cover setting up the entities, models and stores
for our example application. We will also implement some simple query logic
for gracefully handling input variables that will be passed to the underlying
SQL queries.

The posts being split up into multiple parts should also aid in the digestion
of said posts.

First I'd like to cover some terminology that I will be using throughout this
article. This isn't official terminology or anything, but terms that I like to
use to aid in describing the different components of modelling the data.

* Entity - an independent type that is bespoke to the system
* Store - something that returns models from the database
* Model - a representation of an Entity's data that was either taken from the
database, or will be put in the database

This article will be covering a lot, and will be providing lot's of code
examples, all of which will be elided (denoted via `...`) for brevity.

## Setting the Stage {#setting-the-stage}

Let's assume we're building a simple blogging application with the following
entities at play,

1. User
2. Category
3. Post
4. Comment
5. Tag

where a User entity has a one-to-many relationship with Post, and Comment. The
Category entity has a one-to-many relationship with Post, and finally, the Post
entity has a one-to-many relationship with Tag, and Comment.

So let's assume we're implementing the following routes in our blogging
application,

1. `/categories`
2. `/category/{category:[0-9]+}`
3. `/posts`
4. `/posts/{post:[0-9]+}`

>**Note:** we're primarily focussed on readonly routes right now.

where `/categories`, and `/posts` return a list of the respective entities.
The `/categories` route will support the `search` and `page` query parameters,
and the `/posts` route will support the `search`, `page`, and `tag` query
parameters.

Nothing too strenuous so far, so let's move onto the next step and start
modelling our would be blogging application in Go.

## The Initial Models {#the-initial-models}

First we'll break up our entities into their top-level packages. Assume our
source tree looks something like below,

    $ ls -l
    category
    post
    user

you'll notice that we've missed out the Tag and Comment entity packages, this
is because these will both be in the Post entity package.

Next, we'll write out our structs.

    // category/category.go
    type Category struct {
        ID   int64  `db:"id"`
        Name string `db:"name"`
    }

-

    // post/post.go
    type Post struct {
        ID         int64  `db:"id"`
        UserID     int64  `db:"user_id"`
        CategoryID int64  `db:"category_id"`
        Title      string `db:"title"`
        Body       string `db:"body"`
    }

-

    // post/comment.go
    type Comment struct {
        ID     int64  `db:"id"`
        UserID int64  `db:"user_id"`
        PostID int64  `db:"post_id"`
        Body   string `db:"body"`
    }

-

    // post/tag.go
    type Tag struct {
        ID     int64  `db:"id"`
        PostID int64  `db:"post_id"`
        Name   string `db:"name"`
    }

-

    // user/user.go
    type User struct {
        ID       int64  `db:"id"`
        Email    string `db:"email"`
        Username string `db:"username"`
        Password []byte `db:"password"`
    }

We now have our entities modelled. If we think about what we want our blogging
application to do, we want to retrieve multiple models at once, filter them
based on the HTTP request data, and also retrieve individual models too.

Ideally we would also want to keep our logic for querying the database separate
from the model structs themselves, to make them easier to work with from an API
standpoint. We would also like to abstract away some of the commonalities
between our current disparate models.

## Abstracting Commonalities {#abstracting-commonalities}

What things do our models have in common with each other? Well for one they
have primary keys as denoted by the `ID` field, and they also have arbitrary
column values too.

With this in mind let's begin defining a simple `Model` interface that can 
allow us access to the model's primary key, and it's column values. We shall do
this in the `model` package.

    // model/model.go
    type Model interface {
        Primary() (string, int64)

        IsZero() bool

        Values() map[string]interface{}
    }

The `Primary` method will return two values, first a `string` which should be
the name of the column that is the primary key, and the `int64` value of the
primary key itself.

The `IsZero` method will return `bool` denoting whether the model is empty.

The `Values` method will return all of the column values, sans the primary key,
as a map of `string` to `interface{}`.

We will come back to the `Model` interface later on.

Next we should define a struct that ought to be used for handling the querying
of multiple models, or of individual models from the database.

    // model/model.go
    import (
        "database/sql"

        "github.com/andrewpillar/query"

        "github.com/jmoiron/sqlx"
    )

    type Store struct {
        *sqlx.DB
    }

    func (s Store) All(i interface{}, table string, opts ...query.Option) error {
    }

    func (s Store) Get(i interface{}, table string, opts ...query.Option) error {
    }

The two methods, `All` and `Get` have the same interface. First, they both take
an `interface{}`, which will be the destination where the queried data will be
loaded into. Next they take a `string` for the table name to query against, and
finally a variadic list of `query.Option` that will be applied to the
underlying query that is performed.

So let's implement the two methods.

    func (s Store) All(i interface{}, table string, opts ...query.Option) error {
        opts = append([]query.Option{
            query.Columns("*"),
            query.From(table),
        }, opts...)

        q := query.Select(opts...)

        err := s.Select(i, q.Build(), q.Args()...)

        if err == sql.ErrNoRows {
            err = nil
        }
        return err
    }

-

    func (s Store) Get(i interface{}, table string, opts ...query.Option) error {
        opts = append([]query.Option{
            query.Columns("*"),
            query.From(table),
        }, opts...)

        q := query.Select(opts...)

        err := s.DB.Get(i, q.Build(), q.Args()...)

        if err == sql.ErrNoRows {
            err = nil
        }
        return err
    }

The implementation themselves are similar, all that changes is the underlying
call to `sqlx.DB`.

So, we have a simple way of now querying the data, let's look at how we can
utilise this with our models we have so far.

## Entity Stores {#entity-stores}

For our blogging application we will create stores for the Category and Post
entity, since we're mainly concerned with implementing routes for them. This
will also help with keeping this article shorter as it will be covering a lot.

So let's implement stores specific for these entities.

    // category/category.go
    import (
        "blogger/model"

        "github.com/jmoiron/sqlx"
    )
    ...
    type Store struct {
        model.Store
    }

    var table = "categories"

    func NewStore(db *sqlx.DB) Store {
        return Store{
            Store: model.Store,
        }
    }

    func (s Store) All(opts ...query.Option) ([]*Category, error) {
        cc := make([]*Category, 0)

        err := s.Store.All(&cc, table, opts...)
        return cc, err
    }

    func (s Store) Get(opts ...query.Option) (*Category, error) {
        c := &Category{}

        err := s.Store.Get(c, table, opts...)
        return c, err
    }

-

    // post/post.go
    import (
        "blogger/model"

        "github.com/jmoiron/sqlx"
    )
    ...
    type Store struct {
        model.Store
    }

    var table = "posts"

    func NewStore(db *sqlx.DB) Store {
        return Store{
            Store: model.Store,
        }
    }

    func (s Store) All(opts ...query.Option) ([]*Post, error) {
        pp := make([]*Post, 0)

        err := s.Store.All(&pp, table, opts...)
        return pp, err
    }

    func (s Store) Get(opts ...query.Option) (*Post, error) {
        p := &Post{}

        err := s.Store.Get(p, table, opts...)
        return p, err
    }

Pretty east to implement. We setup a constructio function which takes an
`sqlx.DB` connection and returns a new instance of the store. Then, we
implement the `All` and `Get` methods for returning a slice of models, and a
single model respectively. The top level unexported variable is used to define
the table for the entity.

So, with the start of our models and stores in place, let's start writing a
handler for the `/categories` route.

## A Barebones Handler {#a-barebones-handler}

This handler will be incredibly simple, it will simply hit the `All` method
on the `category.Store`, and return a JSON encoded slice of the results. Nothing
too special.

    // category/handler.go
    package category

    import (
        "encoding/json"
        "net/http"

        "blogger/model"
    )

    type Handler struct {
        Store Store
    }

    func (h Handler) Index(w http.ResponseWriter, r *http.Request) {
        cc, err := h.Store.All(model.Search("name", r.URL.Query().Get("search")))

        if err != nil {
            // handle error
        }

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(cc)
    }

We have the handler for `/categories` done, now let's implement
`/categories/{category:[0-9]+}`.

    // category/handler.go
    import (
        "strconv"
    ...
        "github.com/gorilla/mux"
    )
    ...
    func (h Handler) Show(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.ParseInt(mux.Vars(r)["category"], 10, 64)

        if err != nil {
            // handle error
        }

        c, err := h.Store.Get(query.Where("id", "=", id))

        if err != nil {
            // handle error
        }

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(c)
    }

Not as simple as `Index`, but get's the job done. To finish, let's implement a
function to register the routes.

    // category/handler.go
    import (
    ...
        "github.com/jmoiron/sqlx"
    )
    ...
    func RegisterRoutes(db *sqlx.DB, r *mux.Router) {
        h := Handler{
            Store: NewStore(db),
        }

        r.HandleFunc("/categories", h.Index)
        r.HandleFunc("/categories/{category:[0-9]+}", h.Show)
    }

We've implemented the Category entity routes, now time for the Post entity
routes.

    // post/handler.go
    import (
        "encoding/json"
        "net/http"
        "strconv"

        "blogger/model"

        "github.com/gorilla/mux"

        "github.com/jmoiron/sqlx"
    )

    type Handler struct {
        Store Store
    }

    func (h Handler) Index(w http.ResponseWriter, r *http.Request) {
        q := r.URL.Query()

        pp, err := h.Store.All(
            model.Search("title", q.Get("search"),
            WhereTag(q.Get("tag")),
        )

        if err != nil {
            // handle error
        }

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(pp)
    }

    func (h Handler) Show(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.ParseInt(mux.Vars(r)["post"], 10, 64)

        if err != nil {
            // handle error
        }

        p, err := h.Store.Get(query.Where("id", "=", id))

        if err != nil {
            // handle error
        }

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(p)
    }

    func RegisterRoutes(db *sqlx.DB, r *mux.Router) {
        h := Handler{
            Store: NewStore(db),
        }

        r.HandleFunc("/posts", h.Index)
        r.HandleFunc("/posts/{post:[0-9]+}", h.Show)
    }

We have both entity handlers implemented now, however you will notice a few
things,

1. We made reference to `model.Search` in both of them, a function we have not
implemented yet.
2. The Post entity `Index` handler makes reference to a `WhereTag` function
which too doesn't exist.

So, let's implement these custom functions for the API we're building up here.

## Implementing Custom Query Options {#implementing-custom-query-options}

With these custom query options we want to gracefully handle zero-values given
to these functions. This will avoid having to pollute our handlers with checks
for against the values we're pulling in from the request.

First, let's implement `model.Search`,

    // model/model.go
    ...
    import (
        "github.com/andrewpillar/query"
    )
    ...
    func Search(col, pattern string) query.Option {
        return func(q query.Query) query.Query {
            if pattern == "" {
                return q
            }
            return query.Where(col, "LIKE", "%"+pattern+"%")(q)
        }
    }

Simple enough, we check to see if the given `pattern` is a zero-value, if it is
then we do nothing. Otherwise, we return a `WHERE LIKE` clause via a call to
`query.Where` passing it the query.

Now let's implement the `bind.WhereTag` function,

    // build/tag.go
    ...
    import (
        "github.com/andrewpillar/query"
    )
    ...
    var tagTable = "post_tags"
    ...
    func WhereTag(tag string) query.Option {
        return func(q query.Query) query.Query {
            if tag == "" {
                return q
            }
            return query.WhereQuery("id", "IN",
                query.Select(
                    query.Columns("post_id"),
                    query.From(tagTable),
                    query.Where("name", "=", tag),
                ),
            )(q)
        }
    }

Here, we return a `WHERE IN` clause which will limit the posts we select to
those that have an ID in the set returned from the sub `SELECT` query. This
sub-query is what actually uses the given `tag` parameter, and only selects the
`post_id` column from the table. Again we check for a zero-value `tag`, and do
nothing if the check passes.

So let's assume a users hits the following route,

    GET /posts?search=programming&tag=go

the following query would be performed against the `posts` table,

    SELECT * FROM posts
    WHERE (
        title LIKE '%programming%'
        AND WHERE (
            id IN (
                SELECT post_id FROM post_tags
                WHERE (name = 'go')
            )
        )
    )

And of course if no query parameters are given then it will simply perform,

    SELECT * FROM posts

## Conclusion {#conclusion}

So quite a bit has been presented here, we established the entities we would be
working with in this example application. Implemented an interface to help
abstract away the commonalities between the model entities. And started work on
the entity store implementations, which we can build upon for our query
interface.

In the next post we will look into how we can refactor what we currently have,
and handle pagination of the models.
