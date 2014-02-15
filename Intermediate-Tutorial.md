For this next tutorial we're going to get a bit more
ambitious. We're going to build an Om frontend to a simple
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

Now in some other directory run the following to generate the tutorial
from the `om-async-tut` Lein template:

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

The tutorial database now exists and is populated. Quit the
REPL. Let's start the auto building process:

```
lein cljsbuild auto dev
```

We don't have time to cover all the details of Ring or Datomic here
but hopefully you get the basic idea. If you're curious about Datomic
I highly recommend Jonas Enlund's
[tutorial](http://www.learndatalogtoday.org) and the
[Day of Datomic](http://github.com/Datomic/day-of-datomic) tutorial.

Let's read some code. Open `src/clj/om-async/core.clj` in Light
Table. Type "Shift-Command-Enter" to evaluate the entire file. This
will start up the web server. You should be able to point your browser
at `http://localhost:8080`. If you open the JavaScript console you
should see `"Hello world!"` printed.

At the top of the file we have the usual namespace stuff:

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

We then define our main route handler `index`:

```clj
(defn index []
  (file-response "public/html/index.html" {:root "resources"}))
```

Next, we'll define some schema for our database:

```clj
(def schema
  [{:db/doc                "Class ID"
    :db/ident              :class/id
    :db/valueType          :db.type/long
    :db/cardinality        :db.cardinality/one
    :db/id                 #db/id[:db/part/db]
    :db.install/_attribute :db.part/db}

   {:db/doc                "Class name"
    :db/ident              :class/name
    :db/valueType          :db.type/string
    :db/cardinality        :db.cardinality/one
    :db/id                 #db/id[:db/part/db]
    :db.install/_attribute :db.part/db}])
```

And transact it, so that it's available for use in our transactions and queries:

```clj
(d/transact conn schema)
```

Instead of [JSON](http://www.json.org) as a data format we'll instead
use [EDN](http://github.com/edn-format/edn). We write a little helper
for the EDN middleware we'll be using:

```clj
(defn generate-response [data & [status]]
  {:status (or status 200)
   :headers {"Content-Type" "application/edn"}
   :body (pr-str data)})
```

Now let's take a look at `classes`:

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

We then have our `routes`:

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
look familiar, and we enable `console.log` printing:

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

First we write a simple utility for making async requests to the
server with EDN. We rely on Google Closure to deal with cross browser
issues:

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

Next we define our `app-state`. In this case we won't have initial
data because we will need to fetch it from the server. Still we need
to provide the basic structure:

```clj
(def app-state
  {:classes []})
```

Next we need the `display` style helper from the previous tutorial:

```clj
(defn display [show]
  (if show
    #js {}
    #js {:display "none"}))
```

We are now going to write a version of the `editable` component that
does not require hacking around the differences between JavaScript
primitive strings and JavaScript String objects. The following is
strongly recommended over extending native types to `ICloneable`.

Instead of `editable` taking a string cursor from the application state,
instead it will take some larger piece of data and a key to locate the
string to edit. `handle-change` now looks like this:

```clj
(defn handle-change [e data edit-key owner]
  (om/transact! data edit-key (fn [_] (.. e -target -value))))
```

Instead of using channels because `editable` is so simple and to
demonstrate alternative approaches, `editable` will notify its parent
component with a callback when editing is complete:

```clj
(defn end-edit [text owner cb]
  (om/set-state! owner :editing false)
  (cb text))
```
This is `editable`:

```clj
(defn editable [data owner {:keys [edit-key on-edit] :as opts}]
  (reify
    om/IInitState
    (init-state [_]
      {:editing false})
    om/IRenderState
    (render-state [_ {:keys [edit-text editing]}]
      (let [text (get data edit-key)]
        (dom/li nil
          (dom/span #js {:style (display (not editing))} text)
          (dom/input
            #js {:style (display editing)
                 :value text
                 :onChange #(handle-change % data edit-key owner)
                 :onKeyPress #(when (and (om/get-state owner :editing)
                                         (== (.-keyCode %) 13))
                                (end-edit text owner on-edit))
                 :onBlur (fn [e]
                           (when (om/get-state owner :editing)
                             (end-edit text owner on-edit)))})
          (dom/button
            #js {:style (display (not editing))
                 :onClick #(om/set-state! owner :editing true)}
            "Edit"))))))
```

Take the time to read through and understand this code. It's almost
identical to the previous version with the exception that we take
`data` which is a map cursor. We also take an `edit-key` and the
`on-edit` callback via `opts`.

We now have a generic editable component that doesn't require us to hack
JavaScript strings.

When the user commits a change we want to communicate this back to the
server:

```clj
(defn on-edit [id title]
  (edn-xhr
    {:method :put
     :url (str "class/" id "/update")
     :data {:class/title title}
     :on-complete
     (fn [res]
       (println "server response:" res))}))
```

Our `classes-view` will load the data from server on
`om.core/IWillMount`.

```clj
(defn classes-view [app owner]
  (reify
    om/IWillMount
    (will-mount [_]
      (edn-xhr
        {:method :get
         :url "classes"
         :on-complete #(om/transact! app :classes (fn [_] %))}))
    om/IRender
    (render [_]
      (dom/div #js {:id "classes"}
        (dom/h2 nil "Classes")
        (apply dom/ul nil
          (map
            (fn [class]
              (let [id (:class/id class)]
                (om/build editable class
                  {:opts {:edit-key :class/title
                          :on-edit #(on-edit id %)}})))
            (:classes app)))))))

(om/root classes-view app-state
  {:target (gdom/getElement "classes")})
```

That's it. Save your file and refresh. You should see the list of
classes loaded from the server. You should be able to edit a class
title. Press enter to commit the title change. Refresh your browser
and you should see that the change persisted.

Both your backend and frontend can support sophisticated operations on
application state history without incurring incidental complexity.

## Speculative UI Programming

Many difficulties in mordern UI programming arise from the fact that
we're often building distributed systems - clients and servers. Not
only do we have to keep clients and servers in sync but we have to
gracefully handle those cases where sync is not possible.

It's quite common in modern applications to make an optimistic commit,
and then discover that synchronization cannot happen - perhaps
because of a backend bug, undefined behavior in the API
, or commonly the complete lack of network connectivity. In this
distributed setting UI programming becomes a speculative endeavor.

One well known form of speculative UI programming is multi-level
undo. It's telling that many single page user interfaces on the web do
not support this kind of undo - it's hard enough to implement in a
desktop application. Add the distributed component and most developers
raise the white flag.

However an immutable UI approach like Om and immutable databases like
Datomic provide the infrastructure to make these problems very
approachable even in a distributed context.

We'll come back to this but first let's explore modularity in the
context of synchronizing application state.

### Modularity

Ideally we can build our UIs without considering synchronization. And
when we come to the problem of synchronization we should be able to add
it to our application without changing any code we have already written.

This is effectively the goal of
[om-sync](http://github.com/swannodette/om-sync), a reusable
synchronization component for Om. You can take your existing
component, put together an application and impose a synchronization
policy without changing your own code.

Let's modify the previous tutorial so that we can demonstrate this now.

#### Updating the Server

Make sure the Datomic transactor is running as before.

First we need to change our `project.clj` to include a dependency on
`om-sync`.

```clj
:dependencies [[org.clojure/clojure "1.5.1"]
               [org.clojure/clojurescript "0.0-2156"]
               [ring/ring "1.2.1"]
               [org.clojure/core.async "0.1.267.0-0d7780-alpha"]
               [om "0.4.0-SNAPSHOT"]
               [om-sync "0.1.0-SNAPSHOT"] ;; <=== ADD THIS
               [compojure "1.1.6"]
               [fogus/ring-edn "0.2.0"]
               [com.datomic/datomic-free "0.9.4532"]]
```

Lets update the server side code to uniformly handle EDN requests.

After `generate-response` let's add `get-classes`:

```clj
(defn get-classes [db]
  (->> (d/q '[:find ?class
              :where
              [?class :class/id]]
          db)
       (map #(d/touch (d/entity db (first %))))
       vec))
```

We want to be able to properly initialize the client state as well as
give it information so that it can communicate back with the server:

Let's make a request handler called `init`:

```clj
(defn init []
  (generate-response
     {:classes {:url "/classes" :coll (get-classes (d/db conn))}}))
```

We're also going to add a handler for creating classes but we're going
to return a 500 for now to demonstrate some neat properties of Om:

```clj
(defn create-class [params]
  {:status 500})
```

Let's refactor `update-class` a bit:

```clj
(defn update-class [params]
  (let [id    (:class/id params)
        db    (d/db conn)
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

Let's also refactor the `classes` request handler:

```clj
(defn classes []
  (generate-response (get-classes (d/db conn))))
```

And let's provide the new routes:

```clj
(defroutes routes
  (GET "/" [] (index))
  (GET "/init" [] (init))
  (POST "/classes" {params :edn-params} (create-class params))
  (PUT "/classes" {params :edn-params} (update-class params))
  (route/files "/" {:root "resources/public"}))
```

Evaluate everything with Control-Shift-ENTER, that's it for our server
side code. Let's update the client code.

If you have a `lein cljsbuild auto <build-id>` process running, kill
it and restart it. First we need to modify the namespace form. Since
we'll be using `om-sync` we clean up some things:

```clj
(ns om-async.core
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [cljs.core.async :as async :refer [put! chan alts!]]
            [goog.dom :as gdom]
            [om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true]
            [om-sync.core :refer [om-sync]]
            [om-sync.util :refer [tx-tag edn-xhr]]
            [clojure.data :as data]))
```

In order for `om-sync` to work you need modify how you call
`om.core/root`. `om-sync` needs to be able to subscribe to the
application's transactions so that it can observe transactions that
are relevant to it.

We use core.async to publish a channel that components like `om-sync`
can subscribe to. This is done by make the channel a global service via
`:shared`. We also make an EDN request to get our initial state using
the new `init` route we wrote on the backend.

```clj
(let [tx-chan (chan)
      tx-pub-chan (async/pub tx-chan (fn [_] :txs))]
  (edn-xhr
    {:method :get
     :url "/init"
     :on-complete
     (fn [res]
       (reset! app-state res)
       (om/root app-view app-state
         {:target (gdom/getElement "classes")
          :shared {:tx-chan tx-pub-chan}
          :tx-listen
          (fn [tx-data root-cursor]
            (put! tx-chan [tx-data root-cursor]))}))}))
```

We create a channel `tx-chan`. We then we make a subscribeable channel
`tx-pub-chan` so that `om-sync` instances call call
`cljs.core.asyn/sub` on it. We request the initial state of the
application from the server. Here use some `om.core/root` options we
have not seen before. `:shared` allows us to provide a global service
to any components in our application. `:tx-listen` is a callback that
will be invoked anytime the application state transitions. We simply
put this information into `tx-chan`.

While technically we don't need to change `editable` to make this work
we're going to so that `editable` is a better citizen:

First modify `end-edit`:

```clj
(defn end-edit [data edit-key text owner cb]
  (om/set-state! owner :editing false)
  (om/transact! data edit-key (fn [_] text) :update)
  (cb text))
```

The changes here are most around the fact that we now invoke
`om/transact!`. The interesting part is that this `om/transact!`
supplies a tag, `:update`. `om-sync` specifically listens for
`:create`, `:update`, and `:delete` tags. We could make this work
without the transaction tag, but not as simply.

`editable` need to change to accommodate the new `end-edit` signature:

```clj
(defn editable [data owner {:keys [edit-key] :as opts}]
  (reify
    om/IInitState
    (init-state [_]
      {:editing false})
    om/IRenderState
    (render-state [_ {:keys [edit-text editing on-edit]}]
      (let [text (get data edit-key)]
        (dom/li nil
          (dom/span #js {:style (display (not editing))} text)
          (dom/input
            #js {:style (display editing)
                 :value text
                 :onChange #(handle-change % data edit-key owner)
                 :onKeyPress #(when (and (om/get-state owner :editing)
                                         (== (.-keyCode %) 13))
                                (end-edit data edit-key text owner on-edit))
                 :onBlur (fn [e]
                           (when (om/get-state owner :editing)
                             (end-edit data edit-key text owner on-edit)))})
          (dom/button
            #js {:style (display (not editing))
                 :onClick #(om/set-state! owner :editing true)}
            "Edit"))))))
```

We also want to allow people to add classes. We know that the backend
doesn't support this but we'll try it on the front end anyway to
demonstrate the capabilities of `om-sync`.

```clj
(defn create-class [classes owner]
  (let [class-id-el   (om/get-node owner "class-id")
        class-id      (.-value class-id-el)
        class-name-el (om/get-node owner "class-name")
        class-name    (.-value class-name-el)
        new-class     {:class/id class-id :class/title class-name}]
    (om/transact! classes [] #(conj % new-class)
      [:create new-class])
    (set! (.-value class-id-el) "")
    (set! (.-value class-name-el) "")))
```

We update `classes-view` to include some new input fields:

```clj
(defn classes-view [classes owner]
  (reify
    om/IRender
    (render [_]
      (dom/div #js {:id "classes"}
        (dom/h2 nil "Classes")
        (apply dom/ul nil
          (map
            (fn [class]
              (let [id (:class/id class)]
                (om/build editable class
                  {:opts {:edit-key :class/title}})))
            classes))
        (dom/div nil
          (dom/label nil "ID:")
          (dom/input #js {:ref "class-id"})
          (dom/label nil "Name:")
          (dom/input #js {:ref "class-name"})
          (dom/button
            #js {:onClick (fn [e] (create-class classes owner))}
           "Add"))))))
```

All this should look pretty straightforward by now.

Finally we write `app-view` add this right before the `om/root`
expression:

```clj
(defn app-view [app owner]
  (reify
    om/IWillUpdate
    (will-update [_ next-props next-state]
      (when (:err-msg next-state)
        (js/setTimeout #(om/set-state! owner :err-msg nil) 5000)))
    om/IRenderState
    (render-state [_ {:keys [err-msg]}]
      (dom/div nil
        (om/build om-sync (:classes app)
          {:opts {:view classes-view
                  :filter (comp #{:create :update :delete} tx-tag)
                  :id-key :class/id
                  :on-success (fn [res tx-data] (println res))
                  :on-error
                  (fn [err tx-data]
                    (reset! app-state (:old-state tx-data))
                    (om/set-state! owner :err-msg
                      "Ooops! Sorry something went wrong try again later."))}})
         (when err-msg
           (dom/div nil err-msg))))))
```

We render `classes-view` via `om-sync`, this is specified by the
`:view` option. We supply a `:filter` so that individual key presses
don't get synchronized - this is why we tagged the `end-edit`
`om/transact!` above with `:update`. `om-sync` needs to know which
property represents an identifier the server understands in order to
send incremental updates. Finally we establish `:on-success` and
`:on-error` updates.

`:on-error` is the most interesting here - if there's a server error
we just roll back the application state and present an error message.

This is speculative UI programming, we can optimistically update the
client yet trivially roll back if something goes wrong. And all of
this can be done without polluting your actual components with
synchronization or undo logic.

Refresh your browser and make sure your JavaScript Console is
open. Edit a class. You should see `{:status ok}` if it
worked. Refresh the browser, you will see that the state persisted.

Now put some random data into the new input fields and press
"Add". You should see the data briefly appear in the list and then
disappear when we roll back the state because we could not succeed. An
error message appears.

Hopefully this tutorial demonstrates how Om allows application
developers to focus on their problem domain and remove sources of
incidental complexity.

Having made it this far you can probably read through the `om-sync`
source yourself and hopefully be inspired to send a pull request to
improve so it can be leveraged in more contexts.
