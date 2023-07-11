---
title: RSA Signatures in Emacs Lisp
layout: post
date: 2015-10-30T22:35:13Z
tags: [emacs, elisp, lisp]
uuid: 9d9ef14d-d513-3cad-b053-fb016f3c3bf0
---

Emacs comes with a wonderful arbitrary-precision computer algebra
system called [calc][calc]. I've [discussed it previously][prev] and
continue to use it on a daily basis. That's right, people, *Emacs can
do calculus*. Like everything Emacs, it's programmable and extensible
from Emacs Lisp. In this article, I'm going to implement the [RSA
public-key cryptosystem][rsa] in Emacs Lisp using calc.

If you want to dive right in first, here's the repository:

* <https://github.com/skeeto/emacs-rsa>

This is only a toy implementation and not really intended for serious
cryptographic work. It's also far too slow when using keys of
reasonable length.

### Evaluation with calc

The calc package is particularly useful when considering Emacs'
limited integer type. Emacs uses a tagged integer scheme where
integers are embedded within pointers. It's a lot faster than the
alternative (individually-allocated integer objects), but it means
they're always a few bits short of the platform's native integer type.

calc has a large API, but the user-friendly porcelain for it is the
under-documented `calc-eval` function. It evaluates an expression
string with format-like argument substitutions (`$n`).

~~~cl
(calc-eval "2^16 - 1")
;; => "65535"

(calc-eval "2^$1 - 1" nil 128)
;; => "340282366920938463463374607431768211455"
~~~

Notice it returns strings, which is one of the ways calc represents
arbitrary precision numbers. For arguments, it accepts regular Elisp
numbers and strings just like this function returns. The implicit
radix is 10. To explicitly set the radix, prefix the number with the
radix and `#`. This is the same as in the user interface of calc. For
example:

~~~cl
(calc-eval "16#deadbeef")
;; => "3735928559"
~~~

The second argument (optional) to `calc-eval` adjusts its behavior.
Given `nil`, it simply evaluates the string and returns the result.
The manual documents the different options, but the only other
relevant option for RSA is the symbol `pred`, which asks it to return
a boolean "predicate" result.

~~~cl
(calc-eval "$1 < $2" 'pred "4000" "5000")
;; => t
~~~

### Generating primes

RSA is founded on the difficulty of factoring large composites with
large factors. Generating an RSA keypair starts with generating two
prime numbers, `p` and `q`, and using these primes to compute two
mathematically related composite numbers.

calc has a function `calc-next-prime` for finding the next prime
number following any arbitrary number. It uses a probabilistic
primarily test — the <del>Fermat</del> Miller-Rabin primality test
— to efficiently test large integers. It increments the input until
it finds a result that passes enough iterations of the primality test.

~~~cl
(calc-eval "nextprime($1)" nil "100000000000000000")
;; => "100000000000000003"
~~~

So to generate a random n-bit prime, first generate a random n-bit
number and then increment it until a prime number is found.

~~~cl
;; Generate a 128-bit prime, 10 iterations (0.000084% error rate)
(calc-eval "nextprime(random(2^$1), 10)" nil 128)
"111618319598394878409654851283959105123"
~~~

Unfortunately calc's `random` function is based on Emacs' `random`
function, which is entirely unsuitable for cryptography. In the real
implementation I read n bits from `/dev/urandom` to generate an n-bit
number.

~~~cl
(with-temp-buffer
  (set-buffer-multibyte nil)
  (call-process "head" "/dev/urandom" t nil "-c" (format "%d" (/ bits 8)))
  (let ((f (apply-partially #'format "%02x")))
    (concat "16#" (mapconcat f (buffer-string) ""))))
~~~

(Note: `/dev/urandom` *is* the right choice. There's [no reason to use
`/dev/random` for generating keys][myth].)

### Computing e and d

From here the code just follows along from the Wikipedia article.
After generating the primes `p` and `q`, two composites are computed,
`n = p * q` and `i = (p - 1) * (q - 1)`. Lacking any reason to do
otherwise, I chose 65,537 for the public exponent `e`.

The function `rsa--inverse` is just a straight Emacs Lisp + calc
implementation of the extended Euclidean algorithm from [the Wikipedia
article pseudocode][euc], computing `d ≡ e^-1 (mod i)`. It's not much
use sharing it here, so take a look at the repository if you're
curious.

~~~cl
(defun rsa-generate-keypair (bits)
  "Generate a fresh RSA keypair plist of BITS length."
  (let* ((p (rsa-generate-prime (+ 1 (/ bits 2))))
         (q (rsa-generate-prime (+ 1 (/ bits 2))))
         (n (calc-eval "$1 * $2" nil p q))
         (i (calc-eval "($1 - 1) * ($2 - 1)" nil p q))
         (e (calc-eval "2^16+1"))
         (d (rsa--inverse e i)))
    `(:public  (:n ,n :e ,e) :private (:n ,n :d ,d))))
~~~

The public key is `n` and `e` and the private key is `n` and `d`. From
here we can compute and verify cryptographic signatures.

### Signatures

To compute signature `s` of an integer `m` (where `m < n`), compute
`s ≡ m^d (mod n)`. I chose the right-to-left binary method, again
straight from [the Wikipedia pseudocode][pow] (lazy!). I'll share this
one since it's short. The backslash denotes integer division.

~~~cl
(defun rsa--mod-pow (base exponent modulus)
  (let ((result 1))
    (setf base (calc-eval "$1 % $2" nil base modulus))
    (while (calc-eval "$1 > 0" 'pred exponent)
      (when (calc-eval "$1 % 2 == 1" 'pred exponent)
        (setf result (calc-eval "($1 * $2) % $3" nil result base modulus)))
      (setf exponent (calc-eval "$1 \\ 2" nil exponent)
            base (calc-eval "($1 * $1) % $2" nil base modulus)))
    result))
~~~

Verifying the signature is the same process, but with the public key's
`e`: `m ≡ s^e (mod n)`. If the signature is valid, `m` will be
recovered. In theory, only someone who knows `d` can feasibly compute
`s` from `m`. If `n` is [small enough to factor][factor], revealing
`p` and `q`, then `d` can be feasibly recomputed from the public key.
So mind your Ps and Qs.

So that leaves one problem: generally users want to sign strings and
files and such, not integers. A hash function is used to reduce an
arbitrary quantity of data into an integer suitable for signing. Emacs
comes with a bunch of them, accessible through `secure-hash`. It
hashes strings and buffers.

~~~cl
(secure-hash 'sha224 "Hello, world!")
;; => "8552d8b7a7dc5476cb9e25dee69a8091290764b7f2a64fe6e78e9568"
~~~

Since the result is hexadecimal, just prefix `16#` to turn it into a
calc integer.

Here's the signature and verification functions. Any string or buffer
can be signed.

~~~cl
(defun rsa-sign (private-key object)
  (let ((n (plist-get private-key :n))
        (d (plist-get private-key :d))
        (hash (concat "16#" (secure-hash 'sha384 object))))
    ;; truncate hash such that hash < n
    (while (calc-eval "$1 > $2" 'pred hash n)
      (setf hash (calc-eval "$1 \\ 2" nil hash)))
    (rsa--mod-pow hash d n)))

(defun rsa-verify (public-key object sig)
  (let ((n (plist-get public-key :n))
        (e (plist-get public-key :e))
        (hash (concat "16#" (secure-hash 'sha384 object))))
    ;; truncate hash such that hash < n
    (while (calc-eval "$1 > $2" 'pred hash n)
      (setf hash (calc-eval "$1 \\ 2" nil hash)))
    (let* ((result (rsa--mod-pow sig e n)))
      (calc-eval "$1 == $2" 'pred result hash))))
~~~

Note the hash truncation step. If this is actually necessary, then
your `n` is *very* easy to factor! It's in there since this is just a
toy and I want it to work with small keys.

### Putting it all together

Here's the whole thing in action with an extremely small, 128-bit key.

~~~cl
(setf message "hello, world!")

(setf keypair (rsa-generate-keypair 128))
;; => (:public  (:n "74924929503799951536367992905751084593"
;;               :e "65537")
;;     :private (:n "74924929503799951536367992905751084593"
;;               :d "36491277062297490768595348639394259869"))

(setf sig (rsa-sign (plist-get keypair :private) message))
;; => "31982247477262471348259501761458827454"

(rsa-verify (plist-get keypair :public) message sig)
;; => t

(rsa-verify (plist-get keypair :public) (capitalize message) sig)
;; => nil
~~~

Each of these operations took less than a second. For larger,
secure-length keys, this implementation is painfully slow. For
example, generating a 2048-bit key takes my laptop about half an hour,
and computing a signature with that key (any size message) takes about
a minute. That's probably a little too slow for, say, signing ELPA
packages.


[calc]: http://www.gnu.org/software/emacs/manual/html_mono/calc.html
[prev]: /blog/2009/06/23/
[rsa]: https://en.wikipedia.org/wiki/RSA_(cryptosystem)
[euc]: https://en.wikipedia.org/wiki/Extended_Euclidean_algorithm
[pow]: https://en.wikipedia.org/wiki/Modular_exponentiation#Right-to-left_binary_method
[factor]: http://crypto.stackexchange.com/a/5942
[myth]: http://www.2uo.de/myths-about-urandom/
