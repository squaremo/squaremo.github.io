---
#published: false
layout: post
title: Multiple dispatch in JavaScript, part deux
---

In the [last post](multiple-dispatch-in-javascript.html) I described
some of the specifics of *implementing* multimethods in JavaScript,
but I didn't talk about *using* multimethods in JavaScript or give any
examples. Here I'm going to demonstrate a few uses of multimethods.

Before I start, one peculiarity I didn't mention in the previous post
is garbage collecting methods. Method lookup tables are kept as
properties of the "type" objects with which they are defined. This
means that if the object gets collected, so does the method table,
which is good. Less good though, is that if the method has other
arguments: they will retain an entry in their method tables, even
though that method can never be invoked.

For that reason it is important to restrict method definitions to
long-lived objects; in general this is fine since it's the objects
representing types that you care about -- in other words, those that
are supposed to hang around, so other objects can be based on them.

Anyway, yes hello. Examples of multiple dispatch.

<script src="https://gist.github.com/squaremo/5305729.js?file=types.js">
</script>

This first one is for decoding JSON values. The scheme is pretty easy
to figure out: a '!' property in the JSON value is a kind of reader
syntax, giving a type name. `decodeValue` just determines whether the
value uses this special encoding or not. Then there's two procedures:
one with methods specialising on the kind of "normal" object, since
arrays `typeof` to `'object'`; and, another which has methods
specialising on the type name of an encoded object.

This could all be done with a single function, of course. However,
this way I can add a special type elsewhere in the code, which would
otherwise require some kind of registration mechanism.

<script src="https://gist.github.com/squaremo/5305729.js?file=widgetize.js">
</script>

This example is a procedure for making a widget (something that will
be rendered into the web page) given a value of the kind decoded in
the example above. `render` is a procedure created elsewhere that has
methods to render specific widgets to DOM nodes. The idea is that a
value will be `widgetize`d, then the result is `render`ed when
necessary.

There is some tricksiness in the interplay between these two
procedures.

For the sake of not defining more types, primitives (that is strings,
numbers etc.) are their own widgets. We want to define `widgetize` for
`Object` later on, so I have to define methods for the primitive types
individually.

(To be honest it's a bit of a pain that `Object` is both a common type
of value and the supertype of almost everything. In any case, note
that using multimethods has the effect of flattening out what might
otherwise be a nested if-then-else statement -- imagine if these
methods all had their own specific implementations, and there were
three arguments rather than two.)

Then, so that those primitive values can act as widgets, at line 30
the `render` procedure is given a default method that will simply wrap
the stringified value in an HTML element. At line 40 this is
specialised for strings, to put double quote marks around them.

In line 17, `widgetize` is given a method that effectively resends
invocations with one argument to invocations with two arguments,
saving extra definitions.

You may have noticed that all the `render` methods expect a function
as the second argument (it's supplied with a function to output DOM
nodes), so the only argument they specialise on is the first; and,
given that fact, why don't I just use a regular single-dispatch
method?

One reason is that I can specialise on things other than objects;
e.g., literal strings, as in `decodeSpecial` above. Another is that
the multimethods are *values* and as such I can pass them around,
e.g., as arguments to a function (although you can always construct a
function that will invoke the appropriate property, I suppose). Also
since the multimethods are values, there's no need to assign a
property of a global object (say `String`) if I want some new
polymorphic procedure (say `indexOf`).

Again, this opens up the possibility of adding kinds of widget
elsewhere. And in fact, in another file, I have these:

<script src="https://gist.github.com/squaremo/5305729.js?file=repl.js">
</script>

These are special values that aren't from encoded JSON; `Waiting`, for
example, is just a placeholder value (it's rendered to one of those
AJAX spinners you have now).

I ought to note that most of the code doesn't use multiple
dispatch. Quite a lot of it uses regular old single dispatch. The main
uses of multiple dispatch are to

 * allow code to be extended after the original definition
 * avoid adding properties to top-level objects e.g., `String`
 * flatten complicated dispatch into a more tabular form
