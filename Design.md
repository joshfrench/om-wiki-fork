### Warts

#### Preventing Lock-in

Currently when building a generic component you are locked into Om's
API. We could replace the explicit Om dependency by instead requiring
only a higher level shared dependency like `core.async`. Setting app
state or local component state can then be designed such that other
rendering systems may be used. The only remaining assumption is
that the component function take two parameters - one is the data
representing the component, the other is an optional map containing
whatever side information is needed.

I think by pursuing how a component can work outside of the Om model
can guide us to better solutions regarding both application state and
component state.

#### Sketch

The following is more or less what Om exposes. This seems like it
would be enough to take an Om component and make it work
differently. Would need to interpret React life-cycle protocols.

```clj
(defprotocol IBuildComponent
  (build [c f] [c f opts]))

(defprotocol IGlobalState
  (update! [c f] [c ks f]))

(defprotocol IComponentState
  (set-state! [c v] [c ks v])
  (get-state [c] [c ks]))

(defprotocol IRootNode
  (to-cursor [x]))

(defprotocol INodeRef
  (get-node [c id]))
```
