The following assumes familiarity with [core.async](https://github.com/clojure/core.async).

While the beginner and intermediate tutorials are good for getting a
feel for the Om primitives they are not representative guides on how to
structure an Om application for scale.

Cursors are a powerful abstraction but often not the tool to reach for
when structuring larger applications. In the worst case cursors are a
quick trip back to non-modular programs where every component has full
knowledge and direct dependence on the global state of the
application.

In order to recover modularity we must reach for something
else. We'll see how core.async channels allow us to regain modularity
in complex applications.

## A Structure

The following is a loose pattern, real applications may need more or
less than what is described. Still the following is a good starting
point for doing your own explorations around organizing larger Om
programs.

The loose pattern is organized around 3 types of shared channels.

* The Request Channel
* The Publish & Notification Channels

## The Request Channel

### The Problem

In many applications you have some component several levels deep in
the render tree. The question is, how do we get data to this component
without having to pass data to everyone in between the root of the
application and where the component actually lives?

### The Pattern

The Request Channel pattern allows us to avoid passing data through
components uninterested in the data. The strategy is a familiar one -
treat a component as if it were a client browser. Instead of
receiving its data directly it will request it when it mounts. In
order for this to work we need to estable a global service channel
that can process requests from different parts of your application.

Let us assume we'll have a table view somewhere whose contents will be
requested via an XHR request. In the following we'll just mock out the
remote data for simplicity.

The global `serve` will be a multimethod. By using multimethods we can
extensibly serve different "client" components in our
application. Again this is a familiar pattern - a routing table.

Note that `serve` takes a map that represents the request analogous to
how you might process form parameters or a JSON API request server
side. In order to respond to the client one of the arguments is a
*response channel*. We simply write out the data the client needs onto
that channel:

```cljs
(def remote-data (vec (map #(str "Item " %) (range 100))))

(defmulti serve :op)

(defmethod serve :data
  [{:keys [start per-page res]}]
  (put! res (subvec remote-data start per-page)))
```

When we establish our application root we'll do something like the
following. You can ignore `notif-chan` and `pub-chan` for now, they
form the other pattern we'll talk about momentarily:

```cljs
(defn main []
  (let [req-chan   (chan)
        notif-chan (chan)
        pub-chan   (state/mem-pub notif-chan :topic)]

    ;; server loop
    (go
      (while true
        (serve (<! req-chan))))

    (om/root
      (fn [app-cursor owner]
        (reify
          om/IRender
          (render [_]
            ...)))
      app-state
      {:shared {:req-chan   req-chan
                :notif-chan notif-chan
                :pub-chan   pub-chan}
       :target (. js/document (getElementById "app"))})))
```

Your table view component might look something like the following:

```cljs
(defn table-view [_ owner]
  (reify
    om/IInitState
    (init-state [_]
      {:start 0 :per-page 10 :data nil})
    om/IDidMount
    (did-mount [_]
      (let [res (chan)]
        (go
          (>! (:req-chan (om/get-shared owner))
              {:op :data :start start
               :per-page per-page})
          (om/set-state! owner :data (<! res)))))
    om/IRenderState
    (render-state [_ {:keys [data]}]
      (if data
        (apply dom/table nil (map table-row data))
        (dom/div "Loading ...")))))
```

Now any component that needs data can get at it. No need to share
global state all over your application.

## The Publish & Notification Channels

### The Problem

Component B and Component C do not share a common parent Component A
yet they need to communicate to each other. How do we synchronize
state without again passing the global app state around so that B & C
can mutate some value they synchronize on? Component B needs to tell
Component C something important without having a direct reference to
Component C.

### The Publish Channel

Component B should publish a topic onto `pub-chan`.

```cljs
(defn widget-b [data owner]
  (reify
    om/IRender
    (render [_]
      (dom/button
        #js {:onClick
             #(put! (:pub-chan (get-shared owner))
                    {:topic :hello :data "Hi there!"})}
        "Click me!"))))
```

But how will Component C get this message?

### The Notification Channel

When Component C mounts it subscribes to topics it cares about. This
looks something like the following:

```cljs
(defn widget-c [data owner]
  (reify
    om/InitState
    (init-state [_]
      {:message nil})
    om/IDidMount
    (did-mount [_]
      (let [events (sub (:notif-chan (get-shared owner)) :hello)]
        (go
          (loop [e (<! events)]
            (om/set-state! owner :message (:data e))
            (recur))))))
    om/IRenderState
    (render-state [_ {:keys [message]}]
      (if message
        (dom/p nil message)
        (dom/p nil "Waiting ... waiting ... waiting ..."))))
```

## Further Investigation Needed

The two patterns above eliminate many cases where we may have relied
on application state for coordination. Nothing in the tutorial is
conclusive, rather treat it as small set of guidelines to build up
more expressive functionality suitable for the large application you
are building.