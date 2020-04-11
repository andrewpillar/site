---
title: Working with SQL Relations in Go - Part 2
layout: post
index: true
createdAt: 2020-04-08T16:37
updatedAt: 2020-04-08T16:37
---
In the [previous post](/programming/2020/04/07/working-with-sql-relations-in-go-part-1)
we setup the entity models for the blogging application, and built some custom
query options to handle the data we would be receiving via an HTTP request. Here
we will go about implementing pagination for our models, so we can support the
`page` query parameter, look into how we can refactor what we have in our
HTTP handlers, and start looking into loading entity relationships for models.

* [Handling Pagination](#handling-pagination)
* [Refactoring the Handlers](#refactoring-the-handlers)
* [Loading the Post Tags](#loading-the-post-tags)
* [Loading the Rest of the Post Relations](#loading-the-rest-of-the-post-relations)
* [Conclusion](#conclusion)

## Handling Pagination {#handling-pagination}

When it comes to paginating the models we would need the following information
in regards to the set of models we're working with,

1. The next and previous page numbers
2. The current page
3. The offset

With this information in mind let's implement a `Paginate` method on
`model.Store`.

    // model/model.go
    ...
    type Paginator struct {
        Next   int64
        Prev   int64
        Offset int64
        Page   int64
    }

    var PageLimit int64 = 25

    func (s Store) Paginate(table string, page int64, opts ...query.Option) (Paginator, error) {
        p := Paginator{
            Page: page,
        }

        opts = append([]query.Option{
            query.Count("*"),
            query.From(table),
        }, opts...)

        q := query.Select(opts...)

        stmt, err := s.Prepare(q.Build())

        if err != nil {
            return p, err
        }

        defer stmt.Close()

        var count int64

        if err := stmt.QueryRow(q.Args()...).Scan(&count)); err != nil {
            return p, err
        }

        pages := (count / PageLimit) + 1

        p.Offset = (page - 1) * PageLimit
        p.Next = page + 1
        p.Prev = page - 1

        if p.Prev < 1 {
            p.Prev = 1
        }
        if p.Next > pages {
            p.Next = pages
        }
        return p, nil
    }

Here we define a new `Paginator` type that is returned from the `Paginate`
method. This will contain all of the necessary information that is needed to do
a subsequent `SELECT *` on the table again, this time ensuring only a subset of
the results are returned, and at the correct offset.

This method is pretty simple, we simply rely on the count returned via `SELECT
COUNT(*)` to figure out everything else. We calculate the maximum number of
pages with `(count / PageLimit) + 1`, get the next, and previous page by
incrementing, and decrementing the current page respectively. Finally, we check
for underflow/overflow errors of the next and previous page, then return the
`Paginator`.

Now let's do what we did with `store.Get` and `store.All`, and wrap the
`store.Paginate` method for the necessary entity stores.

    // category/category.go
    ...
    func (s Store) Paginate(page int64, opts ...query.Option) (model.Paginator, error) {
        return s.Store.Paginate(table, page, opts...)
    }

-

    // post/post.go
    func (s Store) Paginate(page int64, opts ...query.Option) (model.Paginator, error) {
        return s.Store.Paginate(table, page, opts...)
    }

Now let's move onto refactoring what we have for handling the respective entity
routes.

## Refactoring the Handlers {#refactoring-the-handlers}

In the previous post, we implemented a `handler.Index` method for the Category
and Post entity. They both passthrough data from the HTTP request via
`r.URL.Query()`, to the relevant query options we implemented.

We now need to do a similar thing only for pagination. This will require having
to make a call to `store.Paginate` beforehand, and modifying the query to use
`query.Offset` to offset the records we query.

Now, we could do this all in the handler, however we will move this logic into
the entity's store instead, and create another method on that store called
`Index`. I'll explain why after.

    // category/category.go
    package category

    import (
    ...
        "net/url"
        "strconv"
    ...
    )
    ...
    func (s Store) Index(vals url.Values) ([]*Category, model.Paginator, error) {
        page, err := strconv.ParseInt(vals.Get("page"), 10, 64)

        if err != nil {
            page = 1
        }

        opts := []query.Option{
            model.Search("name", vals.Get("search"),
        }

        paginator, err := s.Paginate(page, opts...)

        if err != nil {
            return []*Category{}, err
        }

        cc, err := s.All(append(
            opts,
            query.Limit(model.PageLimit),
            query.Offset(paginator.Offset),
        )...)
        return cc, paginator, err
    }

-

    // post/post.go
    package post

    import (
    ...
        "net/url"   
        "strconv"
    ...
    )
    ...
    func (s Store) Index(vals url.Values) ([]*Post, model.Paginator, error) {
        page, err := strconv.ParseInt(vals.Get("page"), 10, 64)

        if err != nil {
            page = 1
        }

        opts := []query.Option{
            model.Search("title", vals.Get("search"),
            WhereTag(vals.Get("tag")),
        }

        paginator, err := s.Paginate(page, opts...)

        if err != nil {
            return []*Post{}, err
        }

        pp, err := s.All(append(
            opts,
            query.Limit(model.PageLimit),
            query.Offset(paginator.Offset),
        )...)
        return pp, paginator, err
    }

So, the reason we do this is so we can remove as much of the data handling
logic from the HTTP handlers themselves. Ideally we just want to say to the
store "Here's the data for the models I want, take what you need." Now in our
handlers we can just call `store.Index`, and pass through `r.URL.Query()`. This
new method will also return the `model.Paginator` itself, so we can send the
pagination data as part of the response.

    // category/category.go
    ...
    func (h Category) Index(w http.ResponseWriter, r *http.Request) {
        cc, paginator, err := h.Store.Index(r.URL.Query())

        if err != nil {
            // handle error
        }

        data := struct{
            Pages struct{
                Next int64
                Prev int64
            }
            Data []*Category
        }{}
        data.Pages.Next = paginator.Next
        data.Pages.Prev = paginator.Prev
        data.Data = cc

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(data)
    }

-

    // post/post.go
    ...
    func (h Post) Index(w http.ResponseWriter, r *http.Request) {
        pp, paginator, err := h.Store.Index(r.URL.Query())

        if err != nil {
            // handle error
        }

        data := struct{
            Pages struct{
                Next int64
                Prev int64
            }
            Data []*Category
        }{}
        data.Pages.Next = paginator.Next
        data.Pages.Prev = paginator.Prev
        data.Data = pp

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(data)
    }

We've updated the handlers to now invoke the `Index` method on the respective
store. With this in place we can now update the response we send back with the
necessary pagination data, such as the next, and previous page.

## Loading the Post Tags {#loading-the-post-tags}

So far we've looked at querying the models, and handling the different
parameters that could be given to filter the results. This is all very well and
good, however we haven't touched on how we're going to start working with the
entity relationships.

We've already established some of the relationships at play, for example we
know that a Post has a one-to-many relationship with Tag. So, let's see how we
can load in the necessary tag models when viewing an individual post.

First, let's implement a store for the Tag entity.

    // post/tag.go
    package post

    import (
    ...
        "blogger/model"

        "github.com/andrewpillar/query"

        "github.com/jmoiron/sqlx"
    )
    ...
    type TagStore struct {
        model.Store
    }

    func NewTagStore(db *sqlx.DB) TagStore {
        return TagStore{
            Store: model.Store{DB: db},
        }
    }

    func (s TagStore) All(opts ...query.Option) {
        tt := make([]*Tag, 0)

        err := s.Store.All(&tt, tagTable, opts...)
        return tt, err
    }

With this new store implemented, we can add it as a dependency to the
`post.Handler`.

    // post/handler.go
    ...
    type Handler struct {
        Store    Store
        TagStore TagStore
    }
    ...
    func RegisterRoutes(db *sqlx.DB, r *mux.Router) {
        h := Handler{
            Store:    NewStore(db),
            TagStore: NewTagStore(db),
        }
    ...

When querying the `post_tags` table we only want to get all of the tags for
the current post we're viewing. A trivial SQL query,

    SELECT * FROM post_tags WHERE (post_id = ?)

where the `?` is the ID of the post we're viewing. Let's go ahead and implement
this for the `Show` method.

    // post/handler.go
    package post

    import (
    ...
        "github.com/andrewpillar/query"
    ...
    )
    ...
    func (h Handler) Show(w http.ResponseWriter, r *http.Request) {
    ...
        tt, err := h.TagStore.All(query.Where("post_id", "=", p.ID))

        if err != nil {
            // handle error
        }

        data := struct{
            Post *Post
            Tags []*Tag
        }{}
        data.Post = p
        data.Tags = tt

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(data)
    }

## Loading the Rest of the Post Relations {#loading-the-rest-of-the-post-relations}

One relation has been loaded for the post, but we're missing two more, the
relationship with the Category, and User entities. Let's implement the loading
of that too.

First we'll add the dependency to the `category.Store`.

    // post/handler.go
    import (
    ...
        "blogger/category"
    ...
    )

    type Handler struct {
        Store         Store
        TagStore      TagStore
        CategoryStore category.Store
    }
    ...
    func RegisterRoutes(db *sqlx.DB, r *mux.Router) {
        h := Handler{
            Store:         NewStore(db),
            TagStore:      NewTagStore(db),
            CategoryStore: category.NewStore(db),
        }
    ...

I'm not going to show the implementation of the `category.Store`, as it is
similar to the other entity stores.

Now we can simply load in the category we need.

    // post/handler.go
    ...
    func (h Handler) Show(w http.ResponseWriter, r *http.Request) {
    ...
        c, err := h.CategoryStore.Get(query.Where("id", "=", p.CategoryID))

        if err != nil {
            // handle error
        }

        data := struct{
            Post     *Post
            Category *category.Category
            Tags     []*Tag
        }{}
        data.Post = p
        data.Category = c
        data.Tags = tt

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(data)
    }

Next we'll do the same with the User entity, again the code for the store
implementation has been elided, along with the logic for hooking it up to the
handler.

    // post/handler.go
    ...
    func (h Handler) Show(w http.ResponseWriter, r *http.Request) {
    ...
        u, err := h.UserStore.Get(query.Where("id", "=", p.UserID))

        if err != nil {
            // handle error
        }

        data := struct{
            Post     *Post
            User     *user.User
            Category *category.Category
            Tags     []*Tag
        }{}
        data.Post = p
        data.User = u
        data.Category = c
        data.Tags = tt

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(data)
    }

## Conclusion {#conclusion}

In retrospect we didn't cover much here, though the additional code may make it
seem like we have. I know this post, and the prior have approached this with a
fast pace, this is primarily because it assumes you're already familiar with the
approach of query building I mentioned
[here](/programming/2019/07/13/orms-and-query-building-in-go).

You'll notice that when implementing the relationship loading for the Post
entity we basically dumped all the logic in the HTTP handler. Not an ideal place
to put it, however this will be refactored in the next post, in which we will
cover the following,

1. Defining relations between entities idiomatically
2. Loading entity relations
3. Binding relations to models and stores

And how this can be achieved with interfaces and first class functions.
