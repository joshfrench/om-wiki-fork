What follows is an Om conceptual guide.

## Tangible Data

Object Oriented programming as a paradigm has many real benefits but
one of the worst plagues it has inflicted on programming culture is
obscuring data. Functional programming is not a silver bullet but its
emphasis on unadorned data is a guiding light.

No models.

Rather than introducing a middle man, Om allows programmers to build
user interfaces over unadorned data. Still, structuring UI components
benefits greatly from the good bits of Object Oriented thinking and Om
leaves these be.

## Components

Om presents a model more or less conceptually similar to
React's, but we do not actually pass raw React props or states to
implementers of the life cycle protocols. Instead we pass immutable
values in both cases.

While it may not seem so at first, it's useful to preserve something
like React's component local state for two reasons. Often you do not
want to pollute the original data with transient application state
information. Editing a text field is a good example of this. The other
useful aspect of being able to set component local state is that it's
always guaranteed to be up-to-date. This is not true for application
state since Om renders on `requestAnimationFrame`, and application
state information is only guaranteed to be consistent during the
render phase. Thus event handlers must ask for an update-to-date
view of the application state.

## Application State

Outside the render phase you can use `om.core/read` to get a consistent
snapshot about a particular piece of data in the application
state. `om.core/transact!` is used to transition the application
state. The transition function should not rely on information not
obtained by `om.core/read`, `om.core/get-state!`, or
`om.core/transact!` itself.
