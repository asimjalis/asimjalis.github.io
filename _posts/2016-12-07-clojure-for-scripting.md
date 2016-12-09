---
layout: post
title:  "How To Use Clojure For Scripting"
date:   2016-12-07
---

## Getting to REPL

Suppose you want to try out Clojure, but don't want to spent a lot of
time setting up a project. 

You just want to fire up the REPL and start playing with code.

Also you want to include other Clojure libraries dynamically on a
whim. You don't want to declare all of them first, or restart your
REPL every time you think of a new library to pull in. 

These steps will show you how to do this.

## Boot-clj

First we are going to install `boot-clj`. Boot is similar to Lein with
the difference that it lets you define your builds in Clojure
dynamically.

Install `boot-clj`.

    brew install boot-clj

Start boot repl.

    boot repl

Test the REPL.

```clojure
(println "Hello, world!!")
```

At this point the REPL should work. You can play with all the
functions that ship with Clojure.

## Adding Dependencies

For most interesting applications you want to pull in dependencies
from mvnrepository or from clojars. Clojure core just isn't enough.

Here is how to pull in 3rd party dependencies.

We are going to define a function called `deps` that will give us the
ability to dynamically pull in dependencies from mvnrepository or
clojars.

Paste this into your REPL.

```clojure
; Define deps to pull in dependencies dynamically
(defn deps [new-deps]
  (merge-env! :dependencies new-deps))
```

## Testing With CPrint

Lets test this using cprint, which is a color pretty printing
function. 

To use `cprint` we are going to pull in `lein-cprint` version `1.2.0`.
You can get the dependency name and version from mvnrepository or
clojars. 

```clojure
; Here is how to pull in the dependency 
(deps '[[lein-cprint "1.2.0"]])
```

To use a function in the dependency we can either use `use` or
`requires`. This will pull it into our current namespace.

```clojure
; `use` imports cprint directly into our namespace
(use 'leiningen.cprint)
(cprint (range 10))

; `require` imports cprint as cp/cprint in our namespace
(require '[leiningen.cprint :as cp])
(cp/cprint (range 10))
```

## REPL History

Now after hacking for a bit you will want to page through your REPL
history.

To view the REPL history look at `.nrepl-history` in the directory
where you started `boot repl`.

    cat .nrepl-history

## Loading Scripts

You can copy the history into a file, clean it up, and then refactor
it into an elegant script defining functions.

Next time you start a session you want to load the script file you
created. Here is how to load a script file into the REPL. 

```clojure
(load-file "myscript.clj")
```

This should give you a good starting point in how to use the REPL to
interactively develop Clojure programs without getting bogged down in
setting up a project.

## REPL in Production

Now at this point you might say, "This is a great way to goof around
in Clojure, but to put a system into production we have to put on a
serious face and give up the joys of the REPL."

Not necessarily. I run production servers with a Boot REPL. You can
too. The REPL is a very powerful shell that lets you call and change
parts of your Clojure program on the fly. It's like doing engine
repairs on a plane in flight. 

Using the REPL you inspect and debug errors as they happen. You can
dynamically redefine functions. And most importantly you can retain
the playful REPL mindset in production.

## Clojure Script Files

Can I run my Clojure scripts like I do with my Python, Ruby, Perl, and
Bash scripts?

Yes you can. You can run your scripts directly using this shebang line 
`#!/usr/bin/env boot` on top of your script file.

Save this into `hello.clj` and try it out.

```clojure
#!/usr/bin/env boot
(println "Hello world!")
```

This turns Clojure into a scripting language much like Python, Ruby,
Perl, or Bash.  

## Comments

Share your thoughts on this post
[here](https://news.ycombinator.com/item?id=13134104).



