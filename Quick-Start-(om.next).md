> **WARNING**: This page is under heavy active development

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
JavaScript Console** menu.

In the JavaScript Console you should see `Hello, world!` printed out.

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

### defui

The most important bit is `defui`. The `defui` macro gives us a
succinct syntax for declaring Om components. `defui` supports many of
the features of ClojureScript's `deftype` and `defrecord` with a
variety of modifications better suited to the definition of React
components.

If you are familiar with Om, you will notice this is a big
departure. Om Next components are truly plain JavaScript classes. This
component only declares one JavaScript object method - `render`.

### render

`render` should return an Om or React component. In our case we return
a `div`. Components are usually constructed from two or more
arguments. The first argument will be `props` - the properties that
will customize the component in some way. The remaining arguments will
be `children` - the components should should be rendered the parent
component you are currently rendering.

> **Note**: for a more detailed list of methods that components can
> implement please refer to the
> [React documetation](https://facebook.github.io/react/docs/component-specs.html)

### Next Steps

While it's entertaining that your modifications to the source file are
reflected immediately into the browser this is not a practical path
for application customization that isn't style oriented. We want to be
able to customize a components data. For this we must learn how to
parameterize components.

## Parameterizing Your Components

Like plain React components Om components take props as their first
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
gained significantly in abstraction power. The `HelloWorld` component
is no longer hard coded to a specific string.

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
  (apply dom/p nil
    (map #(hello {:title (str "Hello " %)})
      (range 3)))
  (gdom/getElement "app"))
```
