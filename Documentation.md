# om.core

## Life Cycle Protocols

The Om life cycle protocols map more or less directly to the life
cycle API present in Facebook's React.

### IInitState

### IShouldUpdate

### IWillMount

### IDidMount

### IWillUpdate

### IDidUpdate

### IRender

## Functions

### get-props

### get-state

### get-shared

### root

### build

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
