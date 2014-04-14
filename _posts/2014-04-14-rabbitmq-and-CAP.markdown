---
title: RabbitMQ and CAP
layout: default
---

## Which properties out of CAP does RabbitmQ pick?

It's complicated. Let's start by clearing up a misunderstanding -- one
I have assumed on your behalf.

Recall that the CAP theorem is stated in terms of a distributed system
which reads and writes to a register. The theorem says that if the
system uses an asynchronous network, and the network can lose packets,
then the system cannot have 100% availability *and* 100% consistency.
The unofficial [CAP theorem FAQ][] explains this very well.

With that in mind, let me state the crucial misunderstanding: that CAP
theorem implies you can *pick two* out of availability, consistency,
and partition tolerance. It's tempting to apply the cute
two-out-of-three precis, but here it's just not appropriate.

The term "partition tolerance" is partly to blame; the property is
more accurately described as "possibility of partitions". In practice,
it is absolutely the case that networks are asynchronous and you don't
get a choice about whether partitions occur. Certainly they do in
RabbitMQ clusters and federations, or between RabbitMQ clients and
servers.

Another facet of this misunderstanding is that you can *choose* to
have either 100% consistency or 100% availability. CAP theorem is an
impossibility result: it says that you can't have everything. It
doesn't say anything about how close you can get otherwise. In
practice, distributed systems have many and varied failure modes, not
all of which are related to partitions, and may compensate for
partitions in very different ways depending on the circumstances.

## Where does RabbitMQ lie in relation to the CAP theorem?

The landscape around CAP is labyrinthine, and RabbitMQ (along with all
other distributed systems of any practical interest) is not reduable
to an atomic register. We can try to characterise a few of the
mechanisms at play, however.

In the following I've identified arrangements we can think of as
distributed systems, and what it is that is being synchronised in the
system (i.e., the state that takes the place of the atomic
register).

### Clients and servers and message transfer

You can regard a RabbitMQ client connected to a RabbitMQ server as a
distributed system. The connection between the client and server is
based on TCP, which gives us a kind of all-or-nothing possibility of
partitions: either no packets are considered lost, or we (eventually)
detect a connection failure.

In this arrangement, not unlike TCP itself, it's the receipt of
messages that is being synchronised. When a connection fails, the
client or server cannot be sure of how much the other party heard
before communication stalled. An acknowledgment is used to indicate
the receipt of each message. Because the acknowledgment may also fail
to appear, a message is resent until it has been acknowledged. (Base
AMQP doesn't actually have a full mechanism for this, but RabbitMQ
thoughtfully adds the missing bit, acknowledgments to publishers)

How does this relate to CAP? If you had an out-of-band way of asking
the client and the server where they thought the message was (say if
you were relaying rows from one database to another and you could look
at both databases), then so long as the partition lasted, one of the
client or server may not be able to answer definitively whether a
message has been received. They either have to wait indefinitely until
the connection is re-established, or give a best guess; for example,
to regard the message as received at the server once it's been read
from the socket (in which case the client may have a different idea,
if it's not heard back from the server).

### Clustering and schema information

This is more in the line of what most people think about when they
think about CAP theorem, since it's recognisably a database. In
RabbitMQ, the schema information (that is, the existence and
properties of exchanges, queues, bindings and so on) is in a database
shared among the cluster nodes.

The particular piece of kit running this database, `mnesia`, gives
applications a choice of transactionality for writes and reads. In
RabbitMQ, most (essentially all) writes are ACID, and reads are
sometimes transactional and sometimes
non-transactional. Non-transactional reads are used when the
guarantees of the AMQP protocol, which are fairly loose for schema
operations, cannot be observably contravened. Since the scope of
"observability" is generally the channel, and these run in a single
process, in general this is safe.

In any case, what we're concerned with here is the effects partitions
have. `mnesia` and a RabbitMQ cluster's idea of partition coincide,
since they both use Erlang distribution. There's no distinction in the
short term between a node crashing and a node failing to communicate,
and `mnesia` will keep running, and
[attempt to recover][mnesia recovery] after communication is
restored. In this sense it is trying to remain available, at the risk
of nodes diverging, though the recovery mechanisms (replaying
transactions, basically) mitigate this to an extent. However, it is
still possible for `mnesia` to reach an irreconcilable, inconsistent
state as the result of a partition; in this situation it punts the
decision of what to do, to the application.

RabbitMQ can likewise detect a partition event (and tell you about it
in the management console). Its recovery mechanism amounts to choosing
some variety of [<abbr>STONITH</abbr>][stonith].

If the distributed system is the cluster, and the state is the schema,
we can note that RabbitMQ (and `mnesia`) will try to keep returning
responses in the case of a partition, rather than waiting for it to be
reconciled or refusing to answer. That is not to say it is 100%
available however: nodes that are restarted may drop connections and
resynchronise state, and thereby fail to answer requests.

### Clustering and message transfer

The principal difference between this case and that of a single server
is that there is potentially an extra level of state synchronisation:
mirrored queues. This mechanism actively mitigates queue failures by
replicating queue operations to other nodes. In RabbitMQ's
documentation, this is called "high availability": if the master queue
process crashes, one of replicas can take over.

What happens when there is a partition? If you're connected
(transitively) to the node on which the master queue lives, you can
continue. If you're not, you may fail to get an answer; in other
words, the queue is unavailable to you. "High availability" is really
"high consistency" with respect to partitions.

## What can we say about where RabbitMQ lies in relation to CAP theorem?

Only, in the end, that it's complicated. As with most systems of
practical interest, it's better to talk about the specific behaviour
and guarantees, rather than give a summary characterisation in the
terms of an ideal.

[CAP theorem FAQ]: http://henryr.github.io/cap-faq/
[mnesia recovery]: http://www.erlang.org/doc/apps/mnesia/Mnesia_chap7.html#id78623
[stonith]: http://en.wikipedia.org/wiki/STONITH
[mirrored queues]: http://www.rabbitmq.com/ha.html#behaviour
