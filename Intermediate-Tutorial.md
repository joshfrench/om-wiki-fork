For this next tutorial we're going to get more
ambitious. We're going to build an Om front end to a simple
[Ring](http://github.com/ring-clojure) application that talks to
[Datomic](http://datomic.com) for persistence. You can of course swap
in another database, but Datomic is particularly easy to use from
Clojure and also provides time travel capabilities making it quite
nice to pair with Om.

First download a copy of
[Datomic Free](http://my.datomic.com/downloads/free). Unzip it and run
the following inside the directory:

```
bin/transactor config/samples/free-transactor-template.properties
```

This will start up the Datomic transactor.

Now in some other directory run the following:

```
lein new om-async-tut om-async
```

`cd` into `om-async` and launch a Lein repl with `lein repl`. Once the
repl is up run the following:

```
user=> (use '[om-async.util])
nil
user=> (init-db)
:done
```

The tutorial database is now exists and is populated. Quit the
REPL. Let's start the auto building process:

```
lein cljsbuild auto dev
```

We don't have time to cover all the details of Ring or Datomic here
but hopefully you get the basic idea. If you're curious about Datomic
I highly recommend Jonas Enlund's
[tutorial](http://www.learndatalogtoday.org) and the
[Day of Datomic](http://github.com/Datomic/day-of-datomic) tutorial.

Ok that was a bit of work, let's read some code. Open
`src/clj/om-async/core.clj` in Light Table. Typed
"Shift-Command-Enter" to evaluate the entire file. This will start up
the web server. You should be able to point your browser at
`http://localhost:8080`. If you open the JavaScript console you should
see `"Hello world!"` printed.

We have the usual namespace stuff:

```clj
(ns tut-test.core
  (:require [ring.util.response :refer [file-response]]
            [ring.adapter.jetty :refer [run-jetty]]
            [ring.middleware.edn :refer [wrap-edn-params]]
            [compojure.core :refer [defroutes GET PUT]]
            [compojure.route :as route]
            [compojure.handler :as handler]
            [datomic.api :as d]))
```

Then we establish a connection to Datomic:

```clj
(def uri "datomic:free://localhost:4334/tut_test")
(def conn (d/connect uri))
```

Instead of [JSON](http://www.json.org) as a data format we'll instead
use [EDN](http://github.com/edn-format/edn). So we write a little helper
for the EDN middleware we'll be using:

```clj
(defn generate-response [data & [status]]
  {:status (or status 200)
   :headers {"Content-Type" "application/edn"}
   :body (pr-str data)})
```

Now let's take a look at classes:

```clj
(defn classes []
  (let [db (d/db conn)
        classes
        (vec (map #(d/touch (d/entity db (first %)))
               (d/q '[:find ?class
                      :where
                      [?class :class/id]]
                 db)))]
    (generate-response classes)))
```

This just finds all the classes and returns an EDN response.

Now we have `update-class`. It finds a class by id and modifies the
title:

```clj
(defn update-class [id params]
  (let [db    (d/db conn)
        title (:class/title params)
        eid   (ffirst
                (d/q '[:find ?class
                       :in $ ?id
                       :where 
                       [?class :class/id ?id]]
                  db id))]
    (d/transact conn [[:db/add eid :class/title title]])
    (generate-response {:status :ok})))
```

We then have our routes:

```clj
(defroutes routes
  (GET "/" [] (index))
  (GET "/classes" [] (classes))
  (PUT "/class/:id/update"
    {params :params edn-params :edn-params}
    (update-class (:id params) edn-params))
  (route/files "/" {:root "resources/public"}))
```

Finally we add our EDN middleware and start our server:

```clj
(def app
  (-> routes
      wrap-edn-params))

(defonce server
  (run-jetty #'app {:port 8080 :join? false}))
```

Let's look at the client side portion now, open
`src/cljs/om-async/core.cljs` in Light Table. The `ns` form should
look familiar, and we enable `console.log` printing.

```clj
(ns om-ring.core
  (:require [cljs.reader :as reader]
            [goog.events :as events]
            [goog.dom :as gdom]
            [om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true])
  (:import [goog.net XhrIo]
           goog.net.EventType
           [goog.events EventType]))

(enable-console-print!)
```

We're going to use simple callbacks in this tutorial instead of
relying on core.async.

Now we write simple utility for making async requests to the server
with EDN:

```clj
(def ^:private meths
  {:get "GET"
   :put "PUT"
   :post "POST"
   :delete "DELETE"})

(defn edn-xhr [{:keys [method url data on-complete]}]
  (let [xhr (XhrIo.)]
    (events/listen xhr goog.net.EventType.COMPLETE
      (fn [e]
        (on-complete (reader/read-string (.getResponseText xhr)))))
    (. xhr
      (send url (meths method) (when data (pr-str data))
        #js {"Content-Type" "application/edn"}))))
```

## Speculative UI Programming
