# cljs-load-order-repro

Minimal reproduction of a possible issue with the ClojureScript compiler.

# The issue

The order that namespaces are loaded at runtime seems to be independent of the 
order that the namespaces are `require`d in user code.

It looks like some work to ensure the ordering at compile time was done as 
part of [CLJS-1453](https://dev.clojure.org/jira/browse/CLJS-1453), but it
seems that things can get out of order by the time `cljs_deps.js` is emitted.

This seems to affect all optimization levels.

# Potential cause

It seems that the output in `cljs_deps.js` for a given ClojureScript file is 
ultimately determined by the output of `cljs.compiler/emit-source`. 

The code there removes duplicate deps in the same way that the `'ns` parse
method in `cljs.analyzer` did prior to CLJS-1453.

The attached patch resolves the issue for my narrow use case, but I believe the
underlying functions are passing the `:uses` and `:requires` around as maps, so
more work may be needed for a full solution.


# How to reproduce

Running this command:

```
clj --main cljs.main --compile-opts '{:target :nodejs :main a}' --compile a
```

will compile the `a` namespace defined in `a.cljs`. That namespace requires—via
the `:require` form of the `ns` macro— namespaces `b` through `g` in
alphabetical order.

After compiling, observe the following:

* The `goog.addDependency` call for `a.js` in the output `out/cljs_deps.js` 
file includes its dependencies in an arbitrary order. For example, one run on
my machine resulted in:

```
goog.addDependency("../a.js", ['a'], ['cljs.core', 'e', 'c', 'g', 'b', 'd', 'f']);
```

* The console output given by running the compiled code with `node out/main.js` 
indicates that the runtime load order is identical to the ordering reflected
in `out/cljs_deps.js`

```
body of e

body of c

body of g

body of b 

body of d

body of f
```
