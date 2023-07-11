---
title: The Limits of Emacs Advice
layout: post
tags: [emacs, elisp]
uuid: 1cfcdeee-19e5-33b8-1344-8fef68333e41
---

Today at work I was using [impatient-mode][imp] to share some code
with [Brian][brian]. It makes for a really handy live pastebin. To
limit the buffer to the relevant code, I narrowed it down with
`narrow-to-region`. However, the browser wouldn't update to show only
the narrowed region until I made an edit. This makes sense because
impatient-mode hooks `after-change-functions`.  Narrowing the buffer
doesn't *change* anything in the buffer, so, as expected, this hook is
not called.

The solution would be to also join whatever hook is called when the
buffer restriction changes. Unfortunately,
[no such hook exists][hooks]. I thought I could create this hook with
some [advice][advice], but this turns out to be currently impossible.

### Emacs Advice

What's advice? It's a handy feature of Emacs lisp that allows users to
modify the behavior of almost any function without having to redefine
it. It works a little bit like methods in the Common Lisp Object
System (CLOS): advice is code than can be evaluated before, after, or
around a function.

Advice is defined with `defadvice`. Duh. For example, say we wanted to
be silly and have Emacs say "Ouch!" when a line is killed with
`kill-line`. We can advise this function to display a message.

~~~cl
(defadvice kill-line (after say-ouch activate)
  (message "Ouch!"))
~~~

This says we want to advise the function `kill-line`, we want this
advise to execute *after* `kill-line` has run, our advice is named
"`say-ouch`", and we want to immediately activate this advice so it
gets used right away. The rest is the body of the advice, like the
body of a function. After evaluating this `defadvice`, every time I
hit `C-k` Emacs says "Ouch!" in the minibuffer. Cool!

### narrow-to-region and widen

A hook is a variable that holds a list of functions. (Or maybe hooks
are the functions in this list? Emacs' documentation calls both of
these things hooks.) These functions are called, usually without
arguments, when some specific event occurs. For example, every mode
has its own mode hook which is called when the mode is activated in a
buffer. This allows users to extend or modify the mode — like by
enabling additional minor modes — without editing the mode's source
code directly.

To make our hook work we need to advise `narrow-to-region` and `widen`
to run the hook after they've done their work. These are the primitive
narrowing functions which all the other narrowing functions eventually
call, like `narrow-to-defun`, `narrow-to-page`, and any other
mode-specific narrowing. **Advising these two functions will cover all
buffer narrowing.** It *should* be this simple.

~~~cl
(defvar change-restriction-hook ())

(defadvice narrow-to-region (after hook activate)
  (run-hooks 'change-restriction-hook))

(defadvice widen (after hook activate)
  (run-hooks 'change-restriction-hook))
~~~

At first this seems to work. I can add a test hook see them activate
when I use `M-x narrow-to-region` and `M-x widen`. However, when I use
other narrowing functions, like `narrow-to-defun`, my hook functions
aren't called.

Is there a narrowing primitive I missed? I check the source
code. Nope, these are lisp functions which ultimately call
`narrow-to-region`. Is the advice not getting used when called
indirectly? I test that out.

~~~cl
(defun foo ()
  (interactive)
  (narrow-to-region 1 2))
~~~

This works fine. Hmmm, these other functions are byte-compiled, maybe
that's the problem.

~~~cl
(byte-compile 'foo)
~~~

Bingo. The advice has stopped working. It has something to do with
byte-compilation.

### Bytecode

Let's take a look at the bytecode for `foo`.

~~~cl
(symbol-function 'foo)
;; => #[nil "\300\301}\207" [1 2] 2 nil nil]
~~~

I don't know too much about Emacs' byte code, but here's the gist of
it. A compiled function is a special type of vector (hence the `#[]`
form). This is a legal s-expression which you can use directly in
regular Elisp code just like it was a function. The only reason you'd
do so is for obfuscation, so it would look very suspicious.

The first element of this function vector is the parameter list —
empty in this case. The second is a string containing the actual
bytecodes. The rest holds the various constants from the function
body. This includes the symbols of other functions called by this
function. It's important to note that **`narrow-to-region` does not
appear in this list**!

Curious. Let's take a closer look at the bytecode.

~~~cl
(coerce (aref (symbol-function 'foo2) 1) 'list)
;; => (192 193 125 135)
~~~

Looking at `bytecomp.el` from the Emacs distribution I can see that
codes 192 and 193 are used for accessing constants. This pushes my
constants 1 and 2 onto a stack for use as function arguments. Next up
is 125, which corresponds to `byte-narrow-to-region`. Gotcha!

It turns out `narrow-to-region` is so special — probably because it's
used very frequently — that it gets its own bytecode. The **primitive
function call is being compiled away into a single instruction**. This
means my advice will not be considered in byte-compiled code. Darnit.
The same is true for `widen` (code 126).

### Where to go now?

Since it's not possible to hook or advise the buffer-narrowing
primitives, impatient-mode would need to hook some other event that
tends to happen at the same time. Perhaps any time a command is
executed in the current buffer it could check for changes to the
buffer restriction and, if so, update any attached web clients. I'll
figure something out.


[imp]: http://www.50ply.com/blog/2012/08/13/introducing-impatient-mode/
[brian]: http://www.50ply.com/
[hooks]: http://www.gnu.org/software/emacs/manual/html_node/elisp/Standard-Hooks.html
[advice]: http://www.gnu.org/software/emacs/manual/html_node/elisp/Advising-Functions.html
