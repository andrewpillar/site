---
title: Working with SQL Relations in Go - Part 4
layout: post
index: true
createdAt: 2020-04-11T18:35
updatedAt: 2020-04-11T18:35
---
[Previously](/programming/2020/04/10/working-with-sql-relations-in-go-part-3), we
implemented some new interfaces, and utility functions to aid in the loading and
binding of the model relations. This was a much better improvement than what we
had before, however we still don't have a way of defining the relations
themselves in code. So far, we have heavily utilised callback functions to do
most of the lifting for us. For example, the `model.Bind` function we
implemented returns a callback that will gracefully handle binding the models we
load. In this post we will look at how we can utilise callbacks to set the
relations between models via code.

* [Fleshing out an API](#fleshing-out-an-api)
* [Implementing the API](#implementing-the-api)
* [Utilising the Binder Interface](#utilising-the-binder-interface)
* [Conclusion](#conclusion)

## Fleshing out an API {#fleshing-out-an-api}

Let's think about what we currently have for loading in our Post relationships,
and what an ideal API for it would look like. Right now we have,

    h.TagStore.Load("post_id", model.MapKey("id", p), model.Bind("id", "post_id", p))
    h.UserStore.Load("id", model.MapKey("user_id", p), model.Bind("user_id", "id", p)))
    h.CategoryStore.Load("id", model.MapKey("category_id", p), model.Bind("category_id", "id", p))

however I would prefer something like this,

    LoadRelations(p)

a simple function that will take a variadic list of the Post models we have,
and load in all of the possible relations for this post. This looks good, but
how will this function know which loader to use in order to load which relation?

Perhaps we could pass it a map, where each key in the map is the name of the
relation to load, and the value is the `model.Loader` to use. Then, under the
hood it would simple iterate over that map, and invoke the `Load` method on
each `model.Loader` available. This seems like an ideal way of doing things, so
perhaps the API should look something like this,

    LoadRelations(loaders, p)

This also allows us to only load in certain relations depending on which
`model.Loader` we put in this map.

This solves one aspect of the problem, but what about actually defining the
relationships the Post entity will have? Ideally I would like to have a single
unexported package level variable in the `post` package that describes the
relationships the entity has. This could then be used to figure out how the
model binding will occur, something like,

    relations = map[string][]string{
        "user:      []string{"user_id", "id"},
        "category": []string{"category_id", "id"},
        "tag":      []string{"id", "post_id"},
    }

here you can see we simply describe each relation via the related keys in each
model. The Post entity is related to the User entity via `user_id`, the Category
entity is related via `category_id`, and the Tag entity is related via the `id`
on the Post entity.

So, a possible implementation of this API could look something like this,

    func LoadRelations(loaders map[string]model.Loader, pp ...*Post) error {
        mm := make([]model.Model, 0, len(pp))

        for _, p := range pp {
            mm = append(mm, p)
        }

        for relation, keys := range relations {
            l := loaders[relation]
            err := l.Load(keys[1], model.MapKey(keys[0], mm...), model.Bind(keys[0], keys[1], mm...))

            if err != nil {
                return err
            }
        }
        return nil
    }

This looks ok, but not great. Sure this will work for the Post entity, but what
about the other entities in our blogging application? Also the way we specify
the relations isn't all that descriptive, this could definitely be improved
upon quite a bit.

## Implementing the API {#implementing-the-api}

So we've fleshed out the idea behind the API, and looked at one possible
implementation. Now let's go about implementing it. First we would ideally want
to utilise the `model.Model` interface as much as we can so save ourselves from
repeating too much code. Also, we want the API for specifying the entity
relations to be more descriptive than they currently are. So with these things
in mind let's go ahead an implement some utility functions in the `model`
package for handling relationships,

    // model/model.go
    ...
    type RelationFunc func(Loader, ...Model) error
    ...
    func Relation(a, b string) RelationFunc {
        return func(l Loader, mm ...Model) error {
            return l.Load(b, MapKey(a, mm...), Bind(a, b, mm...))
        }
    }

    func LoadRelations(relations map[string]RelationFunc, loaders map[string]Loader, mm ...Model) error {
        for relation, fn := range relations {
            if err := fn(loaders[relation], mm...); err != nil {
                return err
            }
        }
        return nil
    }

Now let's implement the actual relationship logic for the Post entity,

    // post/post.go
    ...
    relations = map[string]model.RelationFunc{
        "user":     model.Relation("user_id", "id"),
        "category": model.Relation("category_id", "id"),
        "tag":      model.Relation("id", "post_id"),
    }
    ...
    func LoadRelations(loaders map[string]model.Loader, pp ...*Post) error {
        mm := make([]model.Model, 0, len(pp))

        for _, p := range pp {
            mm = append(mm, p)
        }
        return model.LoadRelations(relations, loaders, mm...)
    }

What we've done here looks confusing at first, since we are returning a bunch
of callbacks which are then invoked later on, so let me explain.

First, we defined, and implemented the `Relation` function in the `model`
package. This function takes the two keys which cause the relationship to exist
between two models, whether that relation be one-to-one, one-to-many, or
many-to-one. The `Relation` function will return a callback that expects a
`model.Loader`, and a variadic list of `model.Model` interfaces. This callback
will take the two keys specified, `a` and `b`, and use them to invoke the
`Load` method on the given `model.Loader`, as you can see below.

    return l.Load(b, MapKey(a, mm...) Bind(a, b, mm...))

The next function we implemented was the `LoadRelations` function in the `model`
package. This differs from the one in the `post` package, as it takes a map
of `RelationFunc` types as the first argument, then the map of loaders, and
finally the variadic list of `model.Model` interfaces. This function will
iterate over the given `RelationFunc` map, and invoke the callback, passing
through the corresponding loader from the loaders map.

    for relation, fn := range relations {
        if err := fn(loaders[relation], mm...); err != nil {
            return err
        }
    }

This then all comes together in the `post` package when we implement the
`post.LoadRelations` function. Here, we actually passthrough the top-level
`relations` variable we defined to the `model.LoadRelations` function. Then, we
passthrough whatever loaders we were given, and convert the variadic list of
`Post` structs into a slice of `model.Model` interfaces.

So with this API in place we should be able to refactor the `Show` method on
our Post handler.

    // post/handler.go
    ...
    type Handler struct {
        Posts   Store
        Loaders map[string]model.Loader
    }
    ...
    func (h Handler) Show(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.ParseInt(mux.Vars(r)["post"], 10, 64)

        if err != nil {
            // handle error
        }

        p, err := h.Posts.Get(query.Where("id", "=", id))

        if err != nil {
            // handle error
        }

        if p.IsZero() {
            // handle not found
        }

        if err := LoadRelations(h.Loaders, p); err != nil {
            // handle error
        }
        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(p)
    }
    ...
    func RegisterRoutes(db *sqlx.DB, r *mux.Router) {
        h := Handler{
            Posts:   NewStore(db),
            Loaders: map[string]model.Loader{
                "user":     user.NewStore(db),
                "category": category.NewStore(db),
                "tag":      NewTagStore(db),
            },
        }
    ...

This is much better than what we had before.

## Utilising the Binder Interface {#utilising-the-binder-interface}

In the last post we implemented the `model.Binder` interface as a standalone
type. I did this so we could have entity stores implement the interface. Our
current implementation allows for models to be bound to each other, however it
would also be beneficial to have models bound to stores too.

For example, we have a `post.Store` type, well if we wanted to query all of the
posts by a certain category we'd current have to do something like this,

    NewPostStore(db).All(query.Where("category_id", "=", c.ID))

this is fine, however I would personally prefer it if this was done implicility.
Let's assume for a moment that the `post.Store` type does implement the
`model.Binder` interface, then we could do something like this to get all of
the posts for a certain category,

    posts := NewPostStore(db)
    posts.Bind(c)

    posts.All()

so let's go ahead and implement this.

    // post/post.go
    ...
    type Store struct {
        model.Store

        Category *category.Category
    }
    ...
    func (s *Store) Bind(mm ...model.Model) {
        for _, m := range mm {
            switch m.(type) {
            case *category.Category:
                s.Category = m.(*category.Category)
            }
        }
    }

    func (s Store) All(opts ...query.Option) ([]*Post, error) {
        pp := make([]*Post, 0)

        opts = append([]query.Option{
            model.Where(s.Category, "category_id"),
        }, opts...)

        err := s.Store.All(&pp, table, opts...)
        return pp, err
    }

In the above implementation we simply set the store's `Category` field to the
given model if it can be asserted to a `*category.Category`. Then, in the `All`
method we update the implementation to prepend a new query option to the slice
of given query options.

You will notice however that the `model.Where` query option isn't actually
something we have implemented, so let's go ahead and do that. Like with the
other query options we've implemented, we should make it play nice with
zero-values.

    // model/model.go
    ...
    func Where(m Model, args ...string) query.Option {
        return func(q query.Query) query.Query {
            if len(args) < 1 || m == nil || m.IsZero() {
                return q
            }

            var val interface{}

            col := args[0]

            if len(args) > 1 {
                val = m.Values()[args[1]]
            } else {
                _, val = m.Primary()
            }
            return query.Where(col, "=", val)(q)
        }
    }

With the implementation of the `model.Where` function we specify a variadic
list of string arguments. If only one argument is given then it will return a
`WHERE` clause on that given column, for example,

    model.Where(s.Category, "category_id")

would return an SQL equivalent of,

    WHERE (category_id = ?)

where `?` is the primary key of the model by default. If you wish to use another
value from the model to perform the `WHERE` clause with, then simply specify a
second argument to the variadic list like so.

    model.Where(s.Category, "category_id", "root_id")

So we have it implemented, however I'd also like to change the implementation of
the `post.NewStore` function to accomodate this change. It would be handy if we
could passthrough a variadic list of `model.Model` interfaces during creation of
a store that would automatically be bound to the returned store. Lets' go ahead
and do that,

    // post/post.go
    ...
    func NewStore(db *sqlx.DB, mm ...model.Model) Store {
        s := Store{
            Store: model.Store{DB: db},
        }
        s.Bind(mm...)
        return s
    }

## Conclusion {#conclusion}

We're starting to approach the end of this series of posts, and I hope that
some of the ideas presented throughout are starting to make sense. The end goal
here is to implement a relationship handling system that will make working with
SQL relationships in Go feel as idiomatic as possible.

In the next post I intend on finishing up the implementation of the blogging
application I've been using to demonstrate these ideas, and justifying how I
came about the implementations I've been using. I also intend on uploading the
source code of this toy blogging application so that you can get a feel as to
how this would all fit together. If you've been keen eyed then you should have
noticed some flaws in the design of this blogging application, something that I
too will address in the next post.
