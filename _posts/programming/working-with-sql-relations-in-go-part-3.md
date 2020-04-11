---
title: Working with SQL Relations in Go - Part 3
layout: post
index: true
createdAt: 2020-04-10T12:40
updatedAt: 2020-04-10T12:40
---
[Last time](/programming/2020/04/08/working-with-sql-relations-in-go-part-2), we refactored
our code, and implemented the loading of the relationships for the Post entity.
The implementation of this however was rather messy, as we did the equivalent
of sweeping everything under the rug. In this post however we will look at what
we've done, and see how this could be improved upon.

* [The Current Implementation](#the-current-implementation)
* [Refactoring the Relations](#refactoring-the-relations)
* [Abstracting the Loading](#abstracting-the-loading)
* [Binding Models](#binding-models)
* [Implementing the Abstractions](#implementing-the-abstractions)
* [Cleaning up the Implementations](#cleaning-up-the-implementations)
* [Conclusion](#conclusion)

## The Current Implementation {#the-current-implementation}

If we look at the `Show` handler for the Post entity, you'll see lot's of
repetition. First we query the tags, then we query the category, and finally the
user. Once we have all the data to hand we just cram it into an anonymous struct
and encode it to JSON. It works, but it's not ideal.

I would prefer changing the following things about the current implementation,

1. Placing the relations on the `Post` struct directly
2. Using an abstraction to handle the loading of the relationships
3. Being able to define the relations of the Post entity in code

So let's go about doing those things.

## Refactoring the Relations {#refactoring-the-relations}

All we need to do for this is simply add the necessary fields to our existing
`Post` struct,

    // post/post.go
    package post

    import (
    ...
        "blogger/category"
        "blogger/user"
    )

    type Post struct {
        ID         int64  `db:"id"`
        UserID     int64  `db:"user_id"`
        CategoryID int64  `db:"category_id"`
        Title      string `db:"title"`
        Body       string `db:"body"`

        User     *user.User         `db:"-"`
        Category *category.Category `db:"-"`
        Tags     []*Tag             `db:"-"`
    }

Then in our handler we can remove the anonymous struct entirely, and set the
relations directly on the post we have.

    // post/handler.go
    ...
    func (h Handler) Show(w http.ResponseWriter, r *http.Request) {
    ...
        p.User = u
        p.Category = c
        p.Tags = tt

        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(p)
    }

Ok, so this is the first issue solve, now let's look at implementing an
abstraction for handling the loading of the relations.

## Abstracting the Loading {#abstracting-the-loading}

Let's breakdown what we want this abstraction to do when it comes to loading
the models,

1. We want to tell the abstraction what column to query against
2. We want to tell the abstraction which values to use when querying
3. We want to tell the abstraction how to bind the loaded relations to the
given model

So let's use the Tag relation as an example, currently we do the following,

    tt, err := h.TagStore.Get(query.Where("post_id", "=", p.ID))

with our abstraction we would tell it to use the `post_id` column, and give it
`p.ID`, as the value to use. But how would we go about telling the abstraction
how to bind the model to the `Post` struct we have?

This is where our previously defined `model.Model` interface comes to play. With
this, and a callback function we should be able to achieve what we desire. So,
let's start writing out a `Loader` interface.

    // model/model.go
    ...
    type LoaderFunc func(int, Model)

    type Loader interface {
        Load(string, []interface{}, LoaderFunc) error
    }

The idea behind this abstraction is that we will call `Load` on the underlying
implementation and pass it the column, a slice of values, and a function that
will actually do the loading of the relation to the appropriate model. With our
Tag example it would look something like this,

    load := func(_ int, model.Model) {
        // bind the relation
    }

    err := h.TagStore.Load("post_id", []interface{}{p.ID}, load)

Before we proceed with implementing this on the `TagStore` however I should note
a few things. First, why does the `LoaderFunc` accept an `int` as the first
parameter? This will allow flexibility when loading in a relationship to
multiple models, let's assume we have a list of posts, and we want to load in
all of the tags for each post. Well, with this interface we could do it like so,

    pp, _ := h.Store.All()
    ids := make([]interface{}, 0, len(pp))

    for _, p := range pp {
        ids = append(ids, p.ID)
    }

    load := func(i int, m model.Model) {
        pp[i].Tags = append(pp[i].Tags, m)
    }

    h.TagStore.Load("post_id", ids, load)

The next thing I should point out, is that the `LoaderFunc` also expects
`model.Model` as the second parameter. So, in order to actually use this
abstraction we need to make sure that each model being loaded actually
implements `model.Model`.

And now the final thing to mention, is that we don't actually have a nice way
of binding models to each other via the `LoaderFunc` callback. So let's look at
how we can achieve this.

## Binding Models {#binding-models}

All of the models in our blogging application are somewhat similar, and can be
reasonably represented via the `model.Model` interface. Right now however, we
don't have a way of binding models to each other. What do I mean by binding, and
how does this differ from loading a model?

Loading a model, in this context, is what I mean once we have the models
retrieved from the database. Binding however, is what will occur when we set the
model as a relation to another model. So, with this in mind let's implement
another interface to handle the binding of models.

    // model/model.go
    type Binder interface {
        Bind(...Model)
    }

Let's also embed this interface into the `model.Model` interface itself too.
I'll come back to why this was made a separate interface later on.

    // model/model.go
    type Model Interface {
        Binder
    ...

Looking back at our Tag example from earlier, we should be able to do the
following, providing that the `post.Tag`, and `post.Post` models both implement
the `model.Model` interface.

    load := func(_ int, m model.Model) {
        p.Bind(m)
    }

    h.TagStore.Load("post_id", []interface{}{p.ID}, load)

So, let's go about implementing this for the Tag entity for the time being.

## Implementing the Abstractions {#implementing-the-abstractions}

Our implementation of the `model.Loader` for our `post.TagStore` will look
something like this,

    // post/tag.go
    ...
    import (
        "blogger/model"
    ...
    )
    ...
    var (
        _ model.Moadl  = (*Tag)(nil)
        _ model.Loader = (*TagStore)(nil)

        tagTable = "post_tags"
    )
    ...
    func (s TagStore) Load(key string, vals []interface{}, load model.LoaderFunc) error {
        tt, err := s.All(query.Where(key, "IN", vals...))

        if err != nil {
            return err
        }

        for i := range vals {
            for _, t := range tt {
                load(i, t)
            }
        }
        return nil
    }

In order for this to work however we need to make sure that `post.Tag`
implements the `model.Model` interface so we can actually pass it to the given
callback. So let's do that next,

    // post/tag.go
    ...
    func (t *Tag) Bind(mm ...model.Model) {}

    func (t *Tag) Primary() (string, int64) {
        if t == nil {
            return "id", 0
        }
        return "id", t.ID
    }

    func (t *Tag) IsZero() bool {
        return t == nil || t.ID == 0 && t.PostID == 0 && t.Name == ""
    }

    func (t *Tag) Values() map[string]interface{} {
        if t == nil {
            return map[string]interface{}{}
        }
        return map[string]interface{}{
            "post_id": t.PostID,
            "name":    t.Name,
        }
    }

Next we need to make sure that our `post.Post` model implements `model.Model`
too, so we can make use of the bind functionality.

    // post/post.go
    func (p *Post) Bind(mm ...model.Model) {
        if p == nil {
            return
        }

        for _, m := range mm {
            switch m.(type) {
            case *user.User:
                p.User = m.(*user.User)
            case *category.Category:
                p.Category = m.(*category.Category)
            case *Tag:
                p.Tags = append(p.Tags, m.(*Tag))
            }
        }
    }

    func (p *Post) Primary() (string, int64) {
        if t == nil {
            return "id", 0
        }
        return "id", p.ID
    }

    func (p *Post) Values() map[string]interface{} {
        if p == nil {
            return map[string]interface{}{}
        }
        return map[string]interface{}{
            "user_id":     p.UserID,
            "category_id": p.CategoryID,
            "title":       p.Title,
            "body":        p.Body,
        }
    }

You'll notice that in the implementation of `Bind` we do a type assert on the
given model to see if its one of the relations a Post could have.

Now in our `handler.Show` method we should be able to do the following,

    // post/handler.go
    ...
    p, err := h.Store.Get(query.Where("id", "=", id))

    if err != nil {
        // handle error
    }

    load := func(_ int, m model.Model) {
        p.Bind(m)
    }

    h.TagStore.Load("post_id", []interface{}{p.ID}, load))
    ...

Let's assume we go ahead and implement the `model.Loader` interface for
`user.Store`, and `category.Store`, then we could refactor our handler to be
something like this,

    // post/handler.go
    ...
    p, err := h.Store.Get(query.Where("id", "=", id))

    if err != nil {
        // handle error
    }

    load := func(_ int, m model.Model) {
        p.Bind(m)
    }

    h.TagStore.Load("post_id", []interface{}{p.ID}, load)
    h.UserStore.Load("id", []interface{}{p.UserID}, load)
    h.CategoryStore.Load("id", []interface{}{p.CategoryID}, load)
    ...

This is better than what we had before, but still not very ideal. There's a few
things I'd like to change about this,

1. Dynamically extract the values from the given model based on the key we want
2. Specify how we want the binding to happen based on the foreign key and
primary key of the models

So let's go about doing both of these things.

## Cleaning up the Implementations {#cleaning-up-the-implementations}

It would be nice if we could simply extract the values we want from the model
based on the key of that value, instead of having to clumsily write out
`[]interface{}{p.ID}`. Thankfully we know that our models implement the
`model.Model` interface, which gives us access to the `Primary` and `Values`
methods, so let's set about doing this.

    // model/model.go
    ...
    func MapKey(key string, mm ...Model) []interface{} {
        vals := make([]interface{}, 0, len(mm))

        for _, m := range mm {
            if col, val := m.Primary(); key == col {
                vals = append(vals, val)
                continue
            }

            if val, ok := m.Values()[key]; ok {
                vals = append(vals, val)
            }
        }
        return vals
    }

>**Note:** We check the given key against whatever is returned from `Primary`
first, since the primary key is omitted from the map returned from `Values`.

With the `model.MapKey` function in place we can now specify the values we want
from the given models, based off the value's name like so,

    h.TagStore.Load("post_id", model.MapKey("id", p), load)

Now let's look at specifying how we want the binding to happen based on the
foreign key, and the primary key of the models. If we look at what's happening
during loading, we can safely assume it's going to be the same across all of
our models. Because of this, it would be smart to have a generic function to
handle the binding of related models. Let's see how this could be done.

    // model/model.go
    ...
    func getInt64(key string, m Model) int64 {
        if col, val := m.Primary(); key == col {
            return val
        }
        i, _ := m.Values()[key].(int64)
        return i
    }

    func Bind(a, b string, mm ...Model) LoaderFunc {
        return func(i int, r Model) {
            if i > len(mm) || len(mm) == 0 {
                return
            }

            m := mm[i]
            if getInt64(a, m) == getInt64(b, m) {
                m.Bind(r)
            }
        }
    }

There's quite a bit going on here so let me explain. First, we implement a
`getInt64` function that will return the underlying `int64` type of the given
key. This is working off the assumption that foreign, and primary keys are both
64-bit integers.

Next, we implement the `Bind` function itself, which returns a `LoaderFunc`. The
`Bind` function takes two strings, followed by a variadic list of `Model`
interfaces. The two strings `a`, and `b`, represent the two keys we want to pull
from the model on which binding will occur, and the model which will be bound
respectively.

Within the returned `LoaderFunc` we first check to see if the model exists in
the variadic list based on the given index. Then we actually do the comparison
of keys. Here we assume that the keys being compared are both `int64`, and if
they match then we bind the model `r` to the model `m`.

This seems confusing at first, so let's demonstrate what this would look like
with our Tag example.

    h.TagStore("post_id", model.MapKey("id", p), model.Bind("id", "post_id", p))

And if we were to expand the `model.Bind` function call it would look something
like this,

    h.TagStore("post_id", model.MapKey("id", p), func(i int, r model.Model) {
        if i > len(mm) || len(mm) == 0 {
            return
        }

        m := mm[i]
        if getInt64("id", m) == getInt64("post_id", r) {
            m.Bind(r)
        }
    })

So with these new functions in place let's look at our relationship loading now.

    h.TagStore.Load("post_id", model.MapKey("id", p), model.Bind("id", "post_id", p))
    h.UserStore.Load("id", model.MapKey("user_id", p), model.Bind("user_id", "id", p)))
    h.CategoryStore.Load("id", model.MapKey("category_id", p), model.Bind("category_id", "id", p))

We're slowly improving, but I'm still not satisfied with what we have.

## Conclusion {#conclusion}

We managed to make quite a few improvements here when it comes to loading in
the relationships for a model. However I still think some things could be
improved upon, specifically the defining the entity relationships themselves in
the code.

In the next post we will explore how we can go about utilising more function
callbacks to achieve this.

>**Note:** I know in the previous post I said we would look at defining
relationships idiomatically in code, however this post is already rather long.

I know that some of the things mentioned here perhaps seem like they were
plucked out of thin air. But I do intend on writing a summary post at the end
of this series explaining how I came about these implementations, and why I felt
the need on writing this series in the first place.
