---
title: Emacs Lisp Readable Closures
layout: post
date: 2013-12-30T23:52:38Z
tags: [emacs, elisp, lisp, javascript]
uuid: 84f86fb6-e029-3a57-1450-1d25be3fdee0
---

I've stated before that one of the unique features of Emacs Lisp is
that its closures are *readable*. Closures can be serialized by the
printer and read back in with the reader. I am unaware of any other
programming language that has this feature. In fact it's essential for
Elisp byte-code compilation because byte-compiled Elisp files are
merely s-expressions of byte-code dumped out as source.

### Lisp Printing

The Lisp family of languages are *homoiconic*. Lisp source code is
written in the syntax of its own data structures, s-expressions. Since
a compiler/interpreter is usually provided at run-time, a consequence
of this is that reading and printing are a fundamental feature of
Lisps. A value can be handed to the printer, which will serialize the
value into an s-expression as a sequence of characters. Later on the
reader can parse the s-expression back into an `equal` value.

To compare, JavaScript originally had half of this in place.
JavaScript has convenient object syntax for defining an associative
array, known today as JSON. The `eval` function could (dangerously) be
used as a reader for parsing a string containing JSON-encoded data
into a value. But until `JSON.stringify()` became standard, developers
had to write their own printer. Lisp s-expression syntax is much more
powerful (and complicated) than JSON, maintaining
[both identity and cycles][print] (e.g. `*print-circle*`).

Not all values can be read. They'll still print (when `*print-readably*`
is nil) but will do so using special syntax that will signal an error
in the reader: `#<`. For example, in Emacs Lisp buffers cannot be
serialized so they print using this syntax.

~~~cl
(prin1-to-string (current-buffer))
;; => "#<buffer *scratch*>"
~~~

It doesn't matter what's between the angle brackets, or even that
there's a closing angle bracket. The reader will signal an error as
soon as it hits a `#<`.

#### Almost Everything Prints Readably

Elisp has a small set of primitive data types. All of these primitive
types print readably:

 * integer (`1024`, `?a`)
 * float (`1.7`)
 * cons/list (`(...)`)
 * vector (one-dimensional, `[...]`)
 * bool-vector (`#&n"..."`)
 * string (`"..."`)
 * char-table (`#^[...]`)
 * hash-table (readable as of Emacs 23.3, `#s(hash-table ...)`)
 * byte-code function object (`#[...]`)
 * symbol

Here are all the non-readable types. Each one has a good reason for
not being serializable.

 * buffer
 * process (external state)
 * frame (user interface element)
 * marker (live, automatically updates)
 * overlay (belongs to a buffer)
 * built-in functions (native code)
 * user-ptr (opaque pointers from Emacs 25 dynamic modules)

And that's it. Every other value in Elisp is constructed from one or
more of these primitives, including keymaps, functions, macros, syntax
tables, `defstruct` structs, and EIEIO objects. This means that as
long as these values don't refer to an unreadable value, they
themselves can be printed.

An interesting note here is that, unlike the Common Lisp Object System
(CLOS), EIEIO objects are readable by default. To Elisp they're just
vectors, so of course they print. CLOS objects are unreadable without
manually defining a print method per class.

### Elisp Closures

Elisp got lexical scoping in Emacs 24, released in June 2012. It's now
one of the relatively few languages to have both dynamic and lexical
scope. Like Common Lisp, variables declared with `defvar` (and family)
continue to have dynamic scope. For backwards compatibility with old
Lisp code, lexical scope is disabled by default. It's enabled for a
specific file or buffer by setting `lexical-binding` to non-nil.

With lexical scope, anonymous functions become closures, a powerful
functional programming primitive: a function plus a captured lexical
environment. It also provides some performance benefits. In my own
tests, compiled Elisp with lexical scope enabled is about 10% to 15%
faster than with the default dynamic scope.

What do closures look like in Emacs Lisp? It takes on two forms
depending on whether the closure is compiled or not. For example,
consider this function, `foo`, that takes two arguments and returns a
closure that returns the first argument.

~~~cl
;; -*- lexical-binding: t; -*-
(defun foo (x y)
  (lambda () x))

(foo :bar :ignored)
;; => (closure ((y . :ignored) (x . :bar) t) () x)
~~~

An uncompiled closure is a list beginning with the symbol `closure`.
The second element is the lexical environment, the third is the
argument list (lambda list), and the rest is the body of the function.
Here we can see that both `x` and `y` have been "closed over." This is
a little bit sloppy because the function never makes use of `y`.
Capturing it has a few problems.

 * The closure has a larger footprint than necessary.
 * Values are held longer than necessary, delaying collection.
 * It affects the readability of the closure, which I'll get to later.

Fortunately the compiler is smart enough to see this and will avoid
capturing unused variables. To prove this, I've now compiled `foo` so
that it returns a compiled closure.

~~~cl
(foo :bar :ignored)
;; => #[0 "\300\207" [:bar] 1]
~~~

What's returned here is a byte-code function object, with the `#[...]`
syntax. It has these elements:

 1. The function's lambda list (zero arguments)
 2. Byte-codes stored in a unibyte string
 3. Constants vector
 4. Maximum stack space needed by this function

Notice that the lexical environment has been captured in the constants
vector, specifically noting the lack of `:ignored` in this vector. The
compiler didn't capture it.

For those curious about the byte-code here's an explanation. The
string syntax shown is in octal, representing a string containing two
bytes: 192 and 135. The
[Elisp byte-code interpreter is stack-based][internals]. The 192
(`constant 0`) says to push the first constant onto the stack. The 135
(`return`) says to pop the top element from the stack and return it.

~~~cl
(coerce "\300\207" 'list)
;; => (192 135)
~~~

### The Readable Closures Catch

Since closures are byte-code function objects, they print readably.
You can capture an environment in a closure, serialize it, read it
back in, and evaluate it. That's pretty cool! This means closures can
be transmitted to other Emacs instances in a multi-processing setup
(i.e. [Elnode][elnode], [Async][async])

The catch is that it's easy to accidentally capture an unreadable
value, especially buffers. Consider this function `bar` which uses a
temporary buffer as an efficient string builder. It returns a closure
that returns the result. (Weird, but stick with me here!)

~~~cl
(defun bar (n)
  (with-temp-buffer
    (let ((standard-output (current-buffer)))
      (loop for i from 0 to n do (princ i))
      (let ((string (buffer-string)))
        (lambda () string)))))
~~~

The compiled form looks fine,

~~~cl
(bar 3)
;; => #[0 "\300\207" ["0123"] 1]
~~~

But the interpreted form of the closure has a problem. The
`with-temp-buffer` macro silently introduced a new binding — an
abstraction leak.

~~~cl
(bar 3)
;; => (closure ((string . "0123")
;;              (temp-buffer . #<killed buffer>)
;;              (n . 3) t)
;;      () string)
~~~

The temporary buffer is mistakenly captured in the closure making it
unreadable, but *only* in its uncompiled form. This creates the
awkward situation where compiled and uncompiled code has [different
behavior][bc].


[print]: /blog/2013/03/28/
[async]: https://github.com/jwiegley/emacs-async
[elnode]: https://github.com/nicferrier/elnode
[internals]: /blog/2014/01/04/
[bc]: /blog/2016/12/22/#accidental-closures
