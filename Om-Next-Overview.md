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
to do something simple.

The primary trick I'm going to exploit is the fact that `env` is just a map, and
that we can add stuff to it. When we are in the context of a person, we'll add
`:person` to the environment, and pass that to parse. This makes parsing a
selector like `[:name :age]` as trivial as:

```clj
(defmethod rread :name [{:keys [person]} key _] {:value (get person key)})
(defmethod rread :age [{:keys [person]} key _] {:value (get person key)})
(defmethod rread :married [{:keys [person]} key _] {:value (get person key)})

(defmethod rread :friends [{:keys [state selector parse path] :as env} key params]
  (let [friend-ids (get @state :friends)
        get-person (fn [id]
                     (let [raw-person (get-in @state [:people/by-id id])
                           env' (dissoc env :selector) ; clear the parent selector
                           env-with-person (assoc env' :person raw-person)]
                       (parse env-with-person selector)
                       ))
        friends (mapv get-person friend-ids)]
    {:value friends}
    )
  )
```

The three important bits:

- We need to remove the :selector from the environment, otherwise our nested
  read function will get the old selector on plain keywords, making it 
  impossible to tell if the parser saw [:married-to] vs. { :married-to [...] }.
- For convenience, we add `:person` to the environment.
- The `rread` for plain scalars (like `:name`) are now trivial...just look on the
  person in the environment!

The final piece is hopefully pretty transparent at this point.  For
`:married-to`, we have two possibilities: it is queried as a raw value
`[:married-to]` or it is joined `{ :married-to [:attrs] }`. By clearing
the selector in the `:friends` rread, we can tell the difference
since parse will add back a selector if it parses a join.

**NOTE to David**: Seems like we could clean the parser up a bit. Possibly
pass the AST node type through to the reader, or at least clear the 
selector automatically on a plain prop.

So, our final bit of this parser could be:

```clj
(defmethod rread :married-to
  [{:keys [state person parse selector] :as env} key params]
  (let [partner-id (:married-to person)]
    (cond
      (and selector partner-id) { :value [(select-keys (get-in @state [:people/by-id partner-id]) selector)]}
      :else {:value partner-id}
      )))
```

If further recursion is to be supported on this selector, then rinse and repeat.

For those who read to the end first, here is an overall runnable segment of code
for this parser:

```clj
(def app-state (atom {
                      :window/size  [1920 1200]
                      :friends      #{1 3} ; these are people IDs...see map below for the objects themselves
                      :people/by-id {
                                     1 {:id 1 :name "Sally" :age 22 :married false}
                                     2 {:id 2 :name "Joe" :age 22 :married false}
                                     3 {:id 3 :name "Paul" :age 22 :married true :married-to 2}
                                     4 {:id 4 :name "Mary" :age 22 :married false}}
                      }))

(def query-props [:window/size {:friends [:name :married :married-to]}])
(def query-joined [:window/size {:friends [:name :married {:married-to [:name]}]}])

(defmulti rread om/dispatch)

(defmethod rread :default [{:keys [state]} key params] (println "YOU MISSED " key) nil)

(defmethod rread :window/size [{:keys [state]} key params] {:value (get @state :window/size)})

(defmethod rread :name [{:keys [person selector]} key params] {:value (get person key)})
(defmethod rread :age [{:keys [person selector]} key params] {:value (get person key)})
(defmethod rread :married [{:keys [person selector]} key params] {:value (get person key)})

(defmethod rread :married-to
  ;; person is placed in env by rread :friends
  [{:keys [state person parse selector] :as env} key params]
  (let [partner-id (:married-to person)]
    (cond
      (and selector partner-id) {:value [(select-keys (get-in @state [:people/by-id partner-id]) selector)]}
      :else {:value partner-id}
      )))

(defmethod rread :friends [{:keys [state selector parse path] :as env} key params]
  (let [friend-ids (get @state :friends)
        keywords (filter keyword? selector)
        joins (filter map? selector)
        get-person (fn [id]
                     (let [raw-person (get-in @state [:people/by-id id])
                           env' (dissoc env :selector)
                           env-with-person (assoc env' :person raw-person)]
                       ;; recursively call parse w/modified env
                       (parse env-with-person selector)
                       ))
        friends (mapv get-person friend-ids)]
    {:value friends}
    )
  )

(def my-parser (om/parser {:read rread}))

;; remember to add a require for cljs.pprint to your namespace
(cljs.pprint/pprint (my-parser {:state app-state} query-props))
(cljs.pprint/pprint (my-parser {:state app-state} query-joined))
```

**NOTE**: Comments desired. I'm trying to make the points clear above, not write
the most elegant possible recursive parser. If you have suggestions for
improving the example that will make the main points more clear, please let
me know on the Slack #om channel or via an issue/PR.

## Parameters

In the query grammar most kinds of rules accept parameters. These are intended
to be combined with dynamic queries that will allow your UI to have some control
over what you want to read from the application state (think filtering and such).

Remember that Om Next has a story for integrating with server communications,
and these queries are meant to be transparent (from the UI perspective). If
the UI needs less data, parameters and propery selectors can fine-tune what
gets transferred over the wire.

As you might expect, the parameters are just passed into your read function as
the third argument. You are responsible for both defining and interpreting them.
They have no rules other than they are maps:

```clj
[(:load/start-time {:locale "es-MX" })]                ;;prop + params
```

invokes read with:

```clj
(your-read env :load/start-time { :locale "es-MX" })
```

the implication is clear. The code is up to you.

## References

The query grammar includes a form for an entity reference:

```clj
[:people/by-id id]
```

The *convention* is that the keyword be namespaced with the type of thing,
and the name indicate how it is indexed. This is a convention, but since
it is self-documenting, it is wise to follow it. 

Also, remember the queries are enclosed in vectors, so asking for something by
reference means you end up with a double-nested vector:

```clj
[ [:people/by-id 42] ]
```

means look up a person whose id 42. The response will be:

```clj
{ [:people/by-id 42] ...whatever your read returns... }
```

Again, by convention, your read probably returns a map, but there is nothing
to say that this couldn't be a more complex object (e.g. a Datascript Entity).

### References and state

In the earlier example of writing read/parser, the app state was using raw numbers
as IDs. For demonstration purposes this was simpler (in that it required no
further explanation), but you'll find it much easier to reason about your code
if you store your references in the query grammar format above. It make the 
data more obvious, and it also makes it trivial to write your read methods since
the incoming key can just go straight to a `get-in`. Below is an example of
our previous app state refactored to use reference notation:

```clj
(def app-state (atom {
    :window/size [1920 1200]
    :friends #{[:person/by-id 1] [:person/by-id 3]}
    :person/by-id { 
            1 { :id 1 :name "Sally" :age 22 :married false }
            2 { :id 2 :name "Joe" :age 22 :married false }
            3 {:id 3 :name "Paul" :age 22 :married true :married-to [:person/by-id 2]}
            4 { :id 4 :name "Mary" :age 22 :married false } }
     }))
```

If you now go back and work through the parser code, you'll see that a lot
of it gets a bit shorter. Not a bad exercise for the reader. Those who
have read [[Components, Identity & Normalization]] will recognize that
the built-in normalization support can use your UI code to build
normalized tables that look a lot like what you see above. 

## Mutation

OK, the mutation side of the grammar should now be easier to understand. Some of
the details are the same: The parser understands the basic structure of the
syntax, but you're responsible for the grunt work.

Running mutations is done with `transact!`. For the moment we'll ignore the UI
side of the equation and just concentrate on the data. It's important to
understand that there are two concerns: changing the app state and updating
the UI to reflect the changes. Your mutation functions don't have to worry
(too much) about the UI. The details of UI update are still in a little bit
of flux, and you can definitely get mutation working (i.e. changing app state)
without thinking about the UI at all. 

So, we'll start with understanding how to write our mutations, and we'll verify
that our app state changes.

### The primary mutation grammar

Mutations go in a vector, just like reads. The *element* grammar looks just like
a clojure function call (albeit a function of arity one that optionally takes a
map).

```clj
(fire-missiles! {:target :foo})
```

So, a transaction is just a vector of these:

```clj
[ (make-friend { :person/by-id 42 })
  (send-business-card { :person/by-id 33 }) ]
```

Now, this makes a lot of sense to those that are well-versed in Clojure(script),
but for those that are starting out I want to point out that a transaction
(the vector above) needs to be *data*. If we type it into our code as shown, the
compiler will try to evaluate the plain lists (e.g. `(make-friend)`) and
will crash with "symbol not found". Those "symbols" are not *meant* to be
functions in our code, they are meant to be instructions to our `transact!`.

Thus you'll often see them quoted:

```clj
'[ (f) (g) (h) ]
```

Unfortunately, plain quoting is kind of a pain when you want to embed values
from variables, since everything is literal. If we use a different quote, the
syntax quote, then we also have the ability to unquote with `~`:

```clj
(let [p 42]
  `[ (f { :person ~p }) ]
  )
```

The `~` is an unquote, which says "stop quoting for this next form". Alas, this
still isn't quite right, because syntax quoting tries to "help you" (these are
used in macros, where they do actually help) by putting namespaces on all of the
symbols! Thus, `f` in the above example ends up with the current namespace
prepended (e.g. `om-tutorial.core/f`). You can prevent this by namespacing your
symbols manually. The symbols are all make-believe, so just make them work for
your understanding:

```clj
(let [p 42]
  `[ (people/make-friend { :person/by-id ~p } ]
  )
```

Read up on quote, syntax quote, and unquote. If you're working in Om Next, it's
time you understood how to quote.

Another strategy is to use the `list` function to build the list:

```clj
[ (list 'make-friend {:p p} ) ]
```

This is both better and worse. `p` is properly expanded, we can use plain
quoting on the symbol `make-friend`, namespacing isn't an issue, and there
are no squiggles...but now we have the word list in there. 

One might suggest

```clj
(def invoke-mutation list)
```

so you can write:

```clj
[(invoke-mutation 'make-friend {:p p})]
```

I'm sure others will write macros. Whatever. 

### Checkpoint

So, the summary of facts we covered so far:

- Mutation grammar is just data. It looks like function calls, but should be
  passed to transact as data (for transact to parse, not the cljs compiler)
- The grammar uses lists, which the compiler will be sorely tempted to interpret
  into function calls.
- Use quoting (or whatever mechanism you want) to make sure `transact!` receives
  the grammar unmolested.

### The Pieces of the Mutation Puzzle

There are several bits that fit together to make mutation work (and of course
those pieces are also ultimately interested in your UI being up-to-date).

The pieces are:

- Reconciler : This is the bit that tries to merge novelty into the state. It
  sits between your UI and the mutation functions you write. It is 
  (mostly indirectly) invoked when you run `transact!`.
- Parser : This is the same parser you were using for reads. When it sees
  something that conforms to a `mutate` grammar, it tries to run it instead
  of calling read. The reconciler uses the parser to understand the grammar
  of the `transact!` statements.
- Mutation functions: Code you write that understands how to actually change
  the underlying application state. The parser ends up invoking these when
  it sees a mutation grammar form.
- Indexer : This one keeps track of things. It knows which components on the
  screen share the same Ident (and therefore should update when that underlying
  data changes). The indexer is incomplete, but my understanding of the plan
  is to leverage this to help figure out what needs to update in the UI
  when data changes with more interesting cases than just Ident. At the
  moment, you have to help a bit. 

### Your part: The mutate function

Just like `read`, the `mutate` operation is just a function you write
that is called with the signature `[env key params]`. In the case
of mutate, the `key` will be the symbol you "called" and `params` will
be the map you passed. `env` is as before (includes app state).

Since you invented the symbol name, you invent the operations, but there
is one catch: you don't do it during the call itself. You instead return
a lambda to do it for you when the reconciler decides it wants to do it:

```clj
(defn mutate [env key params]
   (cond
      (= 'make-friend key) { :action (fn [] ...) }
      (= 'send-business-card key) { :action (fn [] ...) }
      ))
```

Of course, as before, you probably want to use a multimethod that dispatches
on key instead of some monolith.

**NOTE:** We're just talking state changes at the moment. There is more to say
about how this works with the UI...there are more requirements.

### Mutation Example (sans UI)

The [[Quick Start (om.next)]] has already covered a mutation example, but
for completeness, we'll include another sample here:

```clj
(def app-state (atom {:count 0}))

(defmulti mutate om/dispatch)

(defmethod mutate 'counter/add [{:keys [state]} key params]
  (let [n (or (:n params) 1)]
    {:action (fn [] (swap! state update :count (partial + n)))}
    ))

(def my-parser (om/parser {:mutate mutate}))

(def reconciler (om/reconciler {:state app-state :parser my-parser}))
```

Now every time you run a `transact!` like this:

```clj
(om/transact! reconciler '[(counter/add {:n 3})])
```

the contents of `@app-state` will go up by amount `n`.

*IMPORTANT*: You should not, in general, call `transact!` directly on the
reconciler. Such a call is legal, but ...(*HELP*: someone tell me the
badness. I know "David says don't do it, normally", but I cannot explain why.
Does it cause a full re-render? I don't have all the facts here...I'm vamping).

### Notes on Mutate

So far, I'm aware of some restrictions:

- Do not remove objects from normalized tables generated by Om Next. A GC is
  planned for that kind of state, and removing the objects could cause bad
  things to happen (e.g. are you sure the object isn't being rendered by
  something else, or isn't needed in the future?)

### More of the Mutate Story

Om Next needs a little help from your mutation function. Sure, it is simple for
you to go about writing new random bits of data into your app state, but Om Next
can't read your mind (or your code, even though the latter might be
possible...it *is* Clojurescript, after all).  

The indexer and declared Ident in the UI code give it some ability to figure out
what changed, but since your read functions could in fact do any number of
interesting things (like derive aggregates) there is no way to completely
automate the re-render of the UI (except by doing the pathalogical thing and
re-render it all!).

For your UI to work properly, you need to give Om Next some hints about what
you changed. This is the purpose of the `:value` attribute in the return
value of mutate (which I've elided to this point):

(defmethod mutate 'counter/add [{:keys [state]} key params]
  (let [n (or (:n params) 1)]
    { :action (fn [] (swap! state update :count (partial + n)))
      :value [:count] }
    ))

*HELP*: I know `:value` is a query, but I'm not clear on the details (and I 
know they are not complete). I'm pretty sure it ends up being used as a key into
the indexer to try to find components that rendered the data, but does it need
to have the recursive bits?

### Calling transact

The general rule for calling `transact!` is to invoke it within a component
instance that conceptually owns the target data of the mutation. The first
argument to `transact!` is that component:

```clj
(defui Thing
  static om/IQuery
  (query [this] '[:boo])
  Object
  (render [this]
      (dom/button #js { :onClick #(transact! this '[(change-boo)]) } )))
```

Components with no IQuery declaration should not invoke it, but instead should
use callbacks. Subcomponents that wish to affect parents should do the same.
This means simply passing a lambda through props.

*Side note*: In discusson on the #om slack channel, it seems that it is likely
as the indexer matures it may become possible to declare more abstractly what
a component relies on such that Om Next can fully derive the UI update set. If
this works out the rules around `transact!` will likely improve.

## The UI (finally!!!)

So, at this point you understand most of the basic state management code
you'll need to write. We'll clarify the mutation story a bit more as we 
talk about real UI, since the remaining bits of mutation have to do
with our biggest UI concern: keeping the UI up-to-date with respect
to the application state. Om Next already has a great story for a lot of this,
and it will likely improve.

Once you've understood the read function, parser, and query grammar then you
should be capable of writing a UI component that gets something on the
screen. Hopefully you've already done that in the [[Quick Start (om.next)]].

In this section I want to make sure you understand:

- How to compose the UI (and queries)
- The difference between stateful and stateless components
- A bit about how Om Next tracks queries
- Proper use of `transact!`
- How to force additional UI updates (possible hack) when using `transact!`
- A bit more on Ident
