---
layout: post
title: AMQP framing and codec in amqplib
---

Nestled amongst the treasure hoard that is AMQP 0-9-1 lie no fewer
than four encoding schemes, all _slightly different_, with overlapping
sets of primitive types (which are helpfully given different names in
different places). Each of these needs its own _slightly different_
approach, although certain things are common of course. What follows
is an explanation of the various encodings, their quirks, and their
implementation in [amqplib][], my AMQP client library for Node.JS.

### Parsing frames

At the bottom layer, bytes on the wire are in *frames*, of a handful
of set layouts. Each frame looks like this:

    Frame format:
    
    0      1         3               7              size+7
    +------+---------+-------------+ +------------+ +-----------+
    | type | channel | size        | | payload    | | frame-end |
    +------+---------+-------------+ +------------+ +-----------+
     octet  short     long            size octets    octet

The `type` identifies the kind of frame, and thus the meaning and
layout of the payload. The 16-bit `channel` identifies a multiplexed
stream (more on this another time). Connection-level frames --
heartbeats and some performatives -- always have a channel of `0` (so
you could argue that `channel` ought to be part of the next
layer). The `frame-end` is a delimiter of set value `0xCE`, which is a
intended to act as a check that the frame size really is the frame
size; of course, the byte in that position might have that value by
coincidence. (Luckily, the byte spent on the redundant frame delimiter
is more than saved elsewhere by two _slightly different_ ridiculous
bit-packing algorithms[1](#note1).)

In amqplib, the incoming byte stream is of course a `Readable`;
amqplib uses a [bitsyntax][] pattern to break it into frames,
proceeding only when it has a full and correctly-delimited frame. It
tests the size and delimiter explicitly rather than including them in
the pattern -- we don't want to get a huge, bogus size and read from
the socket forever trying to accumulate enough bytes.

<script src="https://gist.github.com/squaremo/7373943.js?file=frame_parse.js">
</script>

If the match fails (returns `false`) an outer loop reads the next
chunk of bytes and tries again with all the bytes thus far
collected. It is perhaps slightly sub-optimal to try the full match
every time new bytes come in. An improvement might be to have distinct
header-reading and payload-accumulation states.

By the way, using bitsyntax is just a compact and convenient means of
code generation; one could certainly write equivalent code by hand.

### Decoding and encoding methods and headers

Depending on the frame type, the payload will contain nothing (for
heartbeats), message content, one of several kinds of AMQP method (a
command), or one of one kind of message header. These latter two have
similar encoding schemes with a statically-defined sequence of fields
per method or header, the encoded values of which are simply
concatenated.

Since I have all the method and header definitions in a JSON file, I
can mechanically generate encoding and decoding procedures for them. I
could hand-code them, but there are quite a few methods and it would
take a long and boring time, and I doubt there are any benefits to
doing so, optimisation- or other-wise.

The definitions look like this:

    {"id": 10,
     "arguments": [
       {"type": "octet", "name": "version-major", "default-value": 0},
       {"type": "octet", "name": "version-minor", "default-value": 9},
       {"domain": "peer-properties", "name": "server-properties"},
       {"type": "longstr", "name": "mechanisms", "default-value": "PLAIN"},
       {"type": "longstr", "name": "locales", "default-value": "en_US"}],
     "name": "start",
     "synchronous" : true}

A method frame payload starts with a 32-bit integer denoting the
specific method, then the encoded fields for that method concatenated
together. Here's an encoded ConnectionStart method:

<script src="https://gist.github.com/squaremo/7373943.js?file=frame_eg">
</script>

After some unsavoury string concatenation (view through your fingers
[here][amqplib-generate]), something like the following decoder
procedure is generated for each method:

<script src="https://gist.github.com/squaremo/7373943.js?file=decode_start.js">
</script>

This is deliberately simple-minded, using local variables as registers
of a sort, to keep the code-generating code uniform per stanza and
make debugging easier. In principle. The result is run through uglify
to tighten it up; or, at least, to pretty-print it.

Encoder procedures are also generated. These are not symmetric to the
decoders: they generate a whole frame at once. Otherwise, the method
fields would just have to be concatenated with the few bytes in the
frame header and the frame delimiter at the end, involving another
buffer copy operation.

A few methods, by virtue of the types of their fields, have a fixed
size. For these I allocate an exactly-sized buffer to encode
into. Most, however, contain at least one string or table, so need a
dynamically-sized buffer. Since there's no such thing (well at least,
not without me implementing one), I use a "safely-sized" buffer, one
that is very likely to be big enough in practice. There's a few
improvements I think can be made in this respect:

 - Given the values to be encoded, I could allocate a buffer to
   size. A complication is tables (and arrays, though they only appear
   inside tables), for which the size can only be calculated with an
   encoding pass. Still, since I encode those into their own buffers
   anyway, I could do that first then allocate the whole thing.
   
 - Similarly, encoding frames or even series of frames into a single
   buffer is bound to be more efficient than encoding pieces into
   individual frames then constructing from there. When sending a
   message, there are at least two, and usually at least three, frames
   (the deliver method, the headers, and one or more content
   frames). It may be worth making some special cases for writing all
   of these at once.

 - In the absence of the above, I ought at least to detect if I'm
   going to overrun the "safely-sized" buffer, even if it's
   unlikely. In AMQP 0-9-1 frames have a maximum size, negotiated per
   connection, and it is not specified what is supposed to happen if a
   method cannot be encoded within a single frame. So one *could* say
   I am acting in the spirit of the protocol.

### Mapping primitive types to JavaScript

AMQP 0-9-1 values inhabit a smallish set of types, including UTF8
strings, integers of various widths, floats, a couple of wildcards
`decimal` and `timestamp`, maps (called 'field tables'[2](#note2)) and
arrays (called 'field arrays').

In method fields the types are specified, so the domains are known and
can be checked when encoding. `timestamp` and `decimal` don't appear
as method fields, so I don't have to deal with those there.

Some method fields are tables: these are maps containing arbitrary
keys and values of the types above, including timestamps, decimals,
tables themselves, and arrays of arbitrary values. The obvious choice
for table values is to accept objects. The values *in* tables present
a problem though: they will be arbitrary JavaScript values and I have
to decide for each what type it will be given.

Since JavaScript has only one number type, 64-bit floats, I choose the
smallest encoding that includes the supplied number. I'm relying on
the other end -- either the server or a client somewhere -- promoting
the number if it's expecting something wider or floatier. If
JavaScript number is greater than `2^50`, it's impossible to determine
if the number is "supposed" to be an integer or floating point, so it
gets encoded as a double. An improvement here would be to accept
64-bit integers from one or more big-number libraries.

Strings in AMQP method fields are short -- 8-bit-sized UTF8. These
correspond nicely to JavaScript strings. In table fields, there are
only 32-bit-sized `longstr`s of no particular string encoding, and
32-bit-sized byte arrays which are like, *totally* different to
`longstr`s. In tables and arrays, strings get encoded as AMQP
`longstr`s (no `shortstr`s allowed as values sorry), and decoded as
UTF8 strings[3](#note3). Buffers get encoded as byte arrays and vice
versa.

Because some JavaScript values may represent AMQP values of more than
one type, there is a type tagging mechanism: wrapping any value in an
object with a `'!'` property giving the AMQP type forces it to be
encoded as that type. For example, one could supply a table as the
JavaScript value

    {
        received: {'!': timestamp,
                   'value': +new Date}
    }

A `decimal` has no direct JavaScript equivalent, so is represented as
an object `{'!': 'decimal', digits: uint32, places: uint8}`.
Intriguingly, the digits part is defined in the AMQP specification as
an unsigned integer, so one cannot encode negative decimals. Now
that's optimism.

### Testing

Another benefit of a machine-readable protocol specification is that I
can generate test cases. I do so using [claire][], a property-based
testing library. I have to define all the base types:

<script src="https://gist.github.com/squaremo/7373943.js?file=base_types.js">
</script>

The sum combinator `claire.choice`, has derivatives `claire.Object`
and `claire.Array`, which I can use for field-tables and field-arrays
respectively:

<script src="https://gist.github.com/squaremo/7373943.js?file=sum_types.js">
</script>

With the product combinator `claire.sequence`, I can use the
specification to generate the methods, frames, and so on.

<script src="https://gist.github.com/squaremo/7373943.js?file=product_types.js">
</script>

Now that I have representations of the methods, I can construct traces of frames, and test that they are encoded and parsed correctly.

<script src="https://gist.github.com/squaremo/7373943.js?file=test_trace.js">
</script>

In the above, each generated trace is encoded, then partitioned into
chunks in different ways, to make sure the parsing code deals with
irregular packets as might come in off the wire.

-------

#### Footnotes

<a name="note1">[1] "ridiculous bit-packing"</a>

1) The presence of header fields are given in a `2n byte` bitset. If a
bit is not set, the corresponding field is skipped. For bit-typed
fields absence is overloaded to mean `false`. The lowest bit in each
two byte segment is a continuation bit, which if set, signifies
another two bytes of bitset. None of the one kind of message header
frame has more than fifteen fields, making this embellishment
pointless. Oh, and the number of fields for a header frame is
statically known anyway.

2) Consecutive bit-typed (boolean) fields in methods are packed into
consecutive bits in one or more bytes. To be fair, there are a couple
of methods with consecutive bit-typed fields, e.g.,
`ExchangeDeclare`. So perhaps this is not so ridiculous. By contrast,
booleans in tables and arrays take two bytes a pop: one to mark the
value as a boolean, and one to encode the value.

I'll stop now.

<a name="note2">[2] Field tables and field arrays</a>

I don't know why these are called what they are called. Maybe because
they are tables (maps) or arrays of values that otherwise appear as
method fields? Or because they are only used in paddocks? Or because
they outrank other officer tables and arrays. Oh, *tables* of
*field* values. Yeah, maybe.

<a name="note3">[3] Strings are long but also short and sometimes
UTF8</a>

Methods can contain fields of either `longstr` (which are not required
to be UTF8) and `shortstr` (which are). Since, in principle, I might
get a `longstr` field value that is not UTF8, I have to treat
`longstr`s in method fields as byte buffers. If an object to be
encoded as a field-table contains a string value, however, I have no
choice but to encode it as a `longstr`, since `shortstr` values do not
appear in tables.

Please send help. Not to me though -- send it back in time, to the
AMQP authors.


[amqplib]: https://github.com/squaremo/amqp.node/
[bitsyntax]: https://github.com/squaremo/bitsyntax-js/
[amqplib-generate]: https://github.com/squaremo/amqp.node/blob/b33afef6763011637e9fa9bed133351383a9823b/bin/generate-defs.js
[claire]: https://github.com/hifivejs/claire
