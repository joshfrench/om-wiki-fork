# om.core

## Life Cycle Protocols

The Om life cycle protocols map more or less directly to the life
cycle API present in Facebook's React.

Om component functions return `reify` instances that implement the Om
life cycle protocols.

```clj
(defn my-widget [data owner]
  (reify
    om/IRender
    (render [_]
      (dom/h1 nil "Hello world!"))))
```

It's important to understand the `my-widget` will be called many
times. Thus it's an anti-pattern to wrap `reify` in a `let` to
allocate stateful entities like core.async channels. Stateful entities
should be stored in component local state - `om.core/IInitState` or
`om.core/IWillMount` are good places to do this.

### IInitState

```clj
(defprotocol IInitState
  (init-state [this]))
```

Called only once a Om component. Implementations should return a map
of initial state.

```clj
(defn my-widget [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:text "Hello world!"})
    om/IRenderState
    (render [_ state]
      (dom/h1 nil (:text state)))))
```

### IShouldUpdate

```clj
(defprotocol IShouldUpdate
  (should-update [this next-props next-state]))
```

You should implement this if you really know what you're doing. Even
then you probably shouldn't.

Implementations should return a boolean value. If true then the
components `om.core/IRender` or `om.core/IRenderState` implementation
will be called.

`next-props` is the next application state that the component is
associated with. `next-state` is the next component local state, it is
always a map.

### IWillMount

```clj
(defprotocol IWillMount
  (will-mount [this]))
```

Called once when the component is about to be mounted into the DOM. A
useful place to establish persistent information like core.async
channels.

### IDidMount

```clj
(defprotocol IDidMount
  (did-mount [this node]))
```

Called once when the component has been mounted into the DOM.

### IWillUpdate

```clj
(defprotocol IWillUpdate
  (will-update [this next-props next-state]))
```

Not called on the first render, will be called on all subsequent
renders. This is a good place to detect and act on state transitions.

### IDidUpdate

```clj
(defprotocol IDidUpdate
  (did-update [this prev-props prev-state root-node]))
```

Called when React has rendered the component into the DOM.

### IRender

```clj
(defprotocol IRender
  (render [this]))
```

Called on all changed to application state or component local
state. Must return an Om component, a React component, or a some value
that React knows how to render.

If you implement `om.core/IRender` you should not implement
`om.core/IRenderState`.

### IRenderState

```clj
(defprotocol IRenderState
  (render-state [this state]))
```

The only difference between `om.core/IRender` and
`om.core/IRenderState` is that `IRenderState` implementations get the
state as an argument.

If you implement `om.core/IRenderState` you should not implement
`omc.core/IRender`.

## Functions

### get-props

```clj
(defn get-props [owner]
   ...)
```

`owner` is the backing Om component. `om.core/get-props` returns the
*cursor* into the application state associated with the Om
component. A *cursor* is a piece of the application state that knows
how to update itself.

### get-state

```clj
(defn get-state
  ([owner] ...)
  ([owner korks] ...))
```

`owner` is the backing Om component. `korks` is a key or sequence of
keys. Will return the specified piece of component local state.

### get-shared

```clj
(defn get-shared
  ([owner] ...)
  ([owner korks] ...))
```

`owner` is the backing Om component. `korks` is a key or sequence of
keys. Will return data that shared across the entire render tree. You
can set global shared data with `om.core/root`.

### root

```clj
(defn root
  ([value f target] ...)
  ([value shared f target] ...))
```

`value` is either an tree of associative ClojureScript data structures
or a atom wrapping a tree of associative ClojureScript data
structures. `f` is a function return an instance of `IRender`,
`IRenderState`, a React component, or some value that React knows how
to render. `f` takes two arguments, a root cursor on the application
state and the backing Om component for the root. `shared` is an
optional map of values shared across the entire render tree. `target`
is the DOM node to install the Om render loop on.

`om.core/root` is idempontent. You may safely call it multiple
times. Only one Om render loop is ever allowed on a particular DOM
target.

```clj
(om.core/root
  {:text "Hello world!"}
  (fn [app-state owner]
    (dom/h1 nil (:text app-state)))
  (. js/document getElementById "my-app"))   
```

### build

```clj
(defn build
  ([f cursor] ...)
  ([f cursor m] ...))
```

### build-all

### transact!

### update!

### read

### get-node

### set-state!

### get-render-state

### bind

### graft

# om.dom

The dom functions map directly to the DOM api presented by React. For
example `React.DOM.div` becomes `om.dom/div`. The arguments are
exactly the same as React, the first argument is `props`, all
subsequent arguments are `children`.
