# **WARNING: THIS DOCUMENT HAS NOT BEEN REVIEWED/APPROVED BY DAVID. IT MAY CONTAIN INCORRECT OR MISLEADING INFORMATION**

Also, refresh your browser often. I'm pushing very frequent commits to this file
at the moment.

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

**NOTE**: You will often see quoting used in queries. Many of the examples
in this document do not use quoting, because the data in question is
just data. You should make sure you understand quoting, syntax quote, 
and unquote (in Clojure(script)). Some of the grammar uses parens, 
which is what leads to the quoting in examples.

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

In order to get started, we'll just consider the two most common bits of query
notation: keywords (props) and joins.  The params are simply that: parameters
you want to pass to the query. These can be static data, and can also be
manipulated. But more on that later.

### Keyword (prop)

The simplest item in the vector is just a keyword. More than one indicates you want 
a result for each:

```clj
[:user/name :user/address]
```

which means "I'd like to query for the properties :user/name and :user/address".

The result of a parse of such a query must always be a map. In this case, a map keyed by the
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

Remember that the output of a parse of a query is always a map, and in this case
the query result should take this form:

```clj
{ :people [ {:person/name "Joe" :person/address "..." }
            {:person/name "Sally" :person/address "..." }
            {:person/name "Marge" :person/address "..." } ]
  :places [ {:place/name "Washington D.C." } ]
}
```

*IMPORTANT NOTE*: Om Next has no magic here. It cannot magically create this
output result...you have to play along.  The parser, however, is written to make
it pretty easy. So, let's look at how we do that.

## Implementing Read

When building your application a good deal of the initial effort will be
in writing you read function such that it can pull data that the 
parser needs to fill in the result.

The Om Next parser understands the grammar, but you know your data.
By default, the parse only invokes read for each top-level element
in the query. So:

```clj
[:kw {:j [:v]}]
```

would result in a call to your read function on :kw and {:j [:v]}. Two
calls. No automatic recursion. Done. The output value of the 
*parse* will be a map (that parse creates) which contains
the keys (from the query, copied over by the parser) and 
values (obtained from your read):

```clj
{ :kw value-from-read :j value-from-read }
```

Note that if your read accidentally returns a scalar for `:j` then you've not
done the right thing...a join like `{ :j [:k] }` expects a result that is a
vector of (zero or more) things.

Dealing with recursive queries is a natural fit for a recusive algorithm, and it
is perfectly fine to invoke the om/parse function to descend the query. In fact,
the parse function is passed as part of your environment.

So, the read function you write:

- Will receive three arguments:
    - An environment containing:
        - `:parse`: The query parser
        - `:state`: The application state
        - `:selector`: **if** the query had one E.g. {:people [:user/name]} has selector [:user/name]
    - A key whose form may vary based on the grammar form used (e.g. :user/name).
    - Parameters (which are nil if not supplied in the query)
- Must return a value that has the shape implied the grammar element being read.

The parse will create the output map.

### Reading a keyword

If the parser encounters a keyword `:kw`, your function will be called with:

```clj
(your-read { :state app-state :parse (fn ...) } ;; the environment. App state and the parse function
           :kw                                  ;; the keyword
           nil) ;; no parameters
```

in this case, your read function should return some value that makes sense for
that spot in the grammar.  There are no real restrictions on what that data
value has to be in this case. There is no further shape implied by the grammar.
It could be a string, number, Entity Object, JS Date, nil, etc.

Due to additional features of the parser, your return value must be wrapped in a
map with the key `:value`. Thus, a very simple read for keywords could be:

```clj
(defn read [env key params] { :value 42 }) 
```

and you have a read function that returns the meaning of life the universe and
everything in a single line! Of course we're ignoring the meaning of the
question, but we can now read any query that contains only keywords. Try it out
via the REPL:

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
Joins are the naturally recursive, and if we revert back to
our old parser that thinks `42` is right for all questions:

```clj
(def app-state (atom {:a 1 :b 2 :c 99}))
(defn read [env key params] { :value 42 })
(def my-parser (om/parser {:read read}))
(my-parser {:state app-state} '[:a {:user [:user/name]} :c])
```

then the read of the join is:

```clj
{:a 42, :user 42, :c 42}
```

No recursion happened! If you put a println in the read, you'll see it is only
called three times.

Those that are used to writing parsers probably already see the solution.

Let's clarify what the read function will receive in this case. When
parsing:

```clj
{ :j [:a :b :c] }
```

your read function will be called with:

```clj
(your-read { :state state :parse (fn ...) :selector [:a :b :c] } ; NOTE: selector is set
           :j                                                    ; keyword as expected
           nil)
```

So, we can get a basic recursive parse using just a bit more flat data:

```clj
(def app-state (atom {:a 1 :user/name "Sam" :c 99}))

(defn read [{:keys [state parse selector] :as env} key params]
  (if (= :user key)
    {:value [(parse env selector)]}
    {:value (get @state key)}))

(def my-parser (om/parser {:read read}))
(my-parser {:state app-state} '[:a {:user [:user/name]} :c])
```

The important bit is the `then` part of the `if`. Return a value that is 
the recursive parse of the selector. Otherwise, we just look up the keyword
in the state (which is a very flat map).

The return value now has the correct structure of the desired response:

```clj
{:a 1, :user [{:user/name "Sam"}], :c 99}
```

*IMPORTANT NOTE*: A join implies a there could be many results. That is the
reason we've wrapped the parser result into a vector.

In this case, we're just showing that you can use the parser to parse something
you already know how to parse, and that in turn will call your read function.
In a real application, you will almost certainly not call `parse` in quite this 
way (since you need to actually do something to *run* your join!).

So, let's put a little better state in our application, and write a 
more realistic parser.

### A Non-trivial, Recursive Example

Let's start with the following hand-normalized application state. Note that 
I'm not using the query grammar for object references (which take the 
form [:kw id]). Writing a more complex parser will benefit from doing
so, but it's our data, and we can do what we want to! (actually, there
do seem to be some limits, but I'm still gaining clarity on that).

```clj
(def app-state (atom {
    :window/size [1920 1200]
    :friends #{1 3} ; these are people IDs...see map below for the objects themselves
    :people/by-id { 
            1 { :id 1 :name "Sally" :age 22 :married false }
            2 { :id 2 :name "Joe" :age 22 :married false }
            3 { :id 3 :name "Paul" :age 22 :married true }
            4 { :id 4 :name "Mary" :age 22 :married false } }
     }))
```

now we want to be able to write the following query:

```clj
(def query [:window/size {:friends [:name :married]}])
```

Here is where multi-methods start to come in handy. Let's use 
one:

```clj
(defmulti rread om/dispatch) ; dispatch by key
(defmethod rread :default [{:keys [state]} key params] nil)
```

The `om/dispatch` literally means dispatch by the `key` parameter.
We also define a default method, so that if we fail we'll get
an error message in our console, but the parse will continue
(returning nil from a read elides that key in the result).

Now we can do the easy case: If we see something ask for 
window size:

```clj
(defmethod rread :window/size [{:keys [state]} key params] {:value (get @state :window/size)})
```

Bingo! We've got part of our parser. Try it out:

```clj
(def my-parser (om/parser {:read rread}))
(my-parser {:state app-state} query)
```

and you should see:

```clj
{:window/size [1920 1200]}
```

The join result (`friends`) is elided because our default rread got
called and returned `nil` (no results). OK, let's fix that:

```clj
(defmethod rread :friends [{:keys [state selector parse path]} key params]
      (let [friend-ids (get @state :friends)
            get-friend (fn [id] (get-in @state [:people/by-id id]))
            friends (mapv get-friend friend-ids)]
        {:value friends}
        )
      )
```

when you run the query now, you should see:

```clj
{:window/size [1920 1200], 
 :friends [{:id 1, :name "Sally", :age 22, :married false} 
           {:id 3, :name "Paul", :age 22, :married true, :married-to 2}]}
```

Looks *mostly* right...but we only asked for `:name` and `:married`. Your 
read function is responsible for the value, and we ignored the selector!

This is pretty easy to remedy with the standard `select-keys` function. Change
the get-friend embedded function to:

```clj
            get-friend (fn [id] (select-keys (get-in @state [:people/by-id id]) selector))
```

and now you've satisfied the query:

```clj
{:window/size [1920 1200], 
 :friends [{:name "Sally", :married false} 
           {:name "Paul", :married true}]}
```

Those of you paying close attention will notice that we have yet to need
recursion. We've also done something a bit naive: select-keys assumes
that selector contains only keys! What if our query were instead:

```clj
(def query [:window/size 
            {:friends [:name :married {:married-to [:name]} ]}])
```

Now things get interesting, and I'm sure more than one reader will have
an opinion on how to proceed. My aim is to show that the parser
can be called recursively to handle these things, not to find
the perfect structure for the parser in general, so I'm going 
to do something simple:



