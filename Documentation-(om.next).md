#### Table of Contents

#### om.next

#### om.dom

* [Props](#props)
* [render-to-str](#render-to-str)

### defui

```clj
(defui MyComponent
  Object
  (render [this]
    (div nil "Hello, world!")))
```

Macro for defining components. `defui` creates a JavaScript class that
inherits from `React.Component`. `defui` is like `deftype` but there
is no support for defining fields. In addition there is special
handling of the "static" protocols `om.next/Ident`, `om.next/IQuery`
and `om.next/IQueryParams`.

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

### remove-root!

### transact!

### parser

### ref->components

### ref->any

### class->any

### merge!

### reconciler

### reconciler?

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
