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

### IQueryParams

### IQuery

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

### react-key

### react-type

### shared

### instrument

### props

### set-state!

### get-state!

### get-rendered-state!

### set-query!

### set-params!

### mounted?

### dom-node

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
