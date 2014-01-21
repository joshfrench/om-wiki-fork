This tutorial is optimized for usage with
[Light Table](http://www.lighttable.com/).

Install [Leiningen](http://leiningen.org). Then at the command line
run the following where you like on the command line:

```
lein new mies-om om-tut
```

This will create a folder called `om-tut`. `cd` into it and run the
following command:

```
lein cljsbuild auto om-tut
```

This will start auto building so that recompiles will occur when you
save a file. The first build will take a few seconds. Once the build
has succeeded open `index.html` in your favorite browser (we recommend
Google Chrome as it has excellent support for source maps). You should
see an `h1` tag with the text content `Hello World!` in it.

Open `src/om_tut/core.cljs` in Light Table. Change `:text` value of
`app-state` to be something else other than `Hello World!`. Save the
file. Refresh your browser and you should see the new contents.

That's pretty boring isn't it? Let's do some live coding instead.

Type the key chord `Control-SPACE` to open up the command list. Start
typing `Add Connection`, press enter to select it. In the list of
options select **Browser (External)**. Copy and paste the script tag
into index.html before the `<div id="app"></div>`.

Open the JavaScript Console. You can do this via the Chrome menu
selection **View > Developer > JavaScript**. If everything went well
you should see `XHR finished loading ...` in the console. Refresh your
browser and arrange your windows so that you can see both the Chrome
window and your source code at the same time.

Now at the bottom of the `src/om_tut/core.cljs` source file write the
following:

```clj
(swap! app-state assoc :text "Do it live!")
```

Place your cursor at the end of the expression and type the key chord
`Command-ENTER` to evaluate it. Again it will take a second to make the
initial connection. After the connection is made and the application
is updated, edit the string again, re-evaluate and you will see that
updating the application on fly is pretty snappy.

Before proceeding remove the `swap!` expression

## Om basics

In Om the application state is held in an `atom`, the one reference
type built into ClojureScript. If you change the value of the atom via
`swap!` or `reset!` this will always trigger a re-render of any Om
roots (we'll explain this in a second). You can think of this atom as
the database of your client side application. Everything in the atom
should be an associative data structure - either a ClojureScript map
or indexed sequential data structure such as vector.

### om.core/root

`om.core/root` (which is aliased to `om/root` here), establishes a
Om rendering loop on a specific element in the DOM. The `om.root`
expression in the tutorial at this point looks like this:

```clj
(om/root
  app-state
  (fn [app owner]
    (dom/h1 nil (:text app)))
  (. js/document (getElementById "app")))
```

`om.core/root` is idempotent, that is, it's safe to evaluate it
multiple times. It takes up to four arguments, but we're only
interested in the three argument case. The first argument is the
application state atom. The second argument is a function that takes
the application state data and the backing React component, here
called `owner`. This function must return an Om component, a React
component, or some other value that React itself knows how to
render. The third argument is the target DOM node.

There can be multiple roots. Edit the `index.html`, replace `<div
id="app"></div>` with the following:

```html
<div id="app0"></div>
<div id="app1"></div>
```

And edit `src/om_tut/core.cljs` replacing the `om/root` expression
with the following:

```clj
(om/root
  app-state
  (fn [app owner]
    (dom/h1 nil (:text app)))
  (. js/document (getElementById "app0")))
```

Refresh your browser. You should see just one `h1` tag on the
page. Copy and paste the `om/root` expression and edit the second
one to look like the following:

```clj
(om/root
  app-state
  (fn [app owner]
    (dom/h1 nil (:text app)))
  (. js/document (getElementById "app1"))) ;; <-- "app0" to "app1"
```

Place your cursor at the end of this expression and evaluate it. You
should see the second `h1` tag magically appear.

At the end of the file type the following and evaluate it.

```clj
(swap! app-state assoc :text "Multiple roots!")
```

You should see both `h1` tags update on the fly. Multiple roots are
fully supported and synchronized to render on the same
`requestAnimationFrame`.

Before proceeding remove the `<div id="app1"></div>` from
`index.html` and remove the second `om/root` expression and the
`swap!` expression. Save and refresh the browser.

## Rendering a list of things

Change the `app-state` expression to the following and evaluate
it. Don't bother refreshing,
[John McCarthy](http://library.stanford.edu/collections/john-mccarthy-papers-0)
would be pissed!

```clj
(def app-state (atom {:list ["Lion" "Zebra" "Buffalo" "Antelope"]}))
```

Change the `om/root` expression to the following and evaluate it, you
should see a list of animals now.

```clj
(om/root
  app-state
  (fn [app owner]
    (apply dom/ul nil
      (map #(dom/li nil %) (:list app))))
  (. js/document (getElementById "app0")))
```

You might have noticed that the first argument to `dom/ul` and
`dom/li` is `nil`. This argument is how you set DOM attributes. Change
the `om/root` expression to the following and evaluate it:

```clj
(om/root
  app-state
  (fn [app owner]
    (apply dom/ul #js {:className "animals"}
      (map #(dom/li nil %) (:list app))))
  (. js/document (getElementById "app0")))
```

If you right click on the list in Google Chrome and select **Inspect
Element** you should see that the `ul` tag in the DOM does indeed have
its CSS class attribute set to "animals".

`#js {...}` and `#js [...]` is what is referred to as a reader
literal. ClojureScript support data literals for JavaScript via
`#js`. `#js {...}` is for JavaScript objects:

```clj
#js {:foo "bar"}  ;; is equivalent to
#js {"foo" "bar"}
```

`#js [...]` is for JavaScript arrays:

```clj
#js [1 2 3]
```

The `#js` reader literal support is shallow, take note of the
following:

```clj
#js {:foo [1 2 3]} ;; a JS object with a persistent vector in it
```

## Life without a templating language

In Om you have the full power of the ClojureScript language when building
your user interface. At the same time, Om leaves the door open for
alternate syntaxes for describing the DOM if that's your cup of tea.

Lets edit our code so we get zebra striping on the list. Lets add a
helper function `stripe` before the `om/root` expression:

```clj
(defn stripe [text bgc]
  (let [st #js {:backgroundColor bgc}]
    (dom/li #js {:style st} text)))
```

Don't forget to evaluate it!

Then change the `om/root` expression to the following and evaluate it:

```clj
(om/root
  app-state
  (fn [app owner]
    (apply dom/ul #js {:className "animals"}
      (map stripe (:list app) (cycle ["#ff0" "#fff"]))))
  (. js/document (getElementById "app0")))
```

As we can see ClojureScript offers powerful functional tools that put
most templating languages completely to shame.
