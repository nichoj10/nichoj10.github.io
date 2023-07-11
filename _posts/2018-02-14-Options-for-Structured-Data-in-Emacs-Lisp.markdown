---
title: Options for Structured Data in Emacs Lisp
layout: post
date: 2018-02-14T17:43:34Z
tags: [emacs, elisp]
uuid: 3837b5b2-0aba-3381-ff6f-9432f8ff03e9
---

So your Emacs package has grown beyond a dozen or so lines of code, and
the data it manages is now structured and heterogeneous. Informal plain
old lists, the bread and butter of any lisp, are not longer cutting it.
You really need to cleanly abstract this structure, both for your own
organizational sake any for anyone reading your code.

With informal lists as structures, you might regularly ask questions
like, "Was the 'name' slot stored in the third list element, or was
it the fourth element?" A plist or alist helps with this problem, but
those are better suited for informal, externally-supplied data, not
for internal structures with fixed slots. Occasionally someone
suggests using hash tables as structures, but Emacs Lisp's hash tables
are *much* too heavy for this. Hash tables are more appropriate when
keys themselves are data.

### Defining a data structure from scratch

Imagine a refrigerator package that manages a collection of food in a
refrigerator. A food item could be structured as a plain old list,
with slots at specific positions.

```cl
(defun fridge-item-create (name expiry weight)
  (list name expiry weight))
```

A function that computes the mean weight of a list of food items might
look like this:

```cl
(defun fridge-mean-weight (items)
  (if (null items)
      0.0
    (let ((sum 0.0)
          (count 0))
      (dolist (item items (/ sum count))
        (setf count (1+ count)
              sum (+ sum (nth 2 item)))))))
```

Note the use of `(nth 2 item)` at the end, used to get the item's
weight. That magic number 2 is easy to mess up. Even worse, if lots of
code accesses "weight" this way, then future extensions will be
inhibited. Defining some accessor functions solves this problem.

```cl
(defsubst fridge-item-name (item)
  (nth 0 item))

(defsubst fridge-item-expiry (item)
  (nth 1 item))

(defsubst fridge-item-weight (item)
  (nth 2 item))
```

The `defsubst` defines an inline function, so there's effectively no
additional run-time costs for these accessors compared to a bare
`nth`. Since these only cover *getting* slots, we should also define
some setters using the built-in gv (generalized variable) package.

```cl
(require 'gv)

(gv-define-setter fridge-item-name (value item)
  `(setf (nth 0 ,item) ,value))

(gv-define-setter fridge-item-expiry (value item)
  `(setf (nth 1 ,item) ,value))

(gv-define-setter fridge-item-weight (value item)
  `(setf (nth 2 ,item) ,value))
```

This makes each slot setf-able. Generalized variables are great for
simplifying APIs, since otherwise there would need to be an equal
number of setter functions (`fridge-item-set-name`, etc.). With
generalized variables, both are at the same entrypoint:

```cl
(setf (fridge-item-name item) "Eggs")
```

There are still two more significant improvements.

1. As far as Emacs Lisp is concerned, this isn't a real *type*. The
   type-ness of it is just a fiction created by the conventions of the
   package. It would be easy to make the mistake of passing an
   arbitrary list to these `fridge-item` functions, and the mistake
   wouldn't be caught so long as that list has at least three items.
   An common solution is to add a *type tag*: a symbol at the
   beginning of the structure that identifies it.

2. It's still a linked list, and `nth` has to walk the list (i.e.
   `O(n)`) to retrieve items. It would be much more efficient to use a
   vector, turning this into an efficient `O(1)` operation.

Addressing both of these at once:

```cl
(defun fridge-item-create (name expiry weight)
  (vector 'fridge-item name expiry weight))

(defsubst fridge-item-p (object)
  (and (vectorp object)
       (= (length object) 4)
       (eq 'fridge-item (aref object 0))))

(defsubst fridge-item-name (item)
  (unless (fridge-item-p item)
    (signal 'wrong-type-argument (list 'fridge-item item)))
  (aref item 1))

(defsubst fridge-item-name--set (item value)
  (unless (fridge-item-p item)
    (signal 'wrong-type-argument (list 'fridge-item item)))
  (setf (aref item 1) value))

(gv-define-setter fridge-item-name (value item)
  `(fridge-item-name--set ,item ,value))

;; And so on for expiry and weight...
```

As long as `fridge-mean-weight` uses the `fridge-item-weight`
accessor, it continues to work unmodified across all these changes.
But, *whew*, that's quite a lot of boilerplate to write and maintain
for each data structure in our package! Boilerplate code generation is
a perfect candidate for a macro definition. Luckily for us, Emacs
already defines a macro to generate all this code: `cl-defstruct`.

```cl
(require 'cl-lib)

(cl-defstruct fridge-item
  name expiry weight)
```

In Emacs 25 and earlier, this innocent looking definition expands into
essentially all the above code. The code it generates is expressed in
[the most optimal form][fast] for its version of Emacs, and it
exploits many of the available optimizations by using function
declarations such as `side-effect-free` and `error-free`. It's
configurable, too, allowing for the exclusion of a type tag (`:named`)
— discarding all the type checks — or using a list rather than a
vector as the underlying structure (`:type`). As a crude form of
structural inheritance, it even allows for directly embedding other
structures (`:include`).

#### Two pitfalls

There a couple pitfalls, though. First, for historical reasons, **the
macro will define two namespace-unfriendly functions: `make-NAME` and
`copy-NAME`**. I always override these, preferring the `-create`
convention for the constructor, and tossing the copier since it's
either useless or, worse, semantically wrong.

```cl
(cl-defstruct (fridge-item (:constructor fridge-item-create)
                           (:copier nil))
  name expiry weight)
```

If the constructor needs to be more sophisticated than just setting
slots, it's common to define a "private" constructor (double dash in
the name) and wrap it with a "public" constructor that has some
behavior.

```cl
(cl-defstruct (fridge-item (:constructor fridge-item--create)
                           (:copier nil))
  name expiry weight entry-time)

(cl-defun fridge-item-create (&rest args)
  (apply #'fridge-item--create :entry-time (float-time) args))
```

The other pitfall is related to printing. In Emacs 25 and earlier,
types defined by `cl-defstruct` are still only types by convention.
They're really just vectors as far as Emacs Lisp is concerned. One
benefit from this is that [printing and reading][read] these
structures is "free" because vectors are printable. It's trivial to
serialize `cl-defstruct` structures out to a file. This is [exactly
how the Elfeed database works][db].

The pitfall is that **once a structure has been serialized, there's no
more changing the `cl-defstruct` definition.** It's now a file format
definition, so the slots are locked in place. Forever.

Emacs 26 throws a wrench in all this, though it's worth it in the long
run. There's a new primitive type in Emacs 26 with its own reader
syntax: records. This is similar to hash tables [becoming first class
in the reader in Emacs 23.2][hash]. In Emacs 26, `cl-defstruct` uses
records instead of vectors.

```cl
;; Emacs 25:
(fridge-item-create :name "Eggs" :weight 11.1)
;; => [cl-struct-fridge-item "Eggs" nil 11.1]

;; Emacs 26:
(fridge-item-create :name "Eggs" :weight 11.1)
;; => #s(fridge-item "Eggs" nil 11.1)
```

So far slots are still accessed using `aref`, and all the type
checking still happens in Emacs Lisp. The only practical change is the
`record` function is used in place of the `vector` function when
allocating a structure. But it does pave the way for more interesting
things in the future.

The major short-term downside is that this breaks printed compatibility
across the Emacs 25/26 boundary. The `cl-old-struct-compat-mode`
function can be used for *some* degree of backwards, but not forwards,
compatibility. Emacs 26 can read and use some structures printed by
Emacs 25 and earlier, but the reverse will never be true. This issue
initially [tripped up Emacs' built-in packages][bug], and when Emacs 26
is released we'll see more of these issues arise in external packages.

### Dynamic dispatch

Prior to Emacs 25, the major built-in package for dynamic dispatch —
functions that specialize on the run-time type of their arguments — was
EIEIO, though it only supported single dispatch (specializing on a
single argument). EIEIO brought much of the Common Lisp Object System
(CLOS) to Emacs Lisp, including classes and methods.

Emacs 25 introduced a more sophisticated dynamic dispatch package
called cl-generic. It focuses only on dynamic dispatch and supports
multiple dispatch, completely replacing the dynamic dispatch portion
of EIEIO. Since `cl-defstruct` does inheritance and cl-generic does
dynamic dispatch, there's not really much left for EIEIO — besides bad
ideas like multiple inheritance and method combination.

Without either of these packages, the most direct way to build single
dispatch on top of `cl-defstruct` would be to [shove a function in one
of the slots][cobj]. Then the "method" is just a wrapper that call this
function.

```cl
;; Base "class"

(cl-defstruct greeter
  greeting)

(defun greet (thing)
  (funcall (greeter-greeting thing) thing))

;; Cow "class"

(cl-defstruct (cow (:include greeter)
                   (:constructor cow--create)))

(defun cow-create ()
  (cow--create :greeting (lambda (_) "Moo!")))

;; Bird "class"

(cl-defstruct (bird (:include greeter)
                    (:constructor bird--create)))

(defun bird-create ()
  (bird--create :greeting (lambda (_) "Chirp!")))

;; Usage:

(greet (cow-create))
;; => "Moo!"

(greet (bird-create))
;; => "Chirp!"
```

Since cl-generic is aware of the types created by `cl-defstruct`,
functions can specialize on them as if they were native types. It's a
lot simpler to let cl-generic do all the hard work. The people reading
your code will appreciate it, too:

```cl
(require 'cl-generic)

(cl-defgeneric greet (greeter))

(cl-defstruct cow)

(cl-defmethod greet ((_ cow))
  "Moo!")

(cl-defstruct bird)

(cl-defmethod greet ((_ bird))
  "Chirp!")

(greet (make-cow))
;; => "Moo!"

(greet (make-bird))
;; => "Chirp!"
```

The majority of the time a simple `cl-defstruct` will fulfill your
needs, keeping in mind the gotcha with the constructor and copier
names. Its use should feel almost as natural as defining functions.


[bug]: https://debbugs.gnu.org/cgi/bugreport.cgi?bug=27617
[cobj]: /blog/2014/10/21/
[db]: /blog/2013/09/09/
[fast]: /blog/2017/01/30/
[hash]: /blog/2010/06/07/
[read]: /blog/2013/12/30/
