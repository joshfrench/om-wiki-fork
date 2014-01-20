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

That's pretty boring isn't it. Let's do some live coding instead.

Type the key chord `Control-SPACE` to open up the command list. Start
typing `Add Connection`, press enter to select it. In the list of
options select *Browser (External)*. Copy and paste the script tag
before the `<div id="app"></div>`.

Open the JavaScript Console. You can do this via the Chrome menu
selection *View > Developer > JavaScript*. If everything went well you
should see `XHR finished loading ...` in the console.

Arrange your windows so that you can see both the Chrome window and
your source code at the same time.

Now at the bottom of the `src/om_tut/core.cljs` source file write the
following:

```clj
(swap! app-state assoc :text "Do it live!")
```

Place your cursor at the end of the expression and type the key chord
`Command-ENTER` to evalute it. Again it will take a second to make the
initial connection. After the connection is made and the application
is updated, edit the string again, re-evaluate and you will see that
updating the application on fly is pretty snappy.

Before processing remove the `swap!` expression

## Om basics

In Om the application state is held in an `atom`, the one reference
type build into ClojureScript. If you change the value of the atom via
`swap!` or `reset!` this will always trigger a re-render of the
application. Everything in the atom should be an associative data
structure - either a ClojureScript map or indexed sequential data
structure such as vector.

### om.core/root

`om.core/root` (which is aliased to `om/root` here), establishes a
Om rendering loop on a specific element in the dom. `om.core/root` is
idempotent, that is, it's safe to evaluate it multiple times. It takes
up four argument, but we're only interested in the three argument
case. The first argument is application state atom. The second
argument is a function that takes the application state data and the
backing React component, here called `owner`. This function must
return an Om component, a React component, or some other value that
React itself knows how to render. The third argument is the target DOM
node.

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
page. Copy and past the `om/root` expression and edit the second
to look like the following:

```clj
(om/root
  app-state
  (fn [app owner]
    (dom/h1 nil (:text app)))
  (. js/document (getElementById "app1"))) ;; <-- "app0" to "app1"
```

Place your cursor at the end of this expression and evaluate it. You
should see the second `h1` tag magically appear.

At the end of the file type the following and evalute it.

```clj
(swap! app-state assoc :text "Multiple roots!")
```

You should see both `h1` tags update on the fly. Multiple roots are
fully supported and synchronized to render on the same
`requestAnimationFrame`.