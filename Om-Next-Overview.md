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

So, the rules you have to follow are quite simple:

- You will receive:
    - An environment containing:
        - state: The application state
        - selector: if the query had one E.g. {:people [:user/name]} has selector [:user/name]
    - A key whose form varies based on grammar form used (e.g. :user/name)
    - Parameters (which are nil if not supplied in the query)
- You must return a map that has shape specified by the query result. (e.g.
the nestest map of people and places shown in the prior section).

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


# END OF CURRENT DOCUMENT. TEXT BELOW KEPT FOR QUICK REFERENCE

## Setting Up

Please follow the setup (through Start Figwheel) in the [[Quick Start (om.next)]].


Edit `src/om_tutorial/core.cljs` to look like the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(defui HelloWorld
  Object
  (render [this]
    (dom/div nil "Hello, world!")))

(def hello (om/factory HelloWorld))

(js/React.render (hello) (gdom/getElement "app"))
```

Try modifying the `"Hello, world!"` string and saving the file. You
should see that the browser updates immediately.

This file presents a lot of new ideas, let's break them down.

### The `ns` form

The very first thing we encounter is the ClojureScript `ns` form. This
declares the current namespace (in other languages you might call this
"module"). We require the `goog.dom`, `om.next`, and `om.dom`
libraries. Other languages might call this process "importing".

### `defui`

The `defui` macro gives us a succinct syntax for declaring Om
components. `defui` supports many of the features of ClojureScript's
`deftype` and `defrecord` with a variety of modifications better
suited to the definition of React components.

If you are familiar with Om, you will notice this is a big
departure. Om Next components are truly plain JavaScript classes. This
component only declares one *JavaScript* `Object` method - `render`.

Finally, in order to create an Om component we must first produce a
factory from the component class. The function returned by
`om.next/factory` has the same signature as pure React components with
the exception that the first argument is usually an immutable
data structure.

### `render`

`render` should return an Om Next or React component. In our case we
return a `div`. Components are usually constructed from two or more
arguments. The first argument will be `props` - the properties that
will customize the component in some way. The remaining arguments will
be `children`, the sub-components hosted by this node.

> **Note**: For a more detailed list of React methods that components
> may implement please refer to the
> [React documentation](https://facebook.github.io/react/docs/component-specs.html)

## Parameterizing Your Components

Like plain React components, Om Next components take `props` as their
first argument and `children` as the remaining ones. Let's modify our
file to look like the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(defui HelloWorld
  Object
  (render [this]
    (dom/div nil (get (om/props this) :title))))

(def hello (om/factory HelloWorld))

(js/React.render
  (hello {:title "Hello, world!"})
  (gdom/getElement "app"))
```

This is slightly more verbose than our previous example but we've
gained abstraction power - the `HelloWorld` component no longer hard
codes a specific string.

For example we can change our code to the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(defui HelloWorld
  Object
  (render [this]
    (dom/div nil (get (om/props this) :title))))

(def hello (om/factory HelloWorld))

(js/React.render
  ;; CHANGED
  (apply dom/div nil
    (map #(hello {:title (str "Hello " %)})
      (range 3)))
  (gdom/getElement "app"))
```

We can render as many `HelloWorld` components as we please and they
all receive custom data. Feel free to change `(range 3)` to something
else. Figwheel will reflect these changes immediately.

## Adding State

We have thus far only seen stateless Om Next components. In order to
do something useful you might think that we would need to introduce
stateful components. This is not the case. In Om Next we introduce
state into the *application* via a global atom. 

In Om Next application state changes are managed by a
*reconciler*. The reconciler accepts novelty, merges it into the
application state, finds all affected components based on their
declared queries, and schedules a re-render.

### Naive Design

We will first examine a naive attempt to introduce application
state. We will later revise this approach to something more robust.

In the following we create a global atom to hold our application
state. We create a reconciler using this atom and then we add a root
for the reconciler to control.

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(def app-state (atom {:count 0}))

(defui Counter
  Object
  (render [this]
    (let [{:keys [count]} (om/props this)]
      (dom/div nil
        (dom/span nil (str "Count: " count))
        (dom/button
          #js {:onClick
               (fn [e]
                 (swap! app-state update-in [:count] inc))}
          "Click me!")))))

(def reconciler
  (om/reconciler {:state app-state}))

(om/add-root! reconciler
  Counter (gdom/getElement "app"))
```

You should now see a counter in your browser window. Clicking on the
button will increase the count in the global atom. This triggers the
reconciler to re-render the root.

> **Note**: `om.next/add-root!` takes a reconciler, a root class and a DOM
> element. Unlike `React.render` we do not instantiate the
> component. The reconciler will do this on our behalf as it may
> need to request data from an endpoint first.

### Global State Coupling

The problem with the program above is that the counter is deeply
coupled to the global state atom. The counter has direct knowledge of
the structure of the state atom. While this may be convenient in this
trivial example, in larger applications this will be an endless supply
of incidental complexity.

Previously Om attempted to mitigate deep coupling to state via the
cursor abstraction. Unfortunately cursors brought problems of their
own. In Om Next, instead of introducing a new abstraction we simply
embrace a time tested way of preventing such state coupling - client
server architecture.

## Client Server Architecture

Om Next encourages a separation between components and code that
actually reads and modifies global state. This design is desirable
even if an Om Next application is entirely client side.

Instead of mixing control logic into components as is often
encountered in React based systems, Om Next moves all state management
into a router abstraction. Components declaratively request data
(reads) from the router. In additions, components do not mutate
application state, instead they request application state transitions
(mutations) and the router will apply the state changes.

Applications designed in this way make it trivial to introduce custom
stores like DataScript without touching or changing any components. In
the case where there is a real remote server component, this
architectural design permits seamless arbitrary partitioning of
application state between local client logic and remote server logic,
that is, *fully transparent synchronization*.

### Routing

A client server architecture requires an established protocol
between the client and the server. This protocol must be able to
describe state transfer (reads) and state transitions (mutations).

Typical web applications already follow this pattern in the form of
[REST](https://en.wikipedia.org/wiki/Representational_state_transfer). However
the unit of composition is incredibly inexpressive, a
[URL](https://en.wikipedia.org/wiki/Uniform_Resource_Locator). Relay
and Falcor have already demonstrated the benefits of moving to a
richer expression of client demands.

Om Next also departs from tradition and embraces a simple data
representation of client demands. This simple data representation
eliminates the problematic tradeoffs present in string based
routing. The data representation is a variant on s-expressions,
[EDN](https://github.com/edn-format/edn).

Because of these important differences, in Om Next we call this process
"parsing" rather than routing. The rationale for this departure will
become more self-evident as the tutorial progresses.

### Parsing & Query Expressions

We will first study parsing in isolation.

Parsing involves handing two kinds of expressions - reads and
mutations. Reads should return the requested application state,
mutations should transition the application state to some new
desired state and describe what changed.

A parser is created from two functions that provides semantics for
reads and mutations:

```clj
(def my-parser (om.next/parser {:read read-fn :mutate mutate-fn}))
```

A parser takes a **query expression** and evaluates it using the
provided read and mutate implementations.

Inspired by [Datomic Pull Syntax](http://docs.datomic.com/pull.html),
a Om Next **query expression** is a vector that enumerates the desired
state reads and state mutations.

For example getting a todo list might look something like the
following:

```clj
[{:todos/list [:todo/created :todo/title]}]
```

Updating a todo list item might look something like the following:

```clj
[(todo/update {:id 0 :todo/title "Get Orange Juice"})]
```

We will interactively parse some query expressions at the Figwheel
REPL to build our intuition of this fundamental Om Next concept.

#### A Read Function

The signature of a read function is `[env key params]`. `env` is a
hash map containing any context necessary to accomplish reads. `key`
is the key that is being requested to be read. Finally `params` is a
hash map of parameters that can be used to customize the read. In many
cases `params` will be empty.

Enter the following at the Figwheel REPL:

```cljs
(in-ns 'om-tutorial.core)
(defn read
  [{:keys [state] :as env} key params]
  (let [st @state]
    (if-let [[_ v] (find st key)]
      {:value v}
      {:value :not-found})))
```

Our read function reads from a `:state` property supplied by the `env`
parameter (we will see how `:state` is supplied shortly). Our function
checks if the application state contains the key. Read functions must
return a hash map containing a `:value` entry (note the `:not-found`
value shown here for missing keys has no special meaning in Om Next).

Let's create a parser:

```clj
(def my-parser (om/parser {:read read}))
```

`my-parser` is just a function. We can now read Om Next **query
expressions**.

```clj
(def my-state (atom {:count 0}))
(my-parser {:state my-state} [:count :title])
;; => {:count 0, :title :not-found}
```

Aha! *We* supply the `env` parameter.

A query expression is *always* a vector. The result of parsing a query
expression is *always* a map.

On the frontend the Om Next reconciler will invoke your parser on your
behalf and pass along the `:state` parameter. When writing a backend
parser you will usually supply `env` yourself.

#### A Mutation Function

Components will not just read data from the application state. They
will want to trigger application state transitions based on user
generated events like mouse clicks, keyboard events, and touch
gestures. We need to supply a function that interprets these requests
for application state transition.

Let's create a simple `mutate` function. The signature is identical to
read functions, however the return value is different. Copy and paste
the following into your Figwheel REPL:

```clj
(in-ns 'om-tutorial.core)
(defn mutate
  [{:keys [state] :as env} key params]
  (if (= 'increment key)
    {:value [:count]
     :action #(swap! state update-in [:count] inc)}
    {:value :not-found}))
```

We first check that the key is a mutation that we actually
implement. If it is we return a map containing two keys, `:value` as
before and `:action` which is a thunk. Mutations should return a
**query expression** for the `:value`. This query expression is
a convenience that communicates what read operations should be
followed by a mutation. Mutations can easily change multiple aspects
of the application (think Facebook "Add Friend"). Adding the
option `:value` result helps users identity stale keys which should be
re-read. The guiding principle here shares common goals with
[HATEOAS](https://en.wikipedia.org/wiki/HATEOAS).

`:action` is a thunk that should transition the application state. *You
should never run side effects in the body of a mutate function
yourself*. Doing so makes it more challenging for Om Next to provide
reliable state management.

Assuming you did the previous REPL interactions now try the following:

```clj
(def my-parser (om/parser {:read read :mutate mutate}))
(my-parser {:state my-state} '[(increment)])
@my-state
;; => {:count 1}
```

## Components With Queries & Mutations

Change `src/om_tutorial/core.cljs` to the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(def app-state (atom {:count 0}))

(defn read [{:keys [state] :as env} key params]
  (let [st @state]
    (if-let [[_ value] (find st key)]
      {:value value}
      {:value :not-found})))

(defn mutate [{:keys [state] :as env} key params]
  (if (= 'increment key)
    {:value [:count]
     :action #(swap! state update-in [:count] inc)}
    {:value :not-found}))

(defui Counter
  static om/IQuery
  (query [this]
    [:count])
  Object
  (render [this]
    (let [{:keys [count]} (om/props this)]
      (dom/div nil
        (dom/span nil (str "Count: " count))
        (dom/button
          #js {:onClick
               (fn [e] (om/transact! this '[(increment)]))}
          "Click me!")))))

(def reconciler
  (om/reconciler
    {:state app-state
     :parser (om/parser {:read read :mutate mutate})}))

(om/add-root! reconciler
  Counter (gdom/getElement "app"))
```

Before we dive in, confirm that the behavior is the same as
before. Open the Chrome JavaScript Console and you will see that every
single transaction was logged by Om Next. The object which initiated
the transaction, the contents of the transaction, and a
[UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)
identifying the state of the application before the transaction was
applied.

Copy and paste one of the UUIDs and try the following at the REPL
(your UUID will be different!):

```clj
(in-ns 'om-tutorial.core)
(om/from-history reconciler
  #uuid "9e7160a0-89cc-4482-aba1-7b894a1c54b4")
;; => {:count 2}
```

> **Fix & Continue**: Om Next automatically records the last 100
> states of the application (the number of recorded states can be
> configured). This feature makes it trivial to take the application
> back to a previous state, fix a bug, re-apply a transaction, and
> continue development without further interruption.

### Digging In

The parsing bits should be familiar from the previous section. We only
had to make three changes.

#### 1. Implement `om.next/IQuery`

Om Next components always declare the data they wish to read. This is
done by implementing a simple protocol `om.next/IQuery`. This method
should return a **query expression**. Note that we added `static`
before the protocol. This is required and ensures the method is
attached to the class (it will also be attached to instances).

This is so that the reconciler can determine the query required to
display the application without instantiating any components at all.

#### 2. Invoke `om.next/transact!`

The counter now calls `om.next/transact!` with the desired transaction
rather than touching the application state directly. This removes the
tight coupling between components and global application state.

#### 3. Provide a parser to the reconciler

The reconciler now takes your custom parser. All application state
reads and mutations will go through your own custom parsing code. The
reconciler will populate the `env` parameter with all the necessary
context needed to make decisions about reads and mutations including
whatever `:state` parameter was provided to the reconciler.

### More about `om.next/transact!`

Components can run transactions. But for development convenience it's
also possible to submit transactions directly to the reconciler.

Try the following at Figwheel REPL:

```clj
(in-ns 'om-tutorial.core)
(om.next/transact! reconciler '[(increment)])
```

You should see the change reflected immediately in the UI. If you have
the Chrome JavaScript Console open you should also see that the
transaction was logged.

While Om Next requires a little bit more work over just banging on
atoms, it should now be apparent that Om Next streamlines the
construction of declarative reusable components. The disciplined
separation of application state reads and mutations means you can
scale up your application without rapid expansion of your complexity
budget.

## Changing Queries Over Time

Once you have declarative queries, it becomes quickly apparent
they are an ideal way to change the behavior of the application. Like
Relay, Om Next fully supports query modification. However it does so
in a manner that does not compromise global time travel.

Change `src/om_tutorial/core.cljs` to look like the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(def app-state
  (atom
    {:app/title "Animals"
     :animals/list
     [[1 "Ant"] [2 "Antelope"] [3 "Bird"] [4 "Cat"] [5 "Dog"]
      [6 "Lion"] [7 "Mouse"] [8 "Monkey"] [9 "Snake"] [10 "Zebra"]]}))

(defmulti read (fn [env key params] key))

(defmethod read :default
  [{:keys [state] :as env} key params]
  (let [st @state]
    (if-let [[_ value] (find st key)]
      {:value value}
      {:value :not-found})))

(defmethod read :animals/list
  [{:keys [state] :as env} key {:keys [start end]}]
  {:value (subvec (:animals/list @state) start end)})

(defui AnimalsList
  static om/IQueryParams
  (params [this]
    {:start 0 :end 10})
  static om/IQuery
  (query [this]
    '[:app/title (:animals/list {:start ?start :end ?end})])
  Object
  (render [this]
    (let [{:keys [app/title animals/list]} (om/props this)]
      (dom/div nil
        (dom/title nil title)
        (apply dom/ul nil
          (map
            (fn [[i name]]
              (dom/li nil (str i ". " name)))
            list))))))

(def reconciler
  (om/reconciler
    {:state app-state
     :parser (om/parser {:read read})}))

(om/add-root! reconciler
  AnimalsList (gdom/getElement "app"))
```

By this now most of the the bits related to parsing should be
familiar to you so we'll focus only on the new ideas. We switched
our `read` function to a multimethod - this makes it easier to add cases
in an open ended way.

We implement the `:animals/list` case. We are finally using `params`, and
in this case we destructure `start` and `end`.

This brings us to the `AnimalsList` component. This component defines
`om.next/IQueryParams` along with `om.next/IQuery`. The `params`
method should return a map of bindings. These will be used to replace
any occurrences of `?some-var` in the actual query.

At the Figwheel REPL try the following (you haven't seen
`om.next/class->any` yet, we'll explain it in a moment):

```clj
(in-ns 'om-tutorial.core)
(om/get-query (om/class->any reconciler AnimalsList))
;; => [:app/title (:animals/list {:start 0, :end 10})]
```

It should be clear now that the params have been bound to the query.

### Change the Query!

Let's change the query by modifying the parameters. This can be done
with `om.next/set-params!`.

```clj
(om/set-params! 
  (om/class->any reconciler AnimalsList)
  {:start 0 :end 5})
```

You should see the UI change immediately. You should also see Om Next
log an event in the Chrome JavaScript Console. Query modifications mutate
the state of program and thus are also recorded into the application
state history log.

Grab one of the UUIDs and you'll see that query state is maintained
when you time travel (again your UUID will not be the one below!):

```clj
(reset! app-state 
  (om/from-history reconciler
    #uuid "e0a07c41-413a-430c-8c91-976a155241c3"))
```

### The Indexer

Om Next supports a first class notion of *identity*. While mounted
React components do provide a kind of identity, Om Next provides a
stronger model that can cut across the mounted component tree. In
addition this model delivers powerful debugging and reasoning
facilities over those found in React itself.

Every reconciler has an indexer. The indexer keeps indexes that
maintain a variety of useful mappings. For example a class to all
mounted components of that class, or prop name and all components that
use that prop name. For example we already say `om.next/class->any`
which is incredibly useful when testing things out at a REPL.

The details of the indexer are not fully ironed out yet, but suffice
to say it is a critical component that both simplifies reconciliation
and enhances interactive development.

### Wrapping Up

This wraps up the Om Next fundamentals. Take a breather and dive into
the second tutorial when you're ready. The following tutorial shows
how Om Next provides a powerful but simple tool for cutting across
the component tree - identity:

* [[Components, Identity & Normalization]]

Also, it may not be fully apparent how the Om Next architecture permits
seamless integration of custom stores or remote services. The
following links provide more details:

* [[DataScript Integration Tutorial]]
* [[Datomic Integration Tutorial]]
