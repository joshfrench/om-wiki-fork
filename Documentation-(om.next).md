#### Table of Contents

#### om.next

* [defui](#defui)
* [Ident](#Ident)
* [IQuery](#IQuery)
* [IQueryParams](#IQueryParams)
* [get-query](#get-query)
* [factory](#factory)
* [component?](#component?)
* [react-key](#react-key)
* [react-type](#react-type)
* [props](#props)
* [set-state!](#set-state!)
* [get-state](#get-state)
* [get-rendered-state](#get-rendered-state)
* [set-query!](#set-query!)
* [set-params!](#set-params!)
* [mounted?](#mounted?)
* [dom-node](#dom-node)
* [reconciler](#reconciler)
* [reconciler?](#reconciler?)
* [from-history](#from-history)

#### om.dom

* [Props](#props)
* [render-to-str](#render-to-str)

### defui

```clj
(defui MyComponent
  Object
  (componentDidMount [this]
                     (.log js/console "did mount"))
  (render [this]
          (div nil "Hello, world!")))

```

Macro for defining components. `defui` creates a JavaScript class that
inherits from `React.Component`. `defui` is like `deftype` but there
is no support for defining fields. In addition there is special
handling of the "static" protocols `om.next/Ident`, `om.next/IQuery`
and `om.next/IQueryParams`. The React component specs and lifecycle methods are available at the [react docs](https://facebook.github.io/react/docs/component-specs.html). 

### Ident

```clj
(defui MyComponent
  static om/Ident
  (ident [this props]
    [:some/key (:some/id props)])
```

A protocol for identity resolution. This protocol is used to solve two
problems. First, in the case where initial state or state novelty is
supplied in a denormalized form. Second, when making an association
from a logical entity to multiple component instances. The first case
simplifies the problem of updating data while the later simplifies
keep multiple view of the same data in sync.

### IQuery

```clj
(defui MyComponent
  static om/IQuery
  (query [this]
    [:prop-a :prop-b])
  Object
  (render [this]
    (div nil "Hello, world!")))
```

A protocol for declaring queries. This method should always return
a vector. The query may include quoted symbols that start with `?`. If
bindings for these query varialbes are supplied via `IQueryParams`
they will replace the symbols.

### IQueryParams

```clj
(defui MyComponent
  static om/IQueryParams
  (params [this]
    {:start 0 :end 10})
  static om/IQuery
  (query [this]
    '[(:list/items {:start ?start :end ?end})])
  Object
  (render [this]
    (div nil "Hello, world!")))
```

Define params to bind to the query. Should be a map of keywords that
match query variables present in the query.

### get-query

```clj
(om.next/get-query MyComponent)
```

Returns the bound query for the component.

### factory

```clj
(om.next/factory MyComponent
  {:keyfn :id :validator my-validator})
```

Create a factory function from an Om component. Can optionally supply
`:keyfn` - this should produce the React `key` property from the
component props. Can also supply `:validator`, a function which should
`assert` that the props are valid.

### component?

```clj
(om.next/component? 1) ;; false
```

Returns true if the argument is a compoennt.

### react-key

```clj
(om.next/react-key some-component)
```

Returns the React `key`.

### react-type

```clj
(om.next/react-type some-component)
```

Returns the component constructor. Works even if the component has not
been mounted.

### shared

### instrument

### props

```clj
(defui MyComponent
  Object
  (render [this]
    (let [{:keys [foo bar]} (om/props this)]
      ;; ...
      )))
```

Get the immutable props for an Om component.

### set-state!

```clj
(om.next/set-state! some-component {:foo :bar})
```

Set component local state. Will schedule the component for re-render.

### get-state

```clj
(om.next/get-state some-component)
```

Return the component local state. Will be the latest state, not the
rendered component state.

### get-rendered-state

```clj
(om.next/get-rendered-state some-component)
```

Return the rendered component local state.

### set-query!

```clj
(om.next/set-query! some-component [:foo :baz]})
```

Mutate the query of a component. Schedules the component for re-render.

### set-params!

```clj
(om.next/set-params! some-component {:start 5 :end 10})
```

Mutate the query params of a component. Schedules the component for re-render.

### mounted?

```clj
(om.next/mounted? some-component)
```

Returns true if the component is mounted.

### dom-node

```clj
(om.next/dom-node some-component)
```

Return the DOM node associated with a component.

### add-root!

```clj
(om.next/add-root! reconciler
  MyRootComponent (goog.dom/getElement "app"))
```

Add a DOM root for the reconciler to control. The first argument is a
reconciler, the second a root component, and the last, a DOM node target.

### remove-root!

```clj
(om.next/add-root! reconciler (goog.dom/getElement "app"))
```

Given a reconciler and DOM target node, remove the DOM target from the
reconciler's control.

### transact!

```clj
(om.next/transact! some-component
  `[(todo/update `{:title "Get Milk!"})])
```

Transition the application state. `transact!` only takes two
arguments, the component and a **query expression** that includes
mutation.

### parser

```clj
(om.next/parser {:read read-fn :mutate mutate-fn})
```

Construct a parser from the supplied configuration map. The map should
only have two keys:

* `:read` - a function of three arguments `[env key params]` that
  should return a valid parse result map. This map should only contain
  `:value`, `:remote` or both. If `:value` is supplied will be used to
  rewrite a value in the resulting tree. If `:remote` is supplied will
  be used to determine the remote query when running the parser in
  remote mode.
* `:mutate` - a function of three arguments `[env key params]` that
  should return a valid parse mutation result map. This map should
  only contain the keys valid for `:read` functions in addition to
  a `:action` key. This should be a function of zero arguments that
  applies the requested mutation.

Returns a function of up to three arguments. The first argument should
be the `env` map. The second argument should be the **query
expression**. The final optional argument sets the parse mode to
remote, defaults to `false`.

### ref->components

```clj
(om.next/ref->components reconciler [todo/by-id 0])
```

A development time helper. Given an Om ref return all the components
that match.

### ref->any

```clj
(om.next/ref->any reconciler [todo/by-id 0])
```

A development time helper. Given an Om ref return the first component
that matches.

### class->any

```clj
(om.next/class->any reconciler SomeClass)
```

A development time helper. Given a class return a matching component.

### merge!

### reconciler

```clj
(om.next/reconciler
  {:state app-state
   :parser my-parser})
```

Construct a reconciler based on the supplied configuration. The
configuration can be a map with the following keys:

* `:state` - the application state. If not an atom the reconciler will
normalize the data with the query supplied by the root component.
* `:parser` - a parser

### reconciler?

```clj
(om.next/reconciler? x)
```

Returns true if the argument is a reconciler instance, false otherwise.

### from-history

```clj
(om.next/from-history reconciler
  #uuid "894e7a30-a5b8-4751-8bd6-a51aa122a919")
```

When mutating the application state via transactions or query
modifications, Om will log a UUID associated with an application state
before the change was applied. You can use the UUID to recover a
previous application state. Useful for "Fix & Continue" development
workflows.