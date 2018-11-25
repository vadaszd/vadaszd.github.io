# Open for event based tracing?

In OpenTracing the fundamental concept for representing distributed traces is the (time) span: something that starts and then finishes, can be annotated with key-value pairs and can be "causally" related. This representation gained popularity with Google's Dapper paper and triggered open-source tracing implementations like Zipkin and Jaeger and eventually the OpenTracing specification, but according to the academic literature it is not the only one.

In spite its popularity, the span based trace representation has some shortcomings that limit its applicability. These reside in the lack of precise definition of the meaning of spans and their relations and the representation being tightly coupled to the [synchronous RPC](https://en.wikipedia.org/wiki/Remote_procedure_call) programing model.

In this writeup I would like to show how OpenTracing could overcome these shortcomings by defining an alternate trace model centered on the notion of events. The new trace model would allow for preserving the compatibility of existing instrumentations while opening up the way for extension to other, non-RPC programing models.

The event based trace model has its origins in [one of Leslie Lamport's writings](https://amturing.acm.org/p558-lamport.pdf) on the ordering of events in a distributed computer system. I also acknowledge that most of the ideas presented here were published by Facebook in a [research paper](https://research.fb.com/publications/canopy-end-to-end-performance-tracing-at-scale/) in 2017.



## Events bring order

Why tracing? Because we want to understand how our distributed algorithms are executed: which components contribute to them, how long they last and why. I further argue that if we want to understand the why, we get further by focusing on what orders things as opposed to how long they last.

*If spans are about durations, then events are about ordering.*

Events are a natural way of indicating progress in a sequential program. In contrast to spans they are infinitesimal "points" in the execution, something we can be before or after but not in. It is tempting to write they are "points in time", but (as Lamport points out) in a distributed system there is no notion of a global clock: time is local to the executors. By the locality of time we refer to the fact that the clocks of the executors cannot be synchronized accurately enough to reliably order the events by their timestamps if they happen on different executors. Still, we can interpret a [partial order](https://en.wikipedia.org/wiki/Partially_ordered_set) among events. This is because the executors are programmed to coordinate their local progresses as they cooperate in performing the distributed computation; they arrange for adhering to a partial order according to a virtual shared clock. If we manage to capture who has to wait for what, we get a much better understanding of why some traces last long.

With events as the foundation for distributed tracing, a trace becomes a directed acyclic graph (DAG) of events, where each edge represents an ordering constraint the executors were programmed to obey. The notion of causality gets an abstract but precise meaning: `A` causing `B` means the executors made sure that `A` happened before `B` according to some virtual shared clock.

Contrast the above with the span model. Formally the latter leaves the interpretation of causality open; in practice it invites for perceiving causality as (de)composition. If a piece of work `X` is broken down into steps `Y` and `Z`, then span `X` is linked as the cause of the other two. The lack of a precise definition of what spans are limits the set of applicable analysis tools to those that can cope with the current vague semantics. *A less ambiguous trace model would unleash the evolution of trace analysis tools.*

## Events bring meaning

A corollary of representing traces as a DAG of partially ordered events is some guidance on when to create events: they have to be logged where the executors coordinate their progress in the computation. To record the ordering constraint, we log events:

1.  When a sequential processing unit `1` suspends the execution (waiting for an external precondition necessary to proceed)
1.  When another processing unit `2` reaches the state required to fulfill the condition
1.  When unit `1` learns about this and actually resumes execution

If we arrange that unit `2` propagates a reference to the logged event alongside the information about the fulfillment of the condition, then `1` can attach that to the 3rd event, weaving the DAG.

This synchronization event-pattern captures the ordering constraint independently of the coordination mechanism used. Be that message passing, locking or queuing, the common semantics allow for end-to-end, consistent reasoning about the trace, such as computing the critical path.  

So what is the meaning of an event? In an abstract manner, an event signals that
*   a local computation has reached a stage where some external condition or information is needed or
*   it produced a condition necessary for progressing with another local computation or
*   it learns about such a condition and resumes the computation

A more concrete meaning can be attributed to events if we know what mechanism is used for coordinating progress.

Spans represent time durations, but the span model gives no specific advice on what durations to measure. Mentally, it has its origin in the (synchronous remote) procedure call programming model, in which spans nicely map to function calls: a function or code block is entered (span starts) and then left (span finished), in the meanwhile we record on the span what happened. As soon as the programming model is a bit different, there is no trivial way to map it to spans.

What if the remote procedure call is asynchronous? Shall we finish the span when the request is sent and open a new one for the reply? What if, in the meanwhile, the execution context is transferred to another processing unit and the reply is processed there?

What if the programing model is messaging? Should the span start when the message is produced or when it is dequeued? Should one or two spans represent the sending and receiving of a message? How about one-to-many communication like publish and subscribe?

## Formalizing the terminology
Let's start using a bit more consistent vocabulary (borrowing it mainly from the [Facebook paper](https://research.fb.com/publications/canopy-end-to-end-performance-tracing-at-scale/) mentioned earlier.

An **execution unit** (EU) is an abstraction for something (e.g. a thread, process, coroutine) executing a sequential algorithm. An EU is identified by an *EU ID* (which can be e.g. a 64-bit random number). Its life is bounded by the start and completion of the local, sequential algorithm it executes. An EU is in one of the `running` or `waiting` states.

Note that the system designer may keep some threads, processes or coroutines (in short: sequential executors) transparent for tracing. There is no need to create an EU for every sequential executor participating in the flow. All we require is that the non-represented entities forward the tracing context without altering it and refrain from logging tracing events.

*Sharing an EU among concurrent executors is not allowed.*
Not even if access to the EU is serialized. The reason for this is that the sequence of events generated on an EU is intended to capture the ordering constraints inherent in the sequential algorithm running on the EU. If shared among concurrent executors, then the sequence will capture an ordering of the events, but these will not be actual constraints leading to that order: the order will be accidental. Among events in concurrent executors, there are no ordering constraints.

Creating multiple EUs during the life of a sequential executor is a poor but allowed practice. There will be no ordering information captured among events logged in different EUs.

An **event** is an indication of progress in the algorithm executed on an EU. To express the ordering of the events the EU numbers them sequentially. It also assigns an *event ID* to each event. We require is that the eventID is unique within the trace and within the EU but global uniqueness is not a must. For example, the event ID could be implemented as the (EU ID, *event sequence number*) tuple, or to save space, a hash computed from these.

Events can be annotated with key-value pairs.

In order to attach information to the events about the `running`/`waiting` state transitions and the role in which they participate in inter-EU dependency relationships, we introduce the concept of **event types**.

A **trace** is the set of events generated during a computation distributed over one or more EUs. A trace can be identified by a *trace ID*. (Due to cardinality a 128-bit ID is recommended, and when correlating events of a large number of potentially long-lasting traces, it is a huge advantage if the approximate time stamp of the trace can be extracted from the trace ID, so a version 1 UUID is a prime implementation candidate.)

During the distributed computation the EUs cooperate by exchanging information and coordinating their progress, e.g. by means of messages, locks, signals, etc. How this coordination happens is irrelevant at this stage of the discussion. All we require is that the mechanisms used allow for the propagation of a **tracing context**, whenever the progress of the algorithm on an EU is subject to progress on another EU. The tracing context includes a *(trace ID, event ID)* tuple. The propagation must happen from the EU being depended on to the dependent EU. The event ID in the context must be that of the event indicating the progress being depended on.

Placing the trace ID into the context allows for linking the local progress in the individual EUs to the progress of the distributed computation. Propagating the event ID allows for expressing ordering constraints between events in different EUs.

Events should be created at least when the guidelines described earlier require to do so. Additional events may be created at the discretion of the programmer's interpretation of progress in the computation.

The events logged in a distributed computer system constitute a DAG capturing the ordering constraints affecting an execution of the overall system. Edges between local (same-EU) events are represented implicitly, by neighboring event sequence numbers. Inter-EU dependencies are encoded as explicit references to event IDs propagated in the tracing contexts.

*An individual trace is nothing more than a subgraph of the DAG describing the overall execution of the system.*

An important advantage of the DAG-of-events trace model is that it allows for the programatic identification of the critical paths of the traces. This is invaluable for response time analysis by statistical means. The DAG-of-spans model is not suitable for this purpose, as in that model the edges of the graph do not represent ordering constraints.

Beyond understanding the execution of end-user requests, event-based tracing also let's us argue about how concurrent activities interact via shared resources.

## Convergence paths
OpenTracing has been around for some time, and it is based on the span model. Is there a way we could improve it so that it offers the advantages of the event based trace model?  

#### Easy and wrong: representing events as spans
One could argue that both models represent traces as a DAG, so an existing OpenTracing implementation storing traces as a DAG of spans can be easily used for storing event-based traces: just store events as zero-duration spans. However, this is a trap:

*   Existing API users log non-zero-duration spans, so they would not be able to benefit from the analysis services  using the event-based trace model.  
*   Existing consumers of span based trace data (analysis tools written for the span model) may expect the duration to be non-zero, so they may be unusable with clients trying to use the API following the event-based model (logging zero-duration spans) .

*The central part of OpenTracing is the API it defines, so when seeking for a convergence path, maintaining compatibility is a top requirement.*

In other words, asking if the OpenTracing API can be used to capture event-based traces is wrong. We should ask:

> Is it possible to keep the existing semantics of the OpenTracing API when implementing it on the event-based trace model?

#### Mapping spans to events

Conceptually we can represent a span as a pair of "start" and "stop" events. In event-based tracing an event is associated with an execution unit, of which the specification of the OpenTracing meta-API has no notion. Fortunately an event-based implementation of the API could automatically identify the thread or coroutine the API verbs are called on and implicitly create EU instances for the events it generates.

The tags of a span could be attached to its start event; logs would be modeled as separate events.

Span references are a bit more problematic, because event-based tracing does not know about spans. What could be offered as a replacement is a reference to (or ID of) one of the previous events on the spans, such as:

1.  The starting event of the span.
1.  The most recent local event on the span (i.e. on the same EU as the API call to retrieve the context is being made).
1.  An event newly created when the span context is retrieved.

All three are correct from the event-based point of view, as they will represent valid ordering constraints. The second and especially third options are preferable in the event-based sense as they provide tighter constraints. The first option has the advantage that the retrieval of the reference is idempotent, i.e. the reference can be used for identity comparison.

The examples listed in the [OpenTracing specification](https://github.com/opentracing/specification/blob/master/specification.md) also suggest a secondary, poorly-articulated meaning for the `childOf` reference type: they constitute a (de)composition mechanism for traces. Span-based visualization tools (like the Jaeger UI) make extensive use of this implicit meaning when they organize the spans into a tree and expand and collapse it branches.

Composition and causality are both useful constructs although fundamentally different from each other. The former is a means for controlling the level of detail in the trace, the latter tracks information flow and ordering. Both are useful! So if we want event-based tracing to support decomposition (i.e. drill down in a trace to increasing level of detail), we need to find a new concept; something that is the  generalization of spans in a DAG of events.

#### Supporting composition

To come up with their generalization, first we need to understand what really spans are. The [specification](https://github.com/opentracing/specification/blob/master/specification.md) says little more than they can be started and finished. When we also consider the API, it turns out that spans can be shared among threads but there is no standardized way to distribute a span over processes: a span context can be used to create a child span but not for re-creating and continuing the one from which it was retrieved. From this we can infer a span is a process-local sub-activity in a trace.

*In an event-based model we can think of a span as a local subgraph of the DAG representing the trace.*

How such a subgraph can be delineated from the rest of the trace is not relevant from the point of view of the current discussion. We note however, that the implementation of such a delination mechanism would require annotating the events with a span ID or a carefully planned rule-set for attaching span-demarcation info to the events, as the OpenTracing API does not restrict how spans are shared among the threads or coroutines represented by the EUs.

How can we generalize spans? We remove "local" in the above definition and instead of spans, start speaking about patterns.

*__Patterns__ are small predefined DAGs that can be [bijected](https://en.wikipedia.org/wiki/Bijection) to subgraphs of traces.*

If the patterns are constructed hierarchically (so that the subgraphs they map to are either disjunct or subgraphs of each other, i.e. for any pair of the subgraphs A and B matching any of the patterns, either the [intersection](https://en.wikipedia.org/wiki/Intersection_(set_theory)) or one of the [relative complements](https://en.wikipedia.org/wiki/Complement_(set_theory)) of A and B is empty), then the patterns organize the trace into a number of embedded, hierarchically expandable subgraphs: excellent for drill-down visualization!

By the appropriate definition of event types and their use as subgraph-demarcators, the trace generation and collection infrastructure can be kept agnostic towards patterns. Patterns in the traces could even be efficiently identified during a post-processing step: there is no need to propagate or log e.g. pattern IDs.

## Life beyond spans

One of our selling point for event based tracing is that it allows for several tracing APIs. Thus a developer may choose the one that matches the way he thinks about the code, i.e. it is in line with the *programming model*.

Defining these APIs would be a way too ambitious endavour for a blog. Still, I would like to present some considerations on what kind of programming models would warrant a dedicated API?  

#### What exactly is a programing model?

In general, a *programming model is an abstraction of the underlying computing machinery that allows for the expression of both algorithms and data structures<sup>[1]</sup>*.

   [1]: https://asc.llnl.gov/content/assets/docs/exascale-pmWG.pdf

In particular, when dealing with a distributed computer system, we have to focus on parallel programing models. According to the taxanomy in the [Wikipedia article](https://en.wikipedia.org/wiki/Parallel_programming_model) on the topic:

> Parallel programming models can be divided broadly into two areas: process interaction and problem decomposition

The former relates to how processes interact (i.e coordinate their progresss), the later describes how the program is decomposed into processes. What matters for tracing is the first one.

Along the lines of the article *"the most common forms of interaction are shared memory and message passing, but interaction can also be implicit (invisible to the programmer)"*.

If an interaction is invisible for the programmer, then we cannot reasonably expect her to instrument that code with a tracing API. The other two cases are worth a closer look:

*   In a **shared-memory programming model** asynchronous concurrent access is coordinated through mechanisms such as locks, semaphores and monitors. The tracing API should allow for capturing the interaction with these (e.g. the requesting, acquiring and releasing of locks, signaling, sleeping and waking up in semaphores and monitors). Furthermore, these synchronization constructs would need to be extended with mechanisms to propagate the tracing context.

*   Message passing is in fact a family of programming models:
    -   In **synchronous message passing** the sender has to wait until the receiver is ready. The primary use case for this model is the design and verification of concurrent systems. Implementing tracing for this programming model would require further justification.

    -   **Asynchronous message passing** implementations buffer messages. This programming model is used with [event loops](https://en.wikipedia.org/wiki/Event_loop), point-to-point communication on [message oriented middleware](https://en.wikipedia.org/wiki/Message-oriented_middleware) and the [actor model](https://en.wikipedia.org/wiki/Actor_model). The ubiquity of these use cases warrants a tracing API for the asynchronous message passing programming model. This API should support capturing interactions following any of the asynchronous *point-to-point*, *publish-and-subscribe* and *request-reply* messaging paradigms.

    -   **Remote procedure calls** use the request-reply messaging paradigm. They differ from synchronous message passing in the client waiting for the reply to arrive, not for the server to be ready. This model is covered by the existing OpenTracing AI.

## Conclusion
By now, I hopefully convinced you that event based tracing makes up for some of the shortcomings in the span based trace model.

I hope you agree that the event model could be introduced underneath the OpenTracing API, transparently to already instrumented software.

Perhaps I have even managed to trick you into wondering how the OpenTracing API could be extended at least to the asynchronous messaging programming model.
