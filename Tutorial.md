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

Open `src/om_tut.cljs` in Light Table. Change `:text` value of
`app-state` to be something else other than `Hello World!`. Save the
file. Refresh your browser and you should see the new contents.

That's pretty boring isn't it. Let's do some live coding instead.

Type the key chord `Control-SPACE` to open up the command list. Start
typing `Add Connection`, press enter to select it. In the list of
options select *Browser (External)*. Copy and paste the script tag
before the `<div id="app">` tag.

Open the JavaScript Console. You can do this via the Chrome menu
selection *View > Developer > JavaScript*. If everything went well you
should see `XHR finished loading ...` in the console.

Arrange your windows so that you can see both the Chrome window and
your source code at the same time.

Now at the bottom of the `om_tut.cljs` source file write the following:

```clj
(swap! app-state assoc :text "Do it live!")
```

Place your cursor at the end of the expression and type the key chord
`Command-ENTER` to evalute it. Again it will take a second to make the
initial connection. After the connection is made and the application
is updated, edit the string again, re-evaluate and you will see that
updating the application on fly is pretty snappy.
