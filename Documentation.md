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

Called only once on an Om component. Implementations should return a map
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

You should only implement this if you really know what you're
doing. Even then you probably shouldn't.

Implementations should return a boolean value. If true then the
component's `om.core/IRender` or `om.core/IRenderState` implementation
will be called.

`next-props` is the next application state that the component is
associated with. `next-state` is the next component local state, it is
always a map.

In your implementation if you wish to detect prop transitions you
must use `om.core/get-props`. This is because your component
constructor function is called with the updated props.

### IWillMount

```clj
(defprotocol IWillMount
  (will-mount [this]))
```

Called once when the component is about to be mounted into the DOM. A
useful place to establish persistent information and control like
core.async channels and go loops.

### IDidMount

```clj
(defprotocol IDidMount
  (did-mount [this]))
```

Called once when the component has been mounted into the DOM. The DOM node associated with this component can be retrieved by using `om.core/get-node`.

This is a good place to initialize persistent information and control
that needs the DOM to be present.

### IWillUpdate

```clj
(defprotocol IWillUpdate
  (will-update [this next-props next-state]))
```

Not called on the first render, will be called on all subsequent
renders. This is a good place to detect and act on state transitions.

`next-props` is the next application state associated with this
component. `next-state` is the next component local state, it is
always a map.

In your implementation if you wish to detect prop transitions you
must use `om.core/get-props`. This is because your component
constructor function is called with the updated props.

### IDidUpdate

```clj
(defprotocol IDidUpdate
  (did-update [this prev-props prev-state]))
```

Called when React has rendered the component into the
DOM. `prev-props` is the previous application state associated with
this component. `prev-state` is the previous component local state, it
is always a map.

### IRender

```clj
(defprotocol IRender
  (render [this]))
```

Called on all changes to application state or component local
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
state as an argument. `state` is always a map, you can use destructuring.

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
keys. Will return the specified piece of component local state. Will
always return pending state.

### get-shared

```clj
(defn get-shared
  ([owner] ...)
  ([owner korks] ...))
```

`owner` is the backing Om component. `korks` is a key or sequence of
keys. It will return data that is shared across the entire render tree. 
You can set global shared data with `om.core/root`.

### root

```clj
(defn root
  ([f value options] ...))
```

`f` is a function returning an instance of `IRender`, `IRenderState`,
a React component, or some value that React knows how to render. `f`
takes two arguments, a root cursor on the application state and the
backing Om component for the root. `value` is either a tree of
associative ClojureScript data structures or an atom wrapping a tree
of associative ClojureScript data structures.

`options` is a map containing any key allowed to `om.core/build`.
Additionally the following keys are allowed/required:

* `:target` (required)
* `:shared` (optional) in order to provide global services
* `:tx-listen` in order to subscribe to all transactions in the application.
* `:path` to specify the path of the cursor into app-state (see [#72](https://github.com/swannodette/om/issues/72))

`om.core/root` is idempotent. You may safely call it multiple
times. Only one Om render loop is ever allowed on a particular DOM
target.

```clj
(om.core/root
  (fn [app-state owner]
    (dom/h1 nil (:text app-state)))
  {:text "Hello world!"}
  {:target (. js/document getElementById "my-app")})   
```

### build

```clj
(defn build
  ([f cursor] ...)
  ([f cursor m] ...))
```

Constructs an Om component. `f` must be a function that returns an
instance of `om.core/IRender` or `om.core/IRenderState`. `f` must take
two arguments - a cursor and the backing Om component usually referred
to as the owner. `f` can take a third argument if `:opts` is
specified in `m`. `cursor` should be an Om cursor onto the application
state. `m` is an optional map of options.

Only the following keys are allowed in `m`.

`:key` - a keyword that will be used to lookup a value in `cursor` to
be used a [React key](http://facebook.github.io/react/docs/multiple-components.html#dynamic-children).

`:react-key` - a value to use as a [React key](http://facebook.github.io/react/docs/multiple-components.html#dynamic-children).

`:fn` - a function to apply to `cursor` before invoking `f`.

`:init-state` - a map of initial state to set on the component.

`:state` - a map of state to merge into the component.

`:opts` - a map of side information that is neither a cursor onto the
application state nor component local state.

### build-all

```clj
(defn build-all
  ([f xs] ...)
  ([f xs m] ...)
```

Conceptually the same as `om.core/build`, the only difference is that
it returns a sequence of Om components. `xs` is a sequence of
cursors. `f` and `m` are the same as `om.core/build`.

### transact!

```clj
(defn transact!
  ([cursor f] ...)
  ([cursor korks f])
  ([cursor korks f tag]))
```

The primary way to transition application state. `cursor` is an Om
cursor into the application state. `f` is a function that will receive
the specified piece of application state. `korks` is an optional
key or sequence of keys to access in the cursor. `om.core/transact!`
can be given additional args to pass to `f`. `tag` is optional
information to tag the transaction. `tag` should be either a keyword
or a vector whose first element is a `keyword`.

```clj
(transact! cursor :text (fn [_] "Changed this!"))
```

### update!

```clj
(defn update!
  ([cursor v] ...)
  ([cursor korks v] ...)
  ([cursor korks v tag] ...))
```

Similar to `om.core/transact!` but just sets a cursor to a new value,
analagous to `reset!` for atoms.

```clj
(update! cursor assoc-in [:text] "Changed this!")
```

### get-node

```clj
(defn get-node
  ([owner ref] ...))
```

`owner` is the backing Om component. `ref` is a JavaScript String that
references a DOM node. This functionality is identical to the
`ReactComponent.getDOMNode` in React.

### set-state!

```clj
(defn set-state!
  ([owner korks v] ...))
```

Sets component local state. `owner` is the backing Om
component. `korks` is a key or sequences of keys. `v` is the value to
set. Will trigger a Om re-render.

### get-render-state

```clj
(defn get-render-state
  ([owner] ...)
  ([owner korks] ...))
```

Returns rendered component local state. `owner` is the backing Om
component. `korks` is an optional key or sequences of keys. Similar to
`om.core/get-state` except always returns the rendered state. Useful
for detecting state transitions.

### graft

```clj
(defn graft [value cursor]
  ...)
```

Sometimes it's useful to create stateful Om components that do not
manipulate or correspond to application state. However these component
still need to be a part of the render tree, `om.core/graft` supports
this. Given any `value` and a `cursor` `om.core/graft` will return a
new cursor that can be used to construct an Om component that is
attached to the render tree.

```clj
(build my-widget (graft {:text "I'm not in the app state!"} cursor))
```

# om.dom

The dom functions map directly to the DOM api presented by React. For
example `React.DOM.div` becomes `om.dom/div`. The arguments are
exactly the same as React, the first argument is `props`, all
subsequent arguments are `children`.
