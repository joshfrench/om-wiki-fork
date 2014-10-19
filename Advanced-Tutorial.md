The following assumes familiarity with [core.async](https://github.com/clojure/core.async).

While the beginner and intermediate tutorials are good for getting a
feel for the Om primitives they are not representative guides on how to
structure an Om application for scale.

Cursors are a powerful abstraction but often not the tool to reach for
when structuring larger applications. In the worst case cursors are a
quick trip back to non-modular programs where every component has full
knowledge and direct dependence on the global state of the
application.

In order to recover modularity we must reach for other tools. We'll
see how core.async channels and reference cursors allow us to regain
modularity in complex applications.

The following is a loose set of patterns, real applications may need
more or less than what is described. Still the following is a good
starting point for doing your own explorations around organizing
larger Om programs.

* The Request Channel
* The Publish & Notification Channels
* Reference Cursors

## The Request Channel

### The Problem

In many applications you have some component several levels deep in
the render tree. The question is, how do we get data to this component
without having to pass data to everyone in between the root of the
application and where the component actually lives?

### Solution

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
        pub-chan   (pub notif-chan :topic)]

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
               :per-page per-page
               :res res})
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
      (let [events (sub (:notif-chan (get-shared owner)) :hello (chan))]
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

## Reference Cursors

Building truly modular and reusable components is a time consuming
task. In most cases this level of abstraction is outside the time
constraints of the problem at hand. Introducing channels to do
something as simple as modifying a collection introduces manual
resource management (of the go loops) and a considerable amount of
ceremony.

Om 0.8.0-alpha1 now offers a new tool - reference cursors. Reference
cursors are like cursors but they have the novel propery of
representing an identity in the application state that you can
*observe*. Observation is similar to React event handlers - there is no need
to worry about observing multiple times nor is there any concern about
needing to unobserve. Reference cursors fully support the Om state
management model - time travel properties are preserved.

Using reference cursors is simple and natural and allows components to be
organized around a shared API they can just call out to.

For example suppose we want a logical collection from a vector in our
app state. We can now write an API for it using reference cursors:

```cljs
(def app-state
  (atom {:items [{:text "cat"} {:text "dog"} {:text "bird"}]}))

(defn items []
  (om/ref-cursor (:items (om/root-cursor app-state))))
```

In a subview we can now simply write the following. 

```cljs
(defn sub-view [{:keys [text]} owner]
  (reify
    om/IRender
    (render [_]
      (let [xs (om/observe owner (items))]
        (dom/div nil
          (dom/h2 nil text)
          (apply dom/ul nil
            (map #(dom/li nil (:text %)) xs)))))))
```

This is very natural and straightforward, the component did not need
to take its data from a parent nor did it need to receive it via a
channel.

Now imagine we have a parent view that looks like the following:

```cljs
(defn main-view [_ owner]
  (reify
    om/IRender
    (render [_]
      (let [xs (items)]
        (dom/div nil
          (om/build sub-view {:text "View A"})
          (om/build sub-view {:text "View B"})
          (dom/button
            #js {:onClick
                 (fn [e] (om/transact! xs #(assoc % 1 {:text "zebra"})))}
            "Switch To Zebra!"))))))
```

This parent is blissfully unaware that two children components also
rely on the same logical collection. Notice that it can `transact!` on
the reference cursor the same as any other cursor.

Many existing convulted patterns around the manipulation of
application data can be circumvented simply by using reference cursors
and designing a simple API that any component can call into.

## Further Investigation Needed

The three patterns above eliminate many cases where we may have relied
on passing around too much of the application state for
coordination. Nothing in the tutorial is conclusive, rather treat it
as small set of guidelines to build up more expressive functionality
suitable for the large application you are building.