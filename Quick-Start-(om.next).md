> **DO NOT EDIT THIS PAGE**: This page is under heavy active development. 

## Introduction

Om Next is a uniform yet extensible approach to building networked
interactive applications. By providing a structured discipline over
the management of application state, Om Next narrows the scope of
incidental complexity often found in user interface development. The
Om Next discipline is founded upon immutable data structures,
declarative data specifications, and a simple convention for routing
data access and mutations.

Om Next borrows ideas liberally from
[Facebook's Relay](https://facebook.github.io/relay/),
[Netflix's Falcor](http://netflix.github.io/falcor/), and
[Cognitect's Datomic](http://www.datomic.com). If you are not familiar
with these technologies, fear not, this tutorial makes few
assumptions. You will be guided from the most basic declarative
component to advanced components which transparently synchronize
state divided over local and remote data sources.

## Setting Up

This tutorial uses [Leiningen](http://leiningen.org),
[Figwheel](https://www.youtube.com/watch?v=j-kj2qwJa_E), and
[Google Chrome](http://www.google.com/chrome/). Leiningen is a
standard tool for managing Clojure and ClojureScript library
dependencies. Figwheel is a ClojureScript build tool and REPL that
enables an expressive live programming model well suited for
interactive application development. Figwheel also plays well with
text editors that make traditional REPL integration more challenging.

You can of course use any web browser, but this tutorial only includes
relevant instructions for Chrome to avoid tangential material.

Create a new project and switch into it:

```shell
mkdir om-tutorial
cd om-tutorial
```

Inside your project directory create a `project.clj`:

```clj
touch project.clj
```

Make it look like the following:

```clj
(defproject om-tutorial "0.1.0-SNAPSHOT"
  :description "My first Om program!"
  :dependencies [[org.clojure/clojure "1.7.0"]
                 [org.clojure/clojurescript "1.7.122"]
                 [org.omcljs/om "0.9.0-SNAPSHOT"]
                 [figwheel-sidecar "0.4.0" :scope "provided"]])
```

A Leiningen `project.clj` file simply allows you to declare a variety
of properties about your project. In our the case the most important
is the list of `:dependencies`.

Now create a file `script/figwheel.clj`.

```shell
mkdir script
touch script/figwheel.clj
```

Change `script/figwheel.clj` to look like the following:

```clj
(require '[figwheel-sidecar.repl :as r]
         '[figwheel-sidecar.repl-api :as ra])

(ra/start-figwheel!
  {:figwheel-options {}
   :build-ids ["dev"]
   :all-builds
   [{:id "dev"
     :figwheel true
     :source-paths ["src"]
     :compiler {:main 'om-tutorial.core
                :asset-path "js"
                :output-to "resources/public/js/main.js"
                :output-dir "resources/public/js"
                :verbose true}}]})

(ra/cljs-repl)
```

This file describes how to build your ClojureScript project and starts
a REPL. If you are new to ClojureScript you may find this file a bit
overwhelming. If you would like to know more, after this tutorial you
may want to work through the ClojureScript
[Quick Start](https://github.com/clojure/clojurescript/wiki/Quick-Start)
to re-inforce fundamental ClojureScript concepts encountered in this
tutorial.

## Markup

We now need to provide some basic markup to host our ClojureScript
application.

Make a file `resources/public/index.html`:

```shell
mkdir -p resources/public
touch resources/public/index.html
```

Change the contents of this file to the following:

```html
<!DOCTYPE html>
<html>
    <head lang="en">
        <meta charset="UTF-8">
        <title>Om Tutorial!</title>
    </head>
    <body>
        <div id="app"></div>
        <script src="js/main.js"></script>
    </body>
</html>
```

## Checkpoint

Create a file `src/om_tutorial/core.cljs`:

```shell
mkdir -p src/om_tutorial
touch src/om_tutorial/core.cljs
```

Edit its contents to look like the following:

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(println "Hello world!")
```

Start Figwheel:

```clj
lein run -m clojure.main script/figwheel.clj
```

For enhanced REPL behavior it's recommended that you install
[rlwrap](http://utopia.knoware.nl/~hlub/uck/rlwrap/). Under OS X this
can be easily done with [brew](http://brew.sh).

If you have `rlwrap` installed you can then launch with:

```clj
rlwrap lein run -m clojure.main script/figwheel.clj
```

Point your browser at
[http://localhost:3449](http://localhost:3449). You should see a blank
page with the title "Om Tutorial!" visible on your browser tab.

Open the Chrome developer tools with the **View > Developer >
JavaScript Console** menu. In the JavaScript Console you should see
`Hello, world!` printed out.

Let's write our first component!

## Your First Component

Create a file `src/om_tutorial/core.cljs`:

```shell
mkdir -p src/om_tutorial
touch src/om_tutorial/core.cljs
```

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
should see that the browser immediately updates.

This file presents a lot of new ideas, let's break them down.

### The `ns` form

The very first thing we encounter is the ClojureScript `ns` form. This
declares the current namespace (in other languages you might call this
"module"). We require the `goog.dom`, `om.next`, and `om.dom`
libraries. Other languages might call this processs "importing".

### `defui`

The most important bit is `defui`. The `defui` macro gives us a
succinct syntax for declaring Om components. `defui` supports many of
the features of ClojureScript's `deftype` and `defrecord` with a
variety of modifications better suited to the definition of React
components.

If you are familiar with Om, you will notice this is a big
departure. Om Next components are truly plain JavaScript classes. This
component only declares one *JavaScript* `Object` method - `render`.

Finally, in order to create an Om component we must first produce a
factory from the component class. The function return by
`om.next/factory` has the same signature as plain React component with
the exception that the first argument is usually an immutable
data structure.

### `render`

`render` should return an Om or React component. In our case we return
a `div`. Components are usually constructed from two or more
arguments. The first argument will be `props` - the properties that
will customize the component in some way. The remaining arguments will
be `children` - the components should should be rendered the parent
component you are currently rendering.

> **Note**: For a more detailed list of React methods that components
> may implement please refer to the
> [React documetation](https://facebook.github.io/react/docs/component-specs.html)

## Parameterizing Your Components

Like plain React components, Om components take props as their first
argument and children as the remaining ones. Let's modify our file to
look like the following:

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
  (apply dom/p nil
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
application state, finds all affected components, and schedules a
re-render.

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

Om Next simply embraces a client server architecture to enforce a
separation between components and code that modifies global
state. This design is embraced even if an Om Next application is
entirely client side.

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

Adopting a client server architecture means there must some
established protocol between the client and the server. This protocol
must be able to describe state transfer (reads) and state transitions
(mutations).

Typical web applications already follow this pattern in the form of
REST. However the unit of composition is incredibly inexpressive - a
URL. Relay and Falcor have already demonstrated the benefits of moving
to a richer expression of client demands.

Om Next also departs from tradition and embraces a simple data
representation of client demands. This simple data representation
eliminates the problematic tradeoffs present in string based
routing. The data representation is a variant on s-expressions - EDN.

Because of these important differences, in Om Next we call this process
"parsing" rather than routing. The rationale for this departure will
become more and more self-evident as the tutorial progresses.

### Parsing & Query Expressions

We will first study parsing in isolation.

Parsing involves handing two kinds of expressions - reads and
mutations. Reads should return the requested application state,
mutations should transition the application state to some new
desired state and describe what changed.

A parser is created from two functions that provides semantics for
reads and mutations.

```clj
(def my-parser (om.next/parser {:read read-fn :mutate mutate-fn}))
```

A parser takes a **query expression** and evaluates it using the
provided read and mutate implementations.

Inspired by [Datomic Pull Syntax](http://docs.datomic.com/pull.html),
an Om Next *query expression* is a vector that enumerates the desired
state reads and state mutations.

For example to getting a todo list might look something like the
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

Enter the following into your Figwheel REPL:

```cljs
(defn read
  [{:keys [state] :as env} key params]
  (let [st @state]
    (if-let [[_ v] (find st key)]
      {:value v}
      {:value :not-found})))
```

All our read function does is read from a `:state` property supplied
by the `env` parameter. We will see how `:state` is supplied
shortly. Our read simply checks if the application state contains the
key. Your read function must return a hash map containing a `:value`
entry.

Let's create a parser:

```clj
(def my-parser (om/parser {:read read}))
```

`my-parser` is just a function. We can now read Om Next "query" 
expressions.

```clj
(def my-state (atom {:count 0}))
(my-parser {:state my-state} [:count :title])
;; => {:count 0, :title :not-found}
```

Aha! *We* supply the `env` parameter.

A query expression is *always* a vector. The result of parsing a query
expression is *always* a map.

On the front end the reconciler will invoke your parser on your behalf
and pass along the `:state` parameter.

#### A Mutate Function

Components will not just read data from the application state. They
will want to trigger application state transitions based on user
generated events like mouse clicks, keyboard events, and touch
gestures. We need to supply a function that interprets these requests
for application state transition.

Let's create a simple `mutate` function. The signature is identical to
read functions, however the return value is different. Copy and paste
the following into your Figwheel REPL:

```clj
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
*query expression* for the `:value`. This is query expression is just
a convenience to communicate what read operations should be followed
by a mutation. Mutations can easily change multiple aspects of the
application (think Facebook "Add Friend"), and this helps identity
stale keys which should be re-read.

`:action` is a thunk that should transition the application state. You
should never run side effects in the body of a mutate function
yourself. Doing so makes it more challenging for Om Next to provide
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
               (fn [e] (om/transact this '[(increment)]))}
          "Click me!")))))

(def reconciler
  (om/reconciler
    {:state app-state
     :parser (om/parser {:read read :mutate mutate})}))

(om/add-root! reconciler
  Counter (gdom/getElement "app"))
```

Before we dive in confirm that the behavior is the same as
before. Open the JavaScript Console and you will see that every single
transaction was logged by Om Next. The object which initiated the
transaction, the contents of the transaction, and a UUID identifying the
state of the application before the transaction was applied.

Copy and paste one of the UUIDs and try the following at the REPL,
*note that your UUID will be different!*:

```clj
(om/from-history reconciler
  #uuid "9e7160a0-89cc-4482-aba1-7b894a1c54b4")
;; => {:count 2}
```

> **Fix & Continue**: Om Next automatically records the last 100
> states of the application (the number of recorded states can be
> easily configured). This feature makes it trivial to take the
> application back to a previous state, fix a bug, and re-apply a
> transaction.

### Digging In

The parsing bits should be familiar from the previous section. We only
had to make three changes.

#### 1. Implement `om.next/IQuery`

Om Next components always declare the data they wish to read. This is
done by implementing a simple protocol `om.next/IQuery`. This method
should return a *query expression*. Note that we added a `static`
before the protocol. This is required and ensure the method is
attached to the class (it will also be attached to instances).

This is so that the reconciler can determine the query required to
display the application without instantiating any components.

#### 2. Invoke `om.next/transact`

The counter now calls `om.next/transact` with the desired transaction
rather than touching the application state directly. This removes the
tight coupling between components and global application state.

#### 3. Provide a parser to the reconciler

The reconciler now takes your custom parser. All application state
reads and mutations will go through your own custom parsing code. The
reconciler will populate the `env` parameter with all the necessary
context needed to make decisions about reads and mutations.
