---
# published: false
layout: post
title: Multiple dispatch in JavaScript
---

Towards the end of last year, while hacking on user interface for
[dolt](https://github.com/squaremo/dolt), I started looking at
<abbr>CLIM</abbr>, the <defn>Common LISP Interface
Manager</defn>. Among other unearthed arcana, it makes heavy use
of <abbr>CLOS</abbr> (the <defn>Common LISP Object System</defn>), in
particular [generic
functions](http://en.wikipedia.org/wiki/Generic_function).

I thought it would be an interesting experiment to see if multiple
dispatch helped with programming in JavaScript. Since JavaScript
doesn't have classes, as such, I couldn't quite mimic
<abbr>CLOS</abbr>; however, I remembered
[Slate](http://slatelanguage.org/), which is a dynamic,
prototype-based language with multiple dispatch built in. And happily,
there's a [paper](http://files.slatelanguage.org/doc/pmd/ecoop.pdf)
describing how that's implemented.

The idea is to build up a score for each method, based on how close
(in the delegation chain) its definition of each argument is to the
values supplied at invocation. In CLOS the delegation chain is largely
static, so the system can linearise methods as they are defined. In
Slate, the delegation chain is dynamic, so you have to store the
method information in the objects themselves and look them up when
dispatching.

JavaScript is a bit different to Slate. It's only halfway
prototype-based: an object's prototype is supplied via the
constructor, or as an argument to `Object.create`; i.e., it's assigned
at the time of creation. So, it's not quite as dynamic, but moreso
than CLOS. Nonetheless, it's possible (and common usage) to use
constructors and prototypes to create chains of delegation that also
look like type hierarchies -- or just outright type hierarchies.

Here's a naïve implementation of the central method lookup algorithm:

{% gist squaremo/5086573 method_lookup.js %}

Of the free names there, `get_table` gets the method lookup table for
a value and role (argument position), `delegate` gets a value's
prototype, and `METHODS` is a map of all methods defined. More about
those in a sec. There's also `selector` in the lexical closure, which
is a gensym based on a name supplied for the procedure. (Actually you
could just take a look at [the whole
thing](https://github.com/squaremo/js-pmd/blob/master/index.js) if you
want, it's not long)

There's a handful of translation peculiarities.

One is that JavaScript has value boxing for numbers, strings, and
booleans. The semantics are that if an unboxed value is treated like
an object (e.g., if you assign a property to it), a new boxed value is
created, the operation done with the boxed value, then the boxed value
is thrown away. Since I need to store methods with the values on which
they are specialised, I have to keep maps for the unboxed value types;
luckily they are detectable using `typeof` (`typeof("foo") ===
'string'; typeof(new String("foo")) === 'object'`). That's the purpose
of `get_table`.

However I do want e.g., a literal string to have a place in the type
hierarchy; so, in `delegate` (which gets the prototype of a value), I
use `Object(...)` before asking for the prototype. For objects this is
a no-op; and, for unboxed values it'll return a throwaway object, but
that's fine since I want the prototype not the value itself.

Another is due to JavaScript's constructor mechanism, which is a bit
of a headache.

An aside: the `constructor` property of objects is misleading. It's
not usually a property of a constructed object, but rather, a property
given to the automagically generated prototype of a function, which is
then 'inherited' by the object.

{% gist squaremo/5086573 constructor_inheritance.js %}

If, then, you do what comes naturally and assign to a function's
prototype property in order to create a chain of delegation, the
constructor property is inherited from whatever you assigned, and not
the automagic prototype.

{% gist squaremo/5086573 constructor_inheritance_2.js %}

Anyway. This constructor thing gives me a choice: since they are often
used in the delegation chain style, they make a nice way of naming
types. That is, instead of specialising on `MyConstructor.prototype`,
you can specialise on the constructor `MyConstructor`. The trade is
that you can't specialise on function values -- it'll always assume
you were mentioning a function as a constructor. (You can still use
`Function` if you want to specialise on functions. Just not individual
function values.)

Oh! I didn't say whether multiple dispatch was helpful or not. Maybe
next time.
