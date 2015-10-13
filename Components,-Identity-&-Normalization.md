> **DO NOT EDIT THIS PAGE**: This page is under heavy active development. 

## Introduction

The [[Quick Start (om.next)]] introduced the fundamentals, now we'll
explore more deeply how Om Next permits the coordination of state
across the component without introducing boilerplate or additional
complexity.

Before we begin we need to look at normalization and identity.

### Normalization

Consider the following bit of data. Let's assume for a moment that
this is the data returned from a remote service based on a root
query. Can you identify what is problematic about this representation
in the context of a user interface?

```clj
(def init-date
  {:list/one [{:name "John" :points 0}
              {:name "Mary" :points 0}
              {:name "Bob"  :points 0}]
   :list/two [{:name "Mary" :points 0 :age 27}
              {:name "Gwen" :points 0}
              {:name "Jeff" :points 0}]})
```

The issue is that the value `"Mary"` appears *twice*. In this case
this value represents the same logical entity. While this is fantastic
for rendering this representation is extremely problematic for
updates. You would need to track all the places where `"Mary"` occurs
and update them by hand.

If you give an Om Next reconciler some with data without wrapping it
in an atom, the reconciler assumes the data has not yet been
normalized. It will use the root query to normalize the data. For
example assume the following bit of code:

```clj
(def reconciler
  (om/reconciler
    {:state  init-data
     :parser (om/parser {:read read :mutate mutate})}))
```

If you deref'ed the `reconciler` you would see the following:

```clj
{:list/one
 [[:person/by-name "John"]
  [:person/by-name "Mary"]
  [:person/by-name "Bob"]],
 :list/two
 [[:person/by-name "Mary"]
  [:person/by-name "Gwen"]
  [:person/by-name "Jeff"]],
 :person/by-name
 {"John" {:name "John", :points 0},
  "Mary" {:name "Mary", :points 0, :age 27},
  "Bob" {:name "Bob", :points 0},
  "Gwen" {:name "Gwen", :points 0}, 
  "Jeff" {:name "Jeff", :points 0}}}
```

Notice that all the data has been de-duplicated. In place of the
original values in `:line/one` and `:list/two` we instead have a
vector that can get be used with `get-in` to get the actual
data. `"Mary"` now appears only once and all the fields are
preserved. We can now update `"Mary"` in one location and expect that
all parts of our user interface that need it will update accordingly.

But how was Om Next able to automatically normalize the data?

Surprise, surprise, co-located queries with a little bit of help from
the components them selves!

### Identity

Co-located queries actually give us an incredible amount of
information with regards to intent.

```clj
(defui Person
  static om/Ident
  (ident [this {:keys [name]}]
    [:person/by-name name])
  static om/IQuery
  (query [this]
    '[:name :points :age])
  Object
  (render [this]
    ;; ...
    ))
```

For example if you get a query from a component via
`om.next/get-query` you'll see that the query has some useful
metadata:

```clj
(-> Person om.next/get-query meta)
;; => {:component om-tutorial.core/Person}
```

This means that we know what component is associated with what data in
the denormalized response. Now all we need to do is implement a
protocol `om.next/Ident` to normalize the data. `om.next/Ident` takes
props to client unique key. This key has a purpose beyond
normalization - we can also use this key to know which components are
backed by the same data and trivially keep them in sync.

## Appendix

The complete source for this tutorial.

```clj
(ns om-tutorial.core
  (:require [goog.dom :as gdom]
            [om.next :as om :refer-macros [defui]]
            [om.dom :as dom]))

(enable-console-print!)

(def init-data
  {:list/one [{:name "John" :points 0}
              {:name "Mary" :points 0}
              {:name "Bob"  :points 0}]
   :list/two [{:name "Mary" :points 0 :age 27}
              {:name "Gwen" :points 0}
              {:name "Jeff" :points 0}]})

;; -----------------------------------------------------------------------------
;; Parsing

(defmulti read om/dispatch)

(defmethod read :default
  [{:keys [state] :as env} key params]
  (let [st @state]
    (if-let [[_ value] (find st key)]
      {:value value}
      {:value :not-found})))

(defn get-people [state key]
  (let [st @state]
    (into [] (map #(get-in st %)) (get st key))))

(defmethod read :list/one
  [{:keys [state] :as env} key params]
  {:value (get-people state key)})

(defmethod read :list/two
  [{:keys [state] :as env} key params]
  {:value (get-people state key)})

(defmulti mutate om/dispatch)

(defmethod mutate 'points/increment
  [{:keys [state]} _ {:keys [name]}]
  {:action
   (fn []
     (swap! state update-in
       [:person/by-name name :points]
       inc))})

(defmethod mutate 'points/decrement
  [{:keys [state]} _ {:keys [name]}]
  {:action
   (fn []
     (swap! state update-in
       [:person/by-name name :points]
       #(let [n (dec %)] (if (neg? n) 0 n))))})

;; -----------------------------------------------------------------------------
;; Components

(defui Person
  static om/Ident
  (ident [this {:keys [name]}]
    [:person/by-name name])
  static om/IQuery
  (query [this]
    '[:name :points :age])
  Object
  (render [this]
    (println "Render Person" (-> this om/props :name))
    (let [{:keys [points name foo] :as props} (om/props this)]
      (dom/li nil
        (dom/label nil (str name ", points: " points))
        (dom/button
          #js {:onClick
               (fn [e]
                 (om/transact! this
                   `[(points/increment ~props)]))}
          "+")
        (dom/button
          #js {:onClick
               (fn [e]
                 (om/transact! this
                   `[(points/decrement ~props)]))}
          "-")))))

(def person (om/factory Person {:keyfn :name}))

(defui ListView
  Object
  (render [this]
    (println "Render ListView" (-> this om/path first))
    (let [list (om/props this)]
      (apply dom/ul #js {:style {:border "1px solid black"}}
        (map person list)))))

(def list-view (om/factory ListView))

(defui RootView
  static om/IQuery
  (query [this]
    (let [subquery (om/get-query Person)]
      `[{:list/one ~subquery} {:list/two ~subquery}]))
  Object
  (render [this]
    (println "Render RootView")
    (let [{:keys [list/one list/two]} (om/props this)]
      (apply dom/div nil
        [(dom/h2 nil "List A")
         (list-view one)
         (dom/h2 nil "List B")
         (list-view two)]))))

(def reconciler
  (om/reconciler
    {:state  init-data
     :parser (om/parser {:read read :mutate mutate})}))

(om/add-root! reconciler
  RootView (gdom/getElement "app"))
```
