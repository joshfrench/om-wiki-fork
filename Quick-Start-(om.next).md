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

Create a new Leiningen project and switch into it:

```shell
lein new om-tutorial
cd om-tutorial
```

Inside your project directory modify `project.clj` to look like the
following:

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
[http://localhost:3349](http://localhost:3349). You should see a blank
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

(def hello (om/create-factory HelloWorld))

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

(def hello (om/create-factory HelloWorld))

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

(def hello (om/create-factory HelloWorld))

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

We will first examine at a naive attempt to introduce application
state.

In the following we create a global atom to hold our application
state. We create a reconciler using this atom and then we add a root
for the reonciler to control.

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
trivial example, in larger applications this will be a endless supply
of incidental complexity.

Previously Om attempted to mitigate deep coupling to state via the
cursor abstraction. Unfortunately cursors brought problems of their
own. In Om Next, instead of introducing a new abstraction we simply
embrace a time tested way of preventing such state coupling.

## Client Server Architecture

Om Next simply embraces a client server architecture to enforce a
separation between components and code that modifies global
state. This design is embraced even if an Om Next application is
entirely client side.

Applications designed in this way make it trivial to introduce more
sophisticated stores like DataScript. In the case where there is a
real remote server component, architecting Om Next applications in
this way permits arbitrarily partitioning of state between local
client logic and remote server logic, i.e. *fully transparent
synchronization*.

### Routing

Adopting a client server architecture means there must a protocol
between the client and the server. This protocol must satisfy how
state is transferred to the client and how the client can communicate
state transitions to the server.

One of the most successful and popular concrete design patterns to
emerge is REST due to its extreme simplicity - the unit of composition
is a URL.

Om Next however departs from tradition and embraces a simple data
representation rather than a string. This simple data representation
eliminates the problematic tradeoffs present in string based routing
while delivering the benefits found in systems like Relay and Falcor.

In Om Next we call this process "parsing" rather than routing. The
reason will become apparent as the tutorial progresses.

## Parsing & Query Expressions

We will first study parsing in isolation.

Parsing involves handing two kinds of operations - reads and
mutations. Reads should return the requested application state,
mtuations should transition the application state to some new
desired state.

A parser is created from two functions that provides semantics for
reading and mutation.

```clj
(def my-parser (om.next/parser {:read read-fn :mutate mutate-fn}))
```

A parser takes a **query expression** and evaluates it using your
read and mutate implementations.

Inspired by [Datomic Pull Syntax](http://docs.datomic.com/pull.html),
an Om Next *query expression* is a vector that enumerates desires
state reads and state mutations.

For example to get a todo list might look something like the
following:

```clj
[{:todos/list [:todo/created :todo/title]}]
```

To update a todo list item might look something like the following:

```clj
[(todo/update {:todo/title "Get Orange Juice"})]
```

This might sound a bit abstract so let's just create a simple read
function and a parser and see how it works in practice.

### A Read Function

The signature of a read function is `[env key params]`. `env` is a
hash map containing any context necessary to accomplish reads. `key`
is the key that is being requested to be read. Finally `params` is a
hash map of parameters that can be used to customize the read. In many
cases `params` will be empty.

Let's make this concrete by trying some things at the Figwheel REPL:

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
(def my-state (atom {:count 0})
(my-parser {:state my-state} [:count :title])
;; => {:count 0, :title :not-found}
```

Aha! *We* supply the `env` parameter.

A query expression is *always* a vector. The result of parsing a query
expression is *always* a map.

On the front end the reconciler will invoke your parser on your behalf
and pass along the `:state` parameter.

### A Mutate Function

```clj

```

## Queries

## Mutations

## Remoting
