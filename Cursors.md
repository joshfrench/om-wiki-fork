## Overview

Cursors are Oms way to manage a components mutable state. Cursors split one big mutable atom into small, mutable, in-sync with the original atom parts.

In Om, you keep all your application state in a single atom. Components, however, do not care about the whole state, they work with parts of it. Imagine a text input component: it operates on a single string, renders it and communicates a value back once the user changed it.

Cursors also specify a components dependencies on the state. Each component gets cursors at construction time and will automatically re-render when the value underneath its cursor changes.

At their core, cursors just keep a link to the root atom and a path to the part of data they point to. Sub-cursors are produced by refining the path.

## Accessing cursors

Cursors behave differently during render phase and outside of it.

During render phase, you treat a cursor as a value, as a regular map or vector. Cursors support all the same interfaces `PersistentMap` and `PersistentVector` support, so you can `get-in`, check for keys, etc. 

Outside of the render phase, you cannot treat cursors as values. Deref it (@) and work with the value returned. Deref returns the actual value beneath the cursor: a map or a vector.

## Creating sub-cursors

Root components get cursors created from the atom itself. The atom and all cursors derived from the root cursor stay in sync during all modifications.

You can create sub-cursors from cursors by just calling `get` or `get-in` on them (works only during render phase). There's a gotcha: if the value returned by `get` is a map or a vector, you’ll get a sub-cursor pointing to it. If it's a primitive value, you'll get a primitive value, not a cursor.

In short:

```clj
(def state (atom {:x 1, :y [:a :b [:c]]}))

(defn root-view [cursor _]
  ...
  (render [_]
     (get cursor :x) ;; 1, value
     (get cursor :y) ;; [:a :b [:c]], cursor with path [:y]
     (get-in cursor [:y 0]) ;; :a, value
     (get-in cursor [:y 2]) ;; [:c], cursor with path [:y 2]
```

This means that you cannot create a component that depends on a single string (like text-input). But you can write a component that depends on a vector of one string:

```clj
(def state (atom {:name ["Igor"]}))

(defn text-input [cursor _]
  ...
  (render [_]
    (dom/input #js {:value (first cursor)}))) 

...
(render [_]
  (om/build text-input (:name app-cursor)))
```

## Making changes via cursors

Cursors can propagate changes back to the original atom. For that purpose, `transact!` calls are used:

```clj
(def state (atom {:click-counter [0]}))

(defn btn-view [cursor _]
  (reify om/IRender
    (render [_]
      (om/div nil
        (om/span nil (str "Clicked " (get cursor 0) " times"))
        (om/button
          #js {:onClick (fn [_]
                          (om/transact! cursor [0] inc))})))))
```

## Consistency and observability

`transact!` is allowed during and outside of the render phase. When called during the render phase, no `transact!`-ed changes will be visible until the next frame is rendered (next render). That's to keep each rendered frame consistent app-wise.

When rendering, we want to show a consistent view built from one consistent snapshot of the application state. That's why changes made in the middle of the render phase are not immediately visible for not-yet-rendered components.

Outside the render phase, `deref` returns the current value, while calls to `om.core/value` will return the last rendered value.

## Using multiple cursors

Components might depend on several cursors. Just collect them into a map or a vector and pass it instead of a single cursor:

```clj
(def state (atom {:courses [...], :classes [...], ...}))

(render [_]
  (om/build table-view {:rows (:courses cursor),
                        :cols (:classes cursor)})) 
```