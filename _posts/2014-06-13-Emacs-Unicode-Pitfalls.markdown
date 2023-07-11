---
title: Emacs Unicode Pitfalls
layout: post
date: 2014-06-13T05:58:34Z
tags: [emacs, elisp]
uuid: 1ebd6db9-a40e-3433-dc30-192cf133b2f0
---

GNU Emacs is seven years older than Unicode. Support for Unicode had
to be added relatively late in Emacs' existence. This means Emacs has
existed longer without Unicode support (16 years) than with it (14
years). Despite this, Emacs has excellent Unicode support. It feels as
if it was there the whole time.

However, as a natural result of Unicode covering all sorts of edge
cases for every known human language, there are pitfalls and
complications. As a *user* of Emacs, you're not particularly affected
by these, but extension developers might run into trouble while
handling Emacs character-oriented data structures: strings and
buffers.

In this article I'll go over Elisp's Unicode surprises. I've been
caught by some of these myself. In fact, as a result of writing this
article, I've discovered subtle encoding bugs in some of my own
extensions. None of these pitfalls are Emacs' fault. They're just the
result of complexities of natural language.

### Unicode and Code Points

First, there are excellent materials online for learning Unicode. I
recommend starting with [UTF-8 and Unicode FAQ for Unix/Linux][faq].
There's no reason for me to repeat all this information here, but I'll
attempt to quickly summarize it.

Unicode maps *code points* (integers) to specific characters, along
with a standard name. As of this writing, Unicode defines over 110,000
characters. For backwards compatibility, the first 128 code points are
mapped to ASCII. This trend continues for other character standards,
like Latin-1.

In Emacs, Unicode characters are entered into a buffer with `C-x 8
RET` (`insert-char`). You can enter either the official name of the
character (e.g. "GREEK SMALL LETTER PI" for π) or the hexadecimal code
point. Outside of Emacs it depends on the application, but `C-S-u`
followed by the hexadecimal code works for most of the applications I
care about.

#### Encodings

The Unicode standard also describes several methods for encoding
sequences of code points into sequences of bytes. Obviously a selection
of 110,000 characters cannot be encoded with one byte per letter, so
these are multibyte encodings. The two most popular encodings are
probably UTF-8 and UTF-16.

UTF-8 was designed to be backwards compatible with ASCII, Unix, and
existing C APIs (null-terminated C strings). The first 128 code points
are encoded directly as a single byte. Every other character is
encoded with two to six bytes, with the highest bit of each byte set
to 1. This ensures that no part of a multibyte character will be
interpreted as ASCII, nor will it contain a null (0). The latter means
that C programs and C APIs can handle UTF-8 strings with few or no
changes. Most importantly, every ASCII encoded file is automatically a
UTF-8 encoded file.

UTF-16 encodes all the characters from the *Basic Multilingual Plane*
(BMP) with two bytes. Even the original ASCII characters get two bytes
(*16* bits). The BMP covers virtually all modern languages and is
generally all you'll ever practically need. However, this doesn't
include the important [TROPICAL DRINK][td] or [PILE OF POO][poo]
characters from the supplemental ("astral") plane. If you need to use
these characters in UTF-16, you're going to run into problems:
characters outside the BMP don't fit in two bytes. To accommodate
these characters, UTF-16 uses *surrogate pairs*: these characters are
encoded with two 16-bit units.

Because of this last point, **UTF-16 offers no practical advantages
over UTF-8**. Its [existence was probably a big mistake][harmful]. You
can't do constant-time character lookup because you have to scan for
surrogate pairs. It's not backwards compatible and cannot be stored in
null-terminated strings. In both Java and JavaScript, it leads to the
awkward situation where the "length" of a string is not the number of
characters, code points, or even bytes. Worst of all, [it has serious
security implications][sec114]. New applications should avoid it
whenever possible.

### Emacs and UTF-8

**Emacs internally stores all text as UTF-8.** This was an excellent
choice! When text leaves Emacs, such as writing to a file or to a
process, Emacs automatically converts it to the coding system
configured for that particular file or process. When it accepts text
from a file or process, it either converts it to UTF-8 or preserves it
as raw bytes.

There are two modes for this in Emacs: unibyte and multibyte. Unibyte
strings/buffers are just raw bytes. They have constant access O(1)
time but can only hold single-byte values. The [byte-code compiler
outputs unibyte strings][internals].

Multibyte strings/buffers hold UTF-8 encoded code points. Character
access is O(n) because the string/buffer has to be scanned to count
characters.

The actual encoding is rarely relevant because there's little way (and
need) to access it directly. Emacs automatically converts text as
needed when it leaves Emacs and arrives in Emacs, so there's no need
to know the internal encoding. If you *really* want to see it anyway,
you can use `string-as-unibyte` to get a copy of a string with the
exact same bytes, but as a byte-string.

~~~cl
(string-as-unibyte "π")
;; => "\317\200"
~~~

This can be reversed with `string-as-multibyte`), to change a unibyte
string holding UTF-8 encoded text back into a multibyte string. Note
that these functions are different than `string-to-unibyte` and
`string-to-multibyte`, which will attempt a conversion rather than
preserving the raw bytes.

The `length` and `buffer-size` functions always count characters in
multibyte and bytes in unibyte. Being UTF-8, there are no surrogate
pairs to worry about here. The `string-bytes` and `position-bytes`
functions return byte information for both multibyte and unibyte.

To specify a Unicode character in a string literal without using the
character directly, use `\uXXXX`. The `XXXX` is the hexadecimal code
point for the character and is always 4 digits long. For characters
outside the BMP, which won't fit in four digits, use a capital U with
eight digits: `\UXXXXXXXX`.

~~~cl
"\u03C0"
;; => "π"

"\U0001F4A9"
;; => "💩"  (PILE OF POO)
~~~

Finally, Emacs extends Unicode with 256 additional "characters"
representing raw bytes. This allows raw bytes to be embedded
distinctly within UTF-8 sequences. For example, it's used to
distinguish the code point U+0041 from the raw byte #x41. As far as I
can tell, this isn't used very often.

### Combining Characters

Some Unicode characters are defined as *combining characters*. These
characters modify the non-combining character that appears before it,
typically with accents or diacritical marks.

For example, the word "naïve" can be written as *six* characters as
`"nai\u0308ve"`. The fourth character, U+0308 (COMBINING DIAERESIS),
is a combining character that changes the "i" (U+0069 LATIN SMALL
LETTER I) into an umlaut character.

The most commonly accented characters have a code of their own. These
are called *precomposed characters*. This includes ï (U+00EF LATIN
SMALL LETTER I WITH DIAERESIS). This means "naïve" can also be written
as *five* characters as `"na\u00EFve"`.

#### Normalization

So what happens when comparing two different representations of the
same text? They're not equal.

~~~cl
(string= "nai\u0308ve" "na\u00EFve")
;; => nil
~~~

To deal with situations like this, the Unicode standard defines four
different kinds of normalization. The two most important ones are NFC
(composition) and NFD (decomposition). The former uses precomposed
characters whenever possible and the latter breaks them apart. The
functions `ucs-normalize-NFC-string` and `ucs-normalize-NFD-string`
perform this operation.

Pitfall #1: **Proper string comparison requires normalization.** It
doesn't matter which normalization you use (though NFD should be
slightly faster), you just need to use it consistently. Unfortunately
this can get tricky when using `equal` to compare complex data
structures with multiple strings.

~~~cl
(string= (ucs-normalize-NFD-string "nai\u0308ve")
         (ucs-normalize-NFD-string "na\u00EFve"))
;; => t
~~~

Emacs itself fails to do this. It doesn't normalize strings before
interning them, which is probably a mistake. This means you can have
differently defined variables and functions with the same canonical
name.

~~~cl
(eq (intern "nai\u0308ve")
    (intern "na\u00EFve"))
;; => nil

(defun print-résumé ()
  "NFC-normalized form."
  (print "I'm going to sabotage your team."))

(defun print-résumé ()
  "NFD-normalized form."
  (print "I'd be a great asset to your team."))

(print-résumé)
;; => "I'm going to sabotage your team."
~~~

#### String Width

There are three ways to quantify multibyte text. These are often the
same value, but in some circumstances they can each be different.

* *length*: number of characters, including combining characters
* *bytes*:  number of bytes in its UTF-8 encoding
* *width*:  number of columns it would occupy in the current buffer

Most of the time, one character is one column (a width of one). Some
characters, like combining characters, consume no columns. Many Asian
characters consume two columns (U+4000, 䀀). Tabs consume `tab-width`
columns, usually 8.

Generally, a string should have the same width regardless of which
whether it's NFD or NFC. However, due to bugs and incomplete Unicode
support, this isn't strictly true. For example, some combining
characters, such as U+20DD ⃝, won't combine correctly in Emacs nor in
other applications.

Pitfall #2: **Always measure text by width, not length, when laying
out a buffer**. Width is measured with the `string-width` function.
This comes up when laying out tables in a buffer. The number of
characters that fit in a column depends on what those characters are.

Fortunately I accidentally got this right in [Elfeed][elfeed] because
I used the `format` function for layout. The `%s` directive operates
on width, as would be expected. However, this has the side effect that
the output of may `format` change depending on the current buffer!
Pitfall #3: **Be mindful of the current buffer when using the format
function.**

~~~cl
(let ((tab-width 4))
  (length (format "%.6s" "\t")))
;; => 1

(let ((tab-width 8))
  (length (format "%.6s" "\t")))
;; => 0
~~~

### String Reversal

Say you want to reverse a multibyte string. Simple, right?

~~~cl
(defun reverse-string (string)
  (concat (reverse (string-to-list string))))

(reverse-string "abc")
;; => "cba"
~~~

Wrong! The combining characters will get flipped around to the wrong
side of the character they're meant to modify.

~~~cl
(reverse-string "nai\u0308ve")
;; => "ev̈ian"
~~~

Pitfall #4: **[Reversing Unicode strings is non-trivial][reverse].**
The [Rosetta Code][rosetta] page is full of incorrect examples, and
[I'm personally guilty][guilty] of this, too. The other day I
[submitted a patch to s.el][patch] to correct its `s-reverse` function
for Unicode. If it's accepted, you should never need to worry about
this.

### Regular Expressions

Regular expressions operate on code points. This means combining
characters are counted separately and the match may change depending
on how characters are composed. To avoid this, you might want to
consider NFC normalization before performing some kinds of regular
expressions.

~~~cl
;; Like string= from before:
(string-match-p  "na\u00EFve" "nai\u0308ve")
;; => nil

;; The . only matches part of the composition
(string-match-p "na.ve" "nai\u0308ve")
;; => nil
~~~

Pitfall #5: **Be mindful of combining characters when using regular
expressions.** Prefer NFC normalization when dealing with regular
expressions.

Another potential problem is ranges, though this is quite uncommon.
Ranges of characters can be expressed in inside brackets, e.g.
`[a-zA-Z]`. If the range begins or ends with a decomposed combining
character you won't get the proper range because its parts are
considered separately by the regular expression engine.

~~~cl
(defvar match-weird "[\u00E0-\u00F6]+")

(string-match-p match-weird "áâãäå")
;; => 0  (successful match)

(string-match-p (ucs-normalize-NFD-string match-weird) "áâãäå")
;; => nil
~~~

It's *especially* important to keep all of this in mind when
sanitizing untrusted input, such as when using Emacs as a web server.
An attacker might use a denormalized or strange grapheme cluster to
bypass a filter.

### Interacting with the World

Here's a mistake I've made twice now. Emacs uses UTF-8 internally,
regardless of whatever encoding the original text came in. Pitfall #6:
**When working with bytes of text, the counts may be different than
the original source of the text.**

For example, HTTP/1.1 introduced persistent connections. Before this,
a client connects to a server and asks for content. The server sends
the content and then closes the connection to signal the end of the
data. In HTTP/1.1, when `Connection: close` isn't specified, the
server will instead send a `Content-Length` header indicating the
length of the content in bytes. The connection can then be re-used for
more requests, or, more importantly, pipelining requests.

The main problem is that HTTP headers usually have a different
encoding than the content body. Emacs is not prepared to handle
multiple encodings from a single source, so the only correct way to
talk HTTP with a network process is raw. My mistake was allowing Emacs
to do the UTF-8 conversion, then measuring the length of the content
in its UTF-8 encoding. This just happens to work fine about 99.9% of
the time since clients tend to speak UTF-8, or something like it,
anyway, but it's not correct.

### Further Reading

A lot of this investigation was inspired by JavaScript's and other
languages' Unicode shortcomings.

* [UTF-8 and Unicode FAQ for Unix/Linux][faq]
* [Hacking with Unicode][sec]
* [jsesc](https://github.com/mathiasbynens/jsesc)
* [java.lang.Character Unicode Character Representations][java]
* [GNU Emacs Lisp Reference Manual: Strings and Characters][man]

Comparatively, Emacs Lisp has really great Unicode support. This isn't
too surprising considering that it's primary purpose is for
manipulating text.


[faq]: http://www.cl.cam.ac.uk/~mgk25/unicode.html
[td]: http://www.fileformat.info/info/unicode/char/1f379/index.htm
[poo]: http://www.fileformat.info/info/unicode/char/1F4A9/index.htm
[sec114]: https://speakerdeck.com/mathiasbynens/hacking-with-unicode?slide=114
[sec]: https://speakerdeck.com/mathiasbynens/hacking-with-unicode
[harmful]: http://www.utf8everywhere.org/
[internals]: /blog/2014/01/04/
[elfeed]: /blog/2013/09/04/
[reverse]: https://github.com/mathiasbynens/esrever
[guilty]: /blog/2012/11/15/
[rosetta]: http://rosettacode.org/wiki/Reverse_a_string
[patch]: https://github.com/magnars/s.el/pull/58
[java]: http://docs.oracle.com/javase/7/docs/api/java/lang/Character.html#unicode
[man]: http://www.gnu.org/software/emacs/manual/html_node/elisp/Strings-and-Characters.html
