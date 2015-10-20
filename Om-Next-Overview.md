# **WARNING: THIS DOCUMENT HAS NOT BEEN REVIEWED/APPROVED BY DAVID. IT MAY CONTAIN INCORRECT OR MISLEADING INFORMATION**

## Introduction

It is a good idea to start with [[Quick Start (om.next)]].

This overview is intended to help you understand how Om Next works, 
how you do the basic UI creation and composition, and how to
write the various components you supply to build your application.

At the moment, this document does not cover remote, but instead
covers the basics you need for building UI based purely on local state.

There will be examples that you can try, and they should work just 
fine via the setup described in the [[Quick Start (om.next)]].

## Application state

Your application state can take any form you want. It can be in a browser
database, a cljs map held by an atom, etc. The representation of your
state needs to be something you can reason about and maniuplate.

It is also required that this state be normalized (which Om Next
can automatically do, but for this document, we'll use something
we've normalized ourselves).

Part of the trouble with comprehending Om Next from the start is
the fact that your application state can take any shape you want.
The grammar of the component queries is unrelated (though if you
want to structure your app state to make it easier to join the
two together, that's nice too).

You're trying to make it possible to do two things:

- Write arbitrary functions that can retrieve data when given a query in the Om Query Grammar.
- Write arbitrary functions that can update your app state **and** inform Om Next about
  what you changed.

Let's start with the function that retrieve data, and the Grammar.

## Reads and the Query Grammar

The basic read functions provide a "router/parser" in concept. The 
base requirement is that they understand the Query Grammar, and
return data in a form that matches the expected query response.

All queries are vectors. If the vector contains multiple things, then you are asking 
for multiple seperate results.

OK, so what can you put in this vector?

**NOTE:** The grammar is not yet complete, but I'll cover the bits that 
are generally stable at the time of this writing (Oct 2015).

For reference, here are the defined grammar elements:
```clj
[:some/key]                              ;;prop
[(:some/key {:arg :foo})]                ;;prop + params
[{:some/key [:sub/key]}]                 ;;join + selector
[({:some/key [:sub/key]} {:arg :foo})]   ;;join + params
[[:foo/by-id 0]]                         ;;reference
[(fire-missiles!)]                       ;;mutation
[(fire-missiles! {:target :foo})]        ;;mutation + params
```

with additional ones on the way.

### Getting Started

In order to get started, we'll just consider the two most common bits of query notation: keywords and joins.

### Keyword

The simplest item in the vector is just a keyword. More than one indicates you want 
a result for each:

```clj
[:user/name :user/address]
```

which means "I'd like to query for the properties :user/name and :user/address".

The output of read must always be a map. In this case, a map keyed by the
requested properties:

```clj
{ :user/name "Joe" :user/address "111 Nowhere St." }
```

### Join with selector

Basic form:

```clj
{ :keyword SELECTOR }
```

where SELECTOR is a vector of further things (e.g. keywords or joins).

So a join *entry* for your query is written as a (possibly recursive) map:

```clj
{ :property/name [:key1 :key2] }
```

The intended interpretation is that of a graph walk. 
In other words, "this thing I'm querying will have something it calls
:property/name, which will reference one or more things. I'd like
to get :key1 and :key2 on each of them".

So, a *query* might look like this:

```clj
[
  {:people [:person/name :person/address]} 
  {:places [:place/name]} 
]
```

which is asking for two things (which themselves will contain zero or more
sub-things).

Remember that the output of a query is always a map, and in this case, 
the query result should take this form:

```clj
{ :people [ {:person/name "Joe" :person/address "..." }
            {:person/name "Sally" :person/address "..." }
            {:person/name "Marge" :person/address "..." } ]
  :places [ {:place/name "Washington D.C." } ]
}
```

*IMPORTANT NOTE*: Om Next has no magic here. It cannot magically
create this output result...that is what you are supposed to do.
The parser, however, is written to make it pretty easy. So, let's
look at how we do that.

## Implementing Read

When building your application a good deal of the initial effort will be
in writing you read function such that it can interpret the grammar.

The Om Next parser understands the grammar, but it does nothing more
than read the top-level query and call your function on each element.

```clj
[:kw {:j [:v]}]
```

would result in a call to your read function on :kw and {:j [:v]}. Two
calls. No automatic recursion. Done.

Dealing with recursive queries is a natural fit for a recusive 
algorithm, and it is perfectly fine to invoke the om/parse function
to descend the query. In fact, the parse function is passed as
part of your environment.

So, the read function you write:

- Will receive three arguments:
    - An environment containing:
        - `:parse`: The query parser
        - `:state`: The application state
        - `:selector`: **if** the query had one E.g. {:people [:user/name]} has selector [:user/name]
    - A key whose form may vary based on the grammar form used (e.g. :user/name).
    - Parameters (which are nil if not supplied in the query)
- Must return a value that has the shape implied the grammar element being read.

The parse will create an output map that has the key (e.g. :user/name) associated
with your return value.

### Reading a keyword

If the parser encounters a keyword `:kw`, your function will be called with:

```clj
(your-read { :state app-state :parse (fn ...) } ;; the environment. App state and the parse function
           :kw                                  ;; the keyword
           nil) ;; no parameters
```

in this case, your read function should return some value that makes sense for that spot in the grammar.
There are no real restrictions on what that data value has to be. It could be a string, number,
Entity Object, JS Date, etc. The grammar says a value has to be there, but it doesn't care
about what is in it.

Your return value must be a map, and due to additional features you need to wrap your 
return value in a map with the key `:value`. Thus, a very simple read for keywords 
could be:

```clj
(defn read [env key params] { :value 42 }) 
```

and you have a read function that returns the meaning of life the universe and everything in 
a single line! Of course we're ignoring the meaning of the question, but we can now read
any query that contains only keywords. Try it out via the REPL:

```clj
(def state (atom {}))
(defn read [env key params] { :value 42 })
(def my-parser (om/parser {:read read}))
(my-parser {:state state} '[:a :b :c])
```

and you should see the following output:

```clj
{:a 42, :b 42, :c 42}
```

As advertised! Every property in your system now has the value 42. 

If your app state is just a flat set of values with unique keyword
identities, then a real read is similarly trivial:

```clj
(def app-state (atom {:a 1 :b 2 :c 99}))
(defn read [{:keys [state]} key params] { :value (get @state key) })
(def my-parser (om/parser {:read read}))
(my-parser {:state app-state} '[:a :b :c])
```

and you should see the following output:

```clj
{:a 1, :b 2, :c 99}
```


### Reading a join

Of course, your app state probably has some more structure to it.
Joins are the naturally recursive


