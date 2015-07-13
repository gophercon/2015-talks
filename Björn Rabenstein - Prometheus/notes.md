# Speaker notes

In a talk, the audience should listen to the speaker. Repeating the
words of the speaker on the slides (or even summarize them) is a
distraction. I try not to do that on my slides, which makes them
fairly useless as a summary of the talk. This file contains (rough)
speaker notes to give you an impression what I planned to tell the
audience.

## Slide 1

A good part of my time at SC, I work on an open source monitoring
project called Prometheus.  Project started in 2012, but only made
noise about it in January, rapidly got a very active community.  Hands
up, who has at least a vague understanding of what Prometheus is?  For
those that do not: Unfortunately, I cannot tell you a lot about
monitoring with Prometheus.  This is a Go conference, not a monitoring
conference.  I’ll talk about Go aspects of design and implementation.
But do not despair!

## Slide 2

Awesome website with all the explanations.  By now, a number of
introductory talks have been recorded.  Linked from the website.

## Slide 3

Still, let’s have a short look at the architecture.  Many of the boxes
above are implemented in Go.  The idea here is to pick a couple of
boxes above and explain the related Go gotchas, lessons learned,
design decisions...  Why we picked Go for that component.  And never
ever regretted our decision, ... almost never ever...

The starting point should be the client library, which you use to
instrument your code for Prometheus.

## Slide 4

Implementing a monitoring client library can’t be that difficult.
There are a number of metric types in the Prometheus universe.
The most fundamental is a counter. It counts things, and only ever goes up.
Let’s focus on that one for additional simplicity.

## Slide 5

Doc comments elided.

Explain embedded metric, add arg always positive, Inc == Add(1).

Inc is just Add(1), so we’ll leave it out in the following for the
sake of brevity.

## Slide 6

Naive counter implementation.

Incrementing a number, the easiest thing for a computer.

Done! Great! Next box...

Or not...
Not goroutine safe at all.

So we could just write our program in a way that any given counter is
only updated by one goroutine.

Perhaps...

But then there is the Write call to write out a metric whenever the
Prometheus server asks for it.  Sounds like quite some burden for the
user of the exposition library.

## Slide 7

Let’s use old school mutexes.

Note pointer receiver in Write to not copy in a goroutine unsafe way
(and to not copy mtx).

Perfect, done! Next box...

## Slide 8

But... performance...

It’s a client library, finally you can indulge in micro optimization,
without any remorse... Premature optimization? Certainly somebody out
there will use your library in a way that totally justifies every
single tweak...

Go tooling makes it easy to write microbenchmarks.

Simulate typical prometheus situation, i.e. many increments, few
scrapes, i.e. read (Write() call).

## Slide 9

Acquiring a mutex takes quite some time, actually.

But what’s a nanosecond, anyway? That’s a damn short amount of time, right?

## Slide 10

But what again was the reason to use mutexes in the first place?

Right, concurrent access.

So we should have it in our benchmark, too.

We can also run it with the race detector to test for races (but do
not run with race detection for the actual results because it slows
things down)

The code runs multiple goroutines, and with the waitgroups, it makes
sure to unleash them all at once.

Command line flag -cpu to set GOMAXPROCS.

_Post-talk addendum:_ The testing.B.RunParallel can be used to implement
the above without all the boilerplate code._

## Slide 11


Good example for GOMAXPROCS>1 making things worse for inherently
sequential code.

Go1.5 will probably improve things here.

## Slide 12

But does it matter at all?

So what is that?

That’s a screenshot of the Prometheus expression browser.

But what do we see here?

If you look at the expression above (fine print :), you might guess it:

This is Prometheus monitoring itself, and this server is ingesting
350k samples per second.  While this is a nice result, let’s not think
about the engineering work behind it right now.  Let’s think about our
mutex counter.  Prometheus certainly runs on many CPUs (32 in this
case), and it runs many goroutines to ingest that many samples.  If a
counter increment takes ~1000ns, we need 350ms to increment it 350,000
times.  To count the ingested samples of a single second, we need
350ms.

Monitoring is a bit like measurements in quantum physics, you’ll
inevitably change the target by monitoring it.  But what we have here
is pretty much inacceptable.

So yeah, let’s make it faster.

## Slide 13

Rob 12:3–4

It time to quote from the holy scripture.

So we want a single goroutine managing the counter, and increments and
read requests should be handled by channels.

## Slide 14

Inc and check for negative increments elided.

Constructor function not shown, which needs to make the chans and
starts loop() in a goroutine.

out is a synchronous channel, while we can play with the buffering of
the in channel.

## Slide 15

While the code looks much more go-idiomatic, the result is
frustratingly bad.  Times have at least doubled.

Academic interest in the timing patterns.  Things are more similar
with high contention.  Between buffered and sync channel, but also
compared to mutex.

After all, results are not that surprising. Channels are not meant to
be faster mutex replacement, despite all the good work that has gone
into them to make them faster.  Channels are meant to enable a cleaner
architecture, not to replace mutexes for a trivial low-level task.

As always, holy scriptures have to be applied with caution.

Our initial problem is still unsolved... If we only could atomically
increment an integer.

## Slide 16

Enter sync.atomic.

## Slide 17

Finally...

Numbers are down by an order of magnitude, 25x in the contentious
case...

Done! Hooray! Next box...

## Slide 18

Except... I lied.

Prometheus does not use int's but float's for sample values.  That's a
fundamental design decision discussed elsewhere. (Hint: prometheus.io)
Why would we ever want to increment a counter by fractions of an
integer?  E.g. we want to accumulate latency in seconds. Certainly we
will add something like 0.23s or something.

So now what?  floats cannot be atomically incremented.  We could at
least optimize in cases where only integers are added (even if they
are passed in as a float.)

But we can do better than that...

## Slide 19

Explain in that order:
* valueBits
* Write
* Then Add
* Inc elided as usual.

But will it work? Won’t there be a lot of retries under high contention?

## Slide 20

Yes, it works...

Only half as good as the int counter, but not too bad at all.

Finally, finally done... finally the next box...

## Slide 21

This is a real thing.

As said, it’s a library. People will use it in all kind of situations.
We didn’t pay attention, and neither are some of you, I’m sure.  Just
search github or something.

We ran into it when we accidentally compiled 32bit binaries for Linux.

But people do weirder things than that...

## Slide 22

Oh yes, people even run the whole server on their Raspberry and blog about it.

That was be done by Fish, one of our early contributors,
pre-announcement, who is now at Docker.

## Slide 23

Finally, let’s pick another box in the architecture graph.

Oops.

## Slide 24

So this should have been the honest title of the talk.  Simple things
are sometimes hard...

I'll be around, so catch me if you want to talk about the other
boxes. We should have time on Hack Day.

But we do have a few minutes left,

So please allow me to jump to conclusions along the performance theme,
without looking at particular boxes in the architecture diagram, and
without telling the wholy story around them.

Three top lessons learnt about performance optimization.

## Slide 25

We already know: The first advice should be: Run benchmarks copiously.

But we didn’t talk about the -benchmem flag, because in the very
simple cases discussed so far, allocations were not an issue.
benchmem will show you the number of heap allocations required per op.

Allocation churn used to be a big problem in Prometheus, and it’s even
more of a concern in the client library.  Imagine incrementing a
counter would require a heap allocation. 300,000 allocations per
second, just to count...

Counting without allocations is easy.  How we managed to avoid
allocations in the more complex parts of the library would be a topic
for another talk.

As a troubleshooting hint: If you can’t find out where some mysterious
allocation is happening, it’s often because something “escapes” from
the local scope and has therefore be heap-allocated. look at the
escape analysis the compiler performs to find out.

## Slide 26

pprof is so good!

Just add that one import line.

It’s not a big penalty, so you can leave it “always on”.  Whenever you
see a problem even in a running production system, you can use that
endpoint for forensics.

Helped us countless times to find the weirdest of bugs and optimize
the system to reach the performance we are at.  Just looking at the
goroutine dumps.  Or using the pprof tool to create those beautiful
call graphs.

Plus self monitoring of Prometheus. :)

## Slide 27

Call graph.

## Slide 28

Highly optimized C libraries are great.  Often, go implementation of
the same thing are much slower (for now).  We use levelDB heavily in
Prometheus, and in earlier version we even used it for the heavy
lifting of storing sample data.

LevelDB is an example where a very mature C implementation exist and a
pure Go implementation.  The C library is much faster.  Using it via
cgo sounds like a good idea, but there are obvious issues: Compiling
from sources now requires a C build environment with all its problems.
Statically linking doesn’t necessarily works. Back to shared libraries
with all the problems we left behind with the typical Go static
binary.  But your performance may actually drop.

Best case is very long running individual function calls with very
little data exchange.  Prometheus is kind of the opposite, remember
the 350k samples ingested per second, and then the whole sample data
to ingest is part of the payload.

After we learned it the hard way, others blogged about it, recommended
reading.

Prometheus case: Eventually, we wrote our own special purpose storage
layer for the bulk sample data and use LevelDB only for indices.  The
Go implementation of LevelDB is good enough for that.

## Slide 29

So let’s wrap up.

Special thanks to out two founding fathers, who started Prometheus as
a pet project before it was adopted by SC as their standard monitoring
system, Matt T. Proud and Julius Volz, who are both in the audience.

I’ll be around at the conference, you can ask me about all the other
boxes we didn’t cover in the talk. We also have 5 minutes left to take
questions.  But I’ll redirect the hard ones to Matt and Julius anyway.

Thank you.
