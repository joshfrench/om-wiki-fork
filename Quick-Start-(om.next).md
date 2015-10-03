## Setting Up

Create a new Leiningen project:

```shell
lein new om-tutorial
```

Modify your `project.clj` to look like the following:

```clj
(defproject om-tutorial "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.7.0"]
                 [org.clojure/clojurescript "1.7.122"]
                 [org.omcljs/om "0.9.0-SNAPSHOT"]
                 [figwheel-sidecar "0.4.0" :scope "provided"]])
```

Create a file `script/figwheel.clj`:

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
                :output-to  "resources/public/js/main.js"
                :output-dir "resources/public/js"
                :verbose true}}]})

(ra/cljs-repl)
```

## Markup

Make a file `resources/public/index.html` and include the following:

```html
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title></title>
</head>
<body>
    <div id="app"></div>
    <script src="js/main.js"></script>
</body>
</html>
```

Point your browser at http://localhost:3349.

## Your First Component

Create a file `src/om_tutorial/core.cljs` and edit it to look like the
following:

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
