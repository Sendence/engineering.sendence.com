+++
title = "How the end-to-end back-pressure mechanism inside Wallaroo works"
date = 2018-04-03T12:00:00-06:00
draft = false
author = "slfritchie"
description = "A detailed look at how several back-pressure mechanisms inside Wallaroo create an end-to-end back-pressure mechanism to protect Wallaroo from overload by high-volume data sources.  Part two of a two-part series."
tags = [
    "back-pressure",
    "pony",
    "testing",
]
categories = [
    "Exploring Wallaroo Internals"
]
+++

## Introduction to part two

This is part two of a two-part series on how a Wallaroo system reacts
to workload demands that exceed Wallaroo's capacity, i.e., how
Wallaroo reacts to overload.  Part one, which defines what overload is
and summarizes overload mitigation techniques, can be found here:
["Some Common Mitigation Techniques for Overload in Queueing Networks"](https://blog.wallaroolabs.com/2018/03/some-common-mitigation-techniques-for-overload-in-queueing-networks/).

Wallaroo uses several back-pressure techniques to limit message queue
sizes of all actors within the system.  Together, these techniques form an
end-to-end mechanism that protects Wallaroo from high-volume data
sources.  This article describes how these back-pressure mechanisms are
implemented by and integrated with Wallaroo.  Also, we take a peek at
a big change to Wallaroo's internals that we hope will change Wallaroo
in several good ways.

### Where Does the Term Back-Pressure Come From?

The term "back-pressure" originally described the resistance to
forward flow of a gas or liquid in a confined space, such as an air duct or
pipe.  The gas (or liquid) experiences friction along the sides of the
duct (or pipe).  It is easy to create an experiment to experience
different amounts of back-pressure by a gas.
For example, using your lungs and
mouth, try to blow as much air as you can --
as quickly as possible -- through the following tubes for comparison:

* A small cocktail straw with an opening only 1 or 2 millimeters in diameter
* A regular drink straw, for example, a straw from the soda fountain
  at McDonalds
* The cardboard tube from a roll of paper towels or toilet paper

### TCP's Back-Pressure Mechanism: Sliding Window Flow Control

The TCP protocol uses a flow control mechanism called a "sliding
window" to prevent the sender from using too much of the receiver's
memory and CPU resources.  Wallaroo integrates TCP's flow
control into its back-pressure system.
This isn't the time & place to describe TCP's sliding window flow
control in detail.  However, a picture
is worth a thousand words ([Figure 25.7](#figure257) below).
Let's take a look.

First, let's make some assumptions about a hypothetical
TCP connection.

* The TCP connection has already been established.
* The sender application is Wallaroo.
* Wallaroo is attempting to send 50,000 bytes of data to the receiver
  as quickly as possible.
* The receiver is a TCP data sink application that stores Wallaroo's
  computation output in 1,000 byte chunks.
* The receiver's disk drive is very old and slow.  Writing a single
  1,000 byte block can cause the application to pause
  for several seconds.

[Figure 25.7](#figure257) tracks one of the two advertise windows of
the sliding window protocol for the connection.  We look at one
direction only; TCP maintains a separate sliding window for data
sent in the other direction.

<a name="figure257"></a>
![Figure 25.7 from "Fundamentals Of Computer Networking And Internetworking" by Prof. Douglas Comer of Purdue University](https://www.cs.csustan.edu/~john/classes/previous_semesters/CS3000_Communication_Networks/2015_02_Spring/Notes/CNAI_Figures/figure-25.7.jpeg)

From the receiver's point of view:

* The advertise window starts at 2,500 bytes.

* Packets start arriving from the sender.  The app has read 2,500
  bytes quickly but then does not read any more network data for a
  while: the app
  is put to sleep by the kernel due to very slow file I/O speeds.
* The disk finally finished, so the app wakes up, reads 2,000 more bytes
  from the TCP connection, writes
  the new data to disk, then goes to sleep a second time,
  waiting for disk I/O to finish.
* The app wakes up and then reads another 1,000 bytes from the TCP connection.

The advertise window falls to zero twice in this example.
When the advertise window is zero, the Wallaroo sender must stop transmitting
and wait for an "ack" message with a non-zero advertise window.

How does Wallaroo know the receiver's advertise window size?  That
level of detail is not available through the POSIX network socket API.

A POSIX system can tell a user application when the
advertise window size is exactly zero.
If the socket has been configured in non-blocking I/O mode,
then a `write(2)` or `writev(2)` system call to the socket will fail
with the `errno` value of `EWOULDBLOCK` or `EAGAIN`, depending on the
operating system.  (I will assume the `EWOULDBLOCK` value for this article.)
The `EWOULDBLOCK` condition
tells the application that the kernel was not immediately able to send
any data on the socket.

When Wallaroo sends data to a sink with the `writev(2)` system call,
and the call fails with `EWOULDBLOCK` status,
then we know that the sink is slow.  We don't
know why the receiving sink is slow.  The `EWOULDBLOCK` status can be
caused by other conditions, such as lack of buffer space in the
sender's kernel.  However, we do know that
Wallaroo is sending data faster than the TCP sink can read it and/or
faster than the sender's OS can handle it.  We know that we have a
flow control problem, and we need to create back-pressure to fix the
problem.

Wallaroo needs to spread information about the flow control problem to
other parts of itself.  Let's look at that problem in the next section.

### Wallaroo is a distributed system of actors in a single OS process

Wallaroo is an application written in the Pony language.
For more on this topic, check out our article:
[“Why We Used Pony to Write Wallaroo.”](https://blog.wallaroolabs.com/2017/10/why-we-used-pony-to-write-wallaroo/)

Pony implements the Actor Model of concurrency, which means that a
Pony program is a collection of independent actor objects that perform
computations upon their own private state.  Actors communicate with
other actors only using message passing.

<a name="figure26"></a>
![Wallaroo word count example diagram](/images/post/back-pressure/wallaroo1-partial.png)
Figure 26: Partial view of a Wallaroo application, with actors

In our example, there is an actor called `TCPSink` that is
responsible for writing data to a TCP socket that is connected a
Wallaroo data sink.  When that actor experiences an `EWOULDBLOCK`
event when sending data, it is the only actor that is aware of the
sink's speed problem.  Pony does not allow global variables.  We
cannot simply use a global variable like we can (very naively!) in C,
(e.g., `hey_everyone_slow_down = 1`), to cause the rest of Wallaroo to
slow down.

In an Actor Model system, actors communicate with each other by
sending messages.  Therefore, to tell the other actors about the TCP
connection slowdown,
the `TCPSink` needs to send messages to other Wallaroo actors.  If all
actors in the system are told to stop, then Wallaroo as a whole can
act as if a big finger reached into the system and pressed a "PAUSE"
button.

When that metaphorical "PAUSE" button is pressed, the system no longer
creates data to send to the sink.  Wallaroo's internal queues become
frozen as producing actors stop sending messages.  When paused, the
message mailbox queues for each actor are effectively limited in
size.  As long as
the pause happens quickly enough, we shouldn't have to worry about
uncontrollable memory use by the application as a whole.

### Wallaroo today: The mute protocol

Inside of Wallaroo today, a custom protocol is used to control the
back-pressure of stopping and starting computation in the data stream
pipeline.  We don't really want to broadcast to all
actors: many Wallaroo actors don't need to know about back-pressure or
flow control.  But we do need a scheme to help determine what
actors really do need to participate in the protocol.

The protocol is informally called the mute protocol, based on the
name of one of the messages it sends, called `mute`.  The word "mute"
means to be silent or to cause a speaker to become silent.  That's
what we want for back-pressure: to cause sending actors upstream in
a Wallaroo pipeline to stop sending messages to the `TCPSink`.

As a data stream processor, the computation stages inside of Wallaroo
are arranged in a stream or pipeline.  The diagram below shows a
simplified view of a
["word count" application in Wallaroo](https://github.com/WallarooLabs/wallaroo/tree/master/examples/python/word_count),
showing the actors that are directly involved with processing the
data stream.

<a name="figure27"></a>
![Wallaroo word count example diagram](/images/post/back-pressure/wallaroo1.png)
Figure 27: View of a Wallaroo application "word count"

Now, let's see what happens when a data sink stops consuming data as quickly as
Wallaroo produces it.

When the `TCPSink` actor becomes aware of back-pressure on the TCP
socket (via the `EWOULDBLOCK` error status when trying to send data),
then `TCPSink` sends a `mute` message up the chain to its predecessor.
That predecessor sends a `mute` message to its predecessor, and so on,
all the way backward to the `TCPSource` actor.  (See
[Figure 28](#figure28) below.)

When a `mute` message reaches the `TCPSource` actor, the `TCPSource`
actor stops reading from its socket.
The same TCP flow control
condition that told us that the *data sink* is slow will now
eventually force the *data source* to stop sending.
If the data source sends enough data, then the TCP advertise window will
drop all the way down to zero.  In reaction, the source is forced to stop.

<a name="figure28"></a>
![Wallaroo word count example diagram plus mute and unmute messages](/images/post/back-pressure/wallaroo2.png)
Figure 28: View of a Wallaroo application "word count" with `mute` and
`unmute` message sending.

When the sink TCP socket becomes writable again, then the `TCPSink`
actor sends an `unmute` message to its predecessor.  The `unmute`
message is forwarded step by step back up the chain.  Eventually, the
`unmute` message reaches the `TCPSource` actor.  The `TCPSource` actor
resumes reading from the source TCP socket, which triggers TCP to
send a non-zero advertise window to the data source.  The pipeline starts
flowing again!

Now we have all three pieces of back-pressure in place.  Starting from
the end of the pipeline and working backward, we have:

1. From a slow sink TCP socket to Wallaroo's `TCPSink` actor, by reducing TCP's
   advertise window to zero.  (Recall, the zero window size is signaled
   to `TCPSink` by the `EWOULDBLOCK` condition.)
2. Backward along the stream processing chain from `TCPSink` to
   `TCPSource`, using the custom `mute` protocol within Wallaroo.
3. From Wallaroo's `TCPSource` actor to the source's TCP socket, also
   by reducing TCP's advertise window to zero.  (Recall, the `TCPSource`
   actor stops reading from the source socket when that actor is muted.)

The `mute` protocol between Wallaroo actors is
software, and software tends to have bugs.  It would
be good to replace the artisanal, hand-crafted `mute` protocol
inside Wallaroo
and rely instead on a back-pressure system that applies to all Pony programs,
including Wallaroo.  That general back-pressure system is described
next.

### Wallaroo's future: Plans to use Pony's built-in back-pressure scheduler

A comprehensive back-pressure system was added to Pony in November
2017 by
[Sean T. Allen, VP of Engineering at Wallaroo Labs](https://www.wallaroolabs.com/about).
Look for a Wallaroo Labs article by Sean in the near future that
describes this the back-pressure scheduler in more detail.

Sean's back-pressure implementation operates at the actor scheduling layer
within the Pony runtime.  Pony's actor scheduler maintains internal
state of actors that are paused due to back-pressure.  The scheduler
propagates pause and resume signals in a manner
similar to the `mute`/`unmute` messages shown in [Figure 28](#figure28).
Because the back-pressure system is inside the Pony actor scheduler,
a Pony developer needs to add little or no additional code to get
comprehensive back-pressure scheduling behavior.  In fact, the current
0.4.1 release of Wallaroo uses the back-pressure scheduler today.

To create full end-to-end back-pressure,
Wallaroo needs some
source code change.  The changes fall into two categories:

1. Removal of most of the existing back-pressure mechanism, with its
   `mute` and `unmute` protocol messages.
2. Add small pieces of glue code to allow TCP sockets that experience
   `EWOULDBLOCK` errors when writing data to signal to the runtime
   that the socket's owner actor should initiate back-pressure
   propagation by the scheduler.

## Other Wallaroo actors that use back-pressure

There are two other areas where Wallaroo has actors that
participate in back-pressure propagation.

* Wallaroo has source and sink actors for communicating with Kafka
  servers that act as Wallaroo data sources and sinks.
  Kafka's own protocol imposes
  additional constraints on flow control, but the general TCP
  principles described here also apply to the Kafka-related actors.

* Wallaroo uses TCP to copy data between nodes in a
  multi-worker Wallaroo system.
  Figure 29 has a simplified view of the actors involved in a pipeline
  that is split across two Wallaroo worker processes.
  The class names of the actors used for that
  internal communication are `OutgoingBoundary` and `DataChannel`,
  but the back-pressure
  principles are the same as for the `TCPSource` and `TCPSink` actors.
  The `OutgoingBoundary` and `DataChannel` actors also implement the
  `mute` protocol (today) and are
  subject to the back-pressure mechanism in the scheduler (future).

![Multi-worker communication example](/images/post/back-pressure/wallaroo3.png)
Figure 29: Actors and socket involved in multi-worker communication

## Testing Wallaroo and back-pressure

I have already made many of the above code changes
[on a development branch](https://github.com/WallarooLabs/wallaroo/compare/gh1740),
but are not yet tested systematically.
If you are a frequent reader of the Wallaroo blog,
then you know that we devote a lot of time and effort to correctness
testing.

Testing-wise, I've first done a lot of preparation work to permit Wallaroo
to control the kernel's TCP buffer sizes.
(It's much easier to create reliable, repeatable tests
for the back-pressure system if you can predict the TCP
sockets' buffer sizes in advance!)
The new tests apply to both the current back-pressure `mute`
protocol and to the new Pony runtime's back-pressure scheduler
implementation.

We aren't completely sure that the Pony runtime's
back-pressure scheduler will actually meet Wallaroo's needs.
Here's a partial list of open questions:

* We hope that the runtime's back-pressure scheduler will sharply
  limit the amount of memory used by Wallaroo when back-pressure is
  active.  But we haven't measured Wallaroo's actual memory behavior
  yet.
* We suspect that some additional parameters will be
  necessary to tune the scheduler's back-pressure behavior.
* The back-pressure scheduler may yet be buggy.  Its code is not yet half
  a year old.
* How will the runtime react when back-pressure scheduling has taken
  effect, and then a system administrator wants to query the cluster's
  state?  Or change the cluster's size, larger or smaller?
* Do either of the back-pressure implementations stop too much
  processing?  Zach Tellman points out in his talk
  ([linked below](#refs)) that buffer space can be traded for latency
  and throughput improvements.  Perhaps Wallaroo pipelines may stall
  for too long, with some processing steps idle while waiting for upstream steps
  to resume sending after back-pressure has been released?

These open questions drive my interest in test infrastructure: I
want to find as many bugs in Wallaroo's workload/overload management
as possible before our customers do.  It's a fun task!  And when we find
performance changes and/or interesting bugs in this work, we'll write
about it here.  Stay tuned.

## Series conclusion: Wallaroo uses back-pressure to create end-to-end back-pressure during overload conditions

[The first part of this short series](https://blog.wallaroolabs.com/2018/03/some-common-mitigation-techniques-for-overload-in-queueing-networks/)
on overload mitigation first
gave an informal definition for the term "overload" in the context of
queueing networks.  It then presented several techniques that can
mitigate the effect of overload conditions.  One of those techniques,
back-pressure, nicely fits the goals and constraints of Wallaroo.

The second article (this one) describes three back-pressure mechanisms
used by Wallaroo: the TCP protocol's sliding window protocol,
Wallaroo's `mute` protocol, and an actor scheduler-based back-pressure
implementation.  Wallaroo integrates these techniques into an
end-to-end back-pressure system that can mitigate overload in a
Wallaroo cluster of any size.  As we strive to make Wallaroo a robust
and reliable data stream processing platform, we have more work ahead
to fully integrate and test the back-pressure system.

<a name="refs"></a>
## Pointers additional online resources and to other back-pressure systems

* dataArtisans: [How Apache Flink™ handles backpressure](https://data-artisans.com/blog/how-flink-handles-backpressure)
* Fred Hebert: [Handling Overload](https://ferd.ca/handling-overload.html)
* Henn Idan: [Reactive Streams and the Weird Case of Back Pressure](https://blog.takipi.com/reactive-streams-and-the-weird-case-of-back-pressure/)
* Reactive Streams initiative: [Introduction to JDK9 java.util.concurrent.Flow](http://www.reactive-streams.org)
* Zach Tellman: [Everything Will Flow](https://www.youtube.com/watch?time_continue=1&v=1bNOO3xxMc0),
  an overview of Clojure's `core.async` library.
* Wikipedia:
[Back-Pressure](https://en.wikipedia.org/wiki/Back_pressure),
[TCP Flow Control](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Flow_control),
and
[Sliding window protocol](https://en.wikipedia.org/wiki/Sliding_window_protocol)
topics

Figure 25.7 is excerpted from ["Fundamentals Of Computer Networking And Internetworking" class notes, chapter 25](https://www.cs.csustan.edu/~john/classes/previous_semesters/CS3000_Communication_Networks/2015_02_Spring/Notes/chap25.html)
by Douglas Comer (Purdue University) and Pearson Education.
I used #25 as the basis for
all of the following figures.  I hope my unorthodox
numbering scheme didn't lead you on a goosechase to try to find Figure 3.
