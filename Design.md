### Warts

I'm still not completely happy with how state is handled. In contrast
dealing with paths has become considerably cleaner with the inclusion
of the `ICursor` types - users can simply access their application
data using the ClojureScript standard library and we can track the
path under the hood. This important given the last section on this
page - users can build components and be completely oblivious to the
path abstraction.

The state tension lies in the fact that we want components to be
shareable and in order to be shareable they need to be oblivious as to
where in the UI state their actual information presides. At the same
time we think there's promise in the Om model in that UIs are simply
an alternate interface to data - one that broadens the scope beyond
Clojure(Script) programmers. *Tangible Data* is a worthy goal.

#### Go loops

Go loops get installed when a React/Om component mounts. Any cursors
closed over this point are bound to be stale. One solution is to
provide something like `cursor-chan` from which an up-to-date cursor
may always be read. However this may introduce more non-determinism
then we would like, and there are other uses for a synchronous version
of such a feature.

#### Getting data from root

For joining together components being able to access data from the
root of the application state is useful. Now that paths are implicit
we've dropped `:abs-path` option to `om.core/build`. This functionality
should be restored.

Perhaps we should allows users to create and name an application
state? Then users could access any cursor at any time.

#### Preventing Lock-in

Currently when building a generic component you are locked into Om's
API. We could replace the explicit Om dependency by instead requiring
only a higher level shared dependency like `core.async`. Setting app
state or local component state then can be designed such other
rendering systems may be used. Then the only remaining assumption is
that the component function takes two parameters - one is the data
representing the component, the other is an optional map containing
whatever side information is needed.

I think by pursuing how a component can work outside of the Om model
can guide us to better solutions regarding both application state and
component state.
