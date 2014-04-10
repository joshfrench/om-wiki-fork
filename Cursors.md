## Overview

Cursors are Om’s way to manage components’ mutable state. Cursors split one big mutable atom into small, mutable, in-sync with the original atom parts.

In Om, you keep all your application state in a single atom. Components, however, do not care about whole atom, they work with parts of it. Imagine text input component: it operates on a single string, renders it and communicates value back once user changed it.

Cursors also specify component’s dependencies on a state. Each component get cursors at construction time and will automatically re-render when value underneath its cursor changes.

At their core, cursors just keep link to the root atom and path to the part of data they point to. Sub-cursors are produced by just refining the path part.

## Accessing cursors

Cursors behave differently during render phase and outside of it.

During render phase, you treat cursor as a value, as a regular map or vector. Cursors support all the same interfaces PersistentMap and PersistentVector support, so you can get-in, check for keys, etc. 

Outside render phase, you cannot treat cursor as a value anymore. Deref it (@) and work with the value returned. Deref returns actual value beneath cursor: real map or vector.

## Creating sub-cursors

Root component gets cursor created from atom itself. This atom and all cursors derived from root cursor stay in sync during all modifications.

You can create sub-cursor from cursor by just calling `get` or `get-in` on it (works only during render phase). There’s a gotcha: if the value returned by `get` is map or vector, you’ll get a sub-cursor pointing at it. If it’s a primitive value, you’ll get primitive value, not a cursor.

In short:

    (def state (atom {:x 1, :y [:a :b [:c]]}))

    (defn root-view [cursor _]
      ...
      (render [_]
        (get cursor :x) // 1, value
        (get cursor :y) // [:a :b [:c]], cursor with path [:y]
        (get-in cursor [:y 0]) // :a, value
        (get-in cursor [:y 2]) // [:c], cursor with path [:y 2]

This means that you cannot create a component that depends on a single string (like text-input). But you can write a component that depends on a vector of one string:

    (def state (atom {:name ["Igor"]}))

    (defn text-input [cursor _]
      ...
      (render [_]
        (dom/input #js {:value (first cursor)}))) 

    ...
    (render [_]
      (om/build text-input (:name app-cursor)))
         
## Making changes via cursors

Cursors can propagate changes back to the original atom. For that purpose, `transact!` call is used:

    (def state (atom {:click-counter [0]}))

    (defn btn-view [cursor _]
      (reify om/IRender
        (render [_]
          (om/div nil
            (om/span nil (str "Clicked " (get cursor 0) " times"))
            (om/button
              #js {:onClick (fn [_]
                              (om/transact! cursor [0] inc))})))))

## Consistency and observability

`transact!` is allowed during both inside and outside of render phase. But when inside render, all transact!-ed changes will not be visible until next frame (next render). That’s to keep each rendered frame consistent app-wise.

When rendering, we want to see on screen consistent view built on top of one consistent snapshot of application state. That’s why changes made in the middle of the render phase are not immediately visible for not-yet-rendered components.

Outside the render phase, deref returns actual value, and call to `om.core/value` will return last rendered value.

## Using multiple cursors

Component might depend on a several cursors. Just collect them into a map or a vector and pass it instead of a single cursor:

    (def state (atom {:courses [...], :classes [...], ...}))

    (render [_]
      (om/build table-view {:rows (:courses cursor),
                            :cols (:classes cursor)})) 
