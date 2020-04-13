---
title: Working with SQL Relations in Go - Part 5
layout: post
index: true
createdAt: 2020-04-13T14:04
updatedAt: 2020-04-13T14:04
---
Over these series of posts I have been exploring an approach that could be taken
when working with SQL relationships in Go. The precursor to all of this was the
initial [ORMs and Query Building in Go](/programming/2019/07/13/orms-and-query-building-in-go). This explored one aspect of ORMs, the query building, but it didn't address
how relationships could also be handled in an idiomatic way. So before I wrap up
this series in this final post, let us address the code we have currently.

* [Finishing up the Application](#finishing-up-the-application)
* [Callbacks and Interfaces](#callbacks-and-interfaces)
* [A Note on Generics](#a-note-on-generics)
* [Why Not Make this a Library](#why-not-make-this-a-library)
* [Conclusion](#conclusion)

>**Note:** If you're interested in taking a look at the code I put together for
this example application I put together, then take a look at it online here:
https://github.com/andrewpillar/blogger.

## Finishing up the Application {#finishing-up-the-application}

Previously, we successfully implemented the `Index`, and `Show` methods for the
Post entity. However, for the Category entity we need to update the `Show`
method so that we return a list of posts for that category. This can be done by
utilising the `model.Binder` interface we implemented on the `post.Store`
struct.

    // category/handler.go
    package category

    import (
    ...
        "blogger/post"
    ...
    )
    ...
    func (h Handler) Show(w http.ResponseWriter, r *http.Request) {
    ...
        pp, paginator, err := post.NewStore(h.DB, c).Index(r.URL.Query())

        if err != nil {
            // handle error
        }

        data := struct{
            Category *Category
            Prev     string
            Next     string
            Posts    []*post.Post
        }{
            Category: c,
            Prev:     fmt.Sprintf("/category/%d?page=%d", c.ID, paginator.Prev),
            Next:     fmt.Sprintf("/category/%d?page=%d", c.ID, paginator.Next),
            Posts:    pp,
        }
        w.Header().Set("Content-Type", "application/json; charset=utf-8")
        json.NewEncoder(w).Encode(data)
    }

You'll notice here, that we imported the `blogger/post` package which will
result in an import cycle. This can be easily fixed by creating a third
sub-package in the `post` and `category` packaged called `web` to hold the web
handler implementations.

The application at this point is mostly finished, if you wish to see a complete
example of this then take a look at the repository in GitHub,
https://github.com/andrewpillar/blogger.

Now let me go about trying to justify the approach I took to this problem.

## Callbacks and Interfaces {#callbacks-and-interfaces}

When it comes to working with SQL relationships in Go there is going to be
similarities between how things are done. For example, we want to load
relationships, as well as bind them. Not to mention that obvious similarities
between the entities we have, they all have 64-bit integer primary keys, and
they each have differen relations.

Because of these similarities it is only natural to look to an interface to
implement what we need when it comes to relationship loading. So when it comes
to writing the actual code we can just take the interface we have, and tell it
"load in the relationships I want", without necessarily caring how they're
loaded in. Furthermore, we also implemented a light interface to represent our
entity models.

The actual logic for binding the models to one another is deferred to a function
callback. This makes sense to do, since different models could be bound in
different ways. However, with the implementation of the `model.Model` interface
we were able to implement the `model.Bind` function to have a generic way of
binding our models together, assuming that the models have 64-bit integer keys.

We take these function callbacks a step further, and allow for the description
of how these models are bound together via the `model.Relation` function and
`model.RelationFunc` type.

As you can see, when coupled together, callbacks and interfaces can achieve what
we went in a way that is fairly idiomatic. And we managed to do this without
having to dip into the `reflect` package.

## A Note on Generics {#a-note-on-generics}

I touched on generic behaviour briefly, so I may aswell add my two cents on the
whole generic situation in Go.

When I first approached Go I rather liked the lack of supports for generics, and
wouldn't have minded if the language continued without generics. This belief
mainly arose from the fear of people abusing generics to write god code (code
that is so generic and arbirtrary it could do anything, and yet is hard to
understand). However, I think my fears in this regard are unfounded, mainly
because some people like abusing `interface{}` and `reflect` to achieve this
instead.

That being said, I cannot deny that certain things would be easier with generics
in Go. It is comforting to see some of the [performance gains](https://blog.tempus-ex.com/generics-in-go-how-they-work-and-how-to-play-with-them/) that can be made via the use of generics in Go.

So I would welcome generics in Go, and hope that people use them responsibly.

## Why Not Make this a Library {#why-not-make-this-a-library}

One final thing I should address before wrapping this up, is why didn't I take
what I have written and turn it into a library? Well, I didn't turn this into
a library for a platitude of reasons.

The first being that I don't want to make any assumptions about how people went
about modelling their data. For example, you may use something other than an
integer for your primary key, perhaps a string, a byte array. And I think this
is another thing where ORMs fall short, and that is making assumptions about the
data being worked with.

Second of all, this implementation only contains a handful of functions and
interfaces. And because of what I mentioned in the first post, it makes a
number of assumptions about the data.

Finally, since the implementation is only a handful of functions and interfaces,
I don't think this would make for a very substantial library. Also, I would
defer to one of the [Go proverbs](https://go-proverbs.github.io/) here too,
"A little copying is better than a little dependency".

## Conclusion {#conclusion}

I hope the ideas presented throughout this series of posts will help you when
it comes to working with SQL relationships in Go. This is something I have
struggled with, especially since the solutions out there for modelling data are
lacking, perhaps due to the points I made earlier. I also want to say, that
these ideas are not gospel, just an approach that I have found that works for
me. As always feel free to contact me to discuss this further.
