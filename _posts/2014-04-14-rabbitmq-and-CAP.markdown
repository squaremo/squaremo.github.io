---
title: RabbitMQ and CAP
layout: default
---

**Q: Which properties out of CAP does RabbitMQ pick?**

Let's start by clearing up a misunderstanding -- one I have assumed on
your behalf (I'm sorry).

Recall that the CAP theorem is stated in terms of a distributed system
which reads and writes to a register. The theorem says that if the
system uses an asynchronous network, and the network can drop packets,
then the system cannot have 100% availability *and* 100%
consistency. "Available" means that all requests get an answer;
"Consistent" means operations are atomic. The unofficial
[CAP theorem FAQ][] explains this all quite well.

With that in mind, let me state the rhetorical misunderstanding: that
CAP theorem says you can *pick two* out of availability, consistency,
and partition tolerance. It's tempting to apply the cute
two-out-of-three template, and early renditions of the CAP theorem did
so (including the paper in which appears the proof!) but it's just not
appropriate for real world systems.

The term "partition tolerance" is partly to blame; the property in
question is more accurately described as "possibility of
partitions". In practice, it is absolutely the case that networks are
asynchronous and you don't get a choice about whether partitions can
occur. Certainly they do in RabbitMQ clusters and federations, or
between RabbitMQ clients and servers.

Another facet of this misunderstanding is that you can always choose
to give up consistency or availability to gain the other. CAP theorem
is an impossibility result: it says that you can't have everything. It
doesn't say how close you can get otherwise. In practice, distributed
systems have many and varied failure modes, not all of which are
related to partitions, and may compensate for partitions in very
different ways depending on the circumstances.

**Q: How can RabbitMQ be characterised in CAP theorem terms?**

RabbitMQ is not (easily?) reducable to atomic registers. We can try to
characterise a few of the mechanisms at play, however.

In the following I've identified arrangements we can think of as
distributed systems, what it is that is being synchronised in the
system (i.e., the state that takes the place of the register), and
then examined whether we can consider it available or consistent.

#### Sending and receiving messages

You can regard RabbitMQ clients sending and receiving messages through
a RabbitMQ server as a distributed system. The connection between a
client and the server is based on TCP, which gives us a kind of
all-or-nothing possibility of partitions: either no packets are
considered dropped, or we (eventually) detect a connection failure,
which is in effect a total partition.

In this arrangement, like TCP itself, it's the receipt of messages
that is being synchronised. When a connection fails, the client or
server cannot be sure of how much the other party heard before
communication stalled. An acknowledgment is therefore used to indicate
the receipt of each message. Because the acknowledgment may also fail
to appear, a message is resent until it has been acknowledged. (Base
AMQP doesn't actually have the full mechanism for this, but RabbitMQ
thoughtfully adds the missing bit, acknowledgments to publishers).

How does this relate to consistency and availability? If you had an
out-of-band way of asking the client and the server where they thought
the message was (say if you were relaying rows from a database, and
you could compare that to queue contents), then so long as the
partition lasted, either of the client or server may not be able to
answer definitively whether a message has been received. They either
have to wait indefinitely until the connection is re-established and
acknowledgments sent and received, or give a best guess.

#### Clustering and schema information

In RabbitMQ, the schema information (by which I mean the existence and
properties of exchanges, queues, bindings and so on, but not the
messages) is in a database shared among the cluster nodes. This is
more in the line of what most people think about when they think about
CAP theorem, since it's recognisably a database.

The particular piece of kit running this database, called "mnesia",
gives applications a choice of transactionality for writes and
reads. In RabbitMQ, almost all writes are ACID, some reads are
transactional and some reads are non-transactional. Non-transactional
reads are used when the guarantees of the AMQP protocol, which are
fairly loose, cannot be observably contravened. Since the scope of
"observability" is the AMQP channel, and each of these runs in a
single Erlang process, in general this is safe with respect to AMQP's
notion of consistency. I should point out though that the whole system
is not consistent in the sense of CAP theorem: since some reads are
non-transactional, they can overlap arbitrarily with writes.

In any case, what we're concerned with here is the effect of
partitions. mnesia and a RabbitMQ cluster's idea of partition
coincide, since they both use Erlang distribution. In the event of a
node failing to communicate, and mnesia will keep running, and
[attempt to recover][mnesia recovery] after communication is restored
(which may be after the remote node is restarted).

In this sense it is trying to remain available, at the risk of nodes
diverging, though the recovery mechanisms (replaying transactions,
basically) mitigate the divergence to an extent. However, it is still
possible for mnesia to reach an irreconcilable, inconsistent state as
the result of a partition; in this situation it punts the decision of
what to do, to the application.

RabbitMQ can likewise detect a partition event. Its recovery mechanism
amounts to choosing some variety of [<abbr>STONITH</abbr>][stonith]:
you can tell it to ignore the partition, to stop nodes that are judged
to be in the minority, or restart nodes that are judged to be in the
minority. In all these cases, any restarted nodes have their replica
of the database overwritten, so writes may be lost. It is not
available in the CAP sense, because the restarting nodes can drop
requests on the floor.

There is still a meaningful sense in which a cluster is available; why
have clustering at all otherwise. If your connection drops, you can
still connect to another node and get things done.

#### Clustering and messages

In a cluster, a queue may be located at a single node or mirrored to
several nodes.

The principal difference between this case and that of a single server
is that there is an extra level of state synchronisation: the mirrored
queues. This mechanism actively mitigates queue failures by
replicating queue operations to other nodes. In RabbitMQ's
documentation, this is called "high availability": if the master queue
process crashes, one of replicas can take over, and consumers can
continue to get messages.

What happens when there is a partition? If you're connected
(transitively) to the node on which the master queue lives, you can
continue. If you're not, you may fail to get an answer; in other
words, the queue is unavailable to you until you can connect to
wherever the master is. So this is really "high consistency" with
respect to partitions.

Again though, there is a sense in which mirrored queues are available:
you can reconnect and make progress.

**Q: What can we say about where RabbitMQ lies in relation to CAP theorem?**

As with most systems of practical interest, it's better to talk about
the specific behaviour and guarantees, rather than give a summary
characterisation. RabbitMQ maintains properties that we can refer to
as "consistent" and "available" in various combinations, but they
don't correspond to the terms as used in the CAP theorem. Nonetheless,
they can be useful properties!

It is perhaps more fruitful to consider the more broadly scoped
distributed system of your applications *connected to* RabbitMQ, in
effect reducing RabbitMQ to the part of (somewhat, but not entirely,
reliable) transmission medium.

[CAP theorem FAQ]: http://henryr.github.io/cap-faq/
[mnesia recovery]: http://www.erlang.org/doc/apps/mnesia/Mnesia_chap7.html#id78623
[stonith]: http://en.wikipedia.org/wiki/STONITH
[mirrored queues]: http://www.rabbitmq.com/ha.html#behaviour
