## Onyx Windowing

### Research

We're actively looking over the following papers for ideas on how to best design windowing. We summarize the papers to some extent, and note which parts we want to use and what the drawbacks are.

- [The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing](vldb.org/pvldb/vol8/p1792-Akidau.pdf)
- [Semantics and Evaluation Techniques for Window Aggregates in Data Streams](http://web.cecs.pdx.edu/~tufte/papers/WindowAgg.pdf)
- [Exploiting Predicate-window Semantics over Data Streams](http://docs.lib.purdue.edu/cgi/viewcontent.cgi?article=2621&context=cstech)
- [How Soccer Players Would do Stream Joins](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.362.2471&rep=rep1&type=pdf)
- [No Pane, No Gain: Efficient Evaluation of Sliding-Window Aggregates over Data Streams](https://cs.brown.edu/courses/cs227/papers/opt-slidingwindowagg.pdf)

#### Semantics and Evaluation Techniques for Window Aggregates in Data Streams

This paper outlines a technique that efficiently bucket data into possibly overlapping windows. The approach, named "Window ID", aids in reducing memory usage and execution latency. The DataFlow paper refers back to this paper, noting that it uses a similiar "bucketing operator" for queries.

More directly from the paper:

> We see two major issues with current stream query systems that process
> window queries. One is the lack of explicit window semantics. As a result,
> the exact content of each window extent tends to be confused with window
> operator implementation and physical stream properties. The other is
> implementation efficiency, in particular memory usage and execution time.
> To evaluate sliding window aggregate queries where consecutive window extents
> overlap (i.e., each tuple belongs to multiple window extents), most current
> proposals for window queries keep all active input tuples in an in-memory buffer.
> In addition, each tuple is reprocessed multiple times—once for each window extent
> to which it belongs. We will propose an approach that avoids intra-operator
> buffering and tuple re-processing.

WID is powerful because it can handle disorder for events defined on any totally ordered domain. The algorithms change depending on the generality of range and slide attributes. For our initial purposes, the range and slide values are presumed to be the same. This has been implemented and explained in the `window_id.clj` file, and generative tests have been written to confirm its implementation semantics.

The authors note on extensibility:

> Our window specification is quite expressive, and the semantic framework suggests
> a general way to define window semantics. We have discussed existing types of
> windows that we have seen.

Another note on totally ordered domains:

> Potentially, WATTR can be any tuple attribute with a totally ordered domain.
> Having this option allows us to define windows over timestamps assigned by external
> data sources or internally by the system; to handle a stream with a schema containing
> multiple timestamp attributes; and to window over non-temporal tuple attributes.

Why it's important to allow windows to be defined by features of the data itself:

> SQL-99 defines a window clause for use on stored data. SQL-99 limits
> windows to sliding by each tuple (i.e., each tuple defines a window extent),
> thus tying each output tuple to an input tuple. We call such windows data-driven.
> In comparison, stream queries often use domain-driven window semantics where users
> specify how far the consecutive window extents are spaced from each other in
> terms of domain values [15]. We believe domain-driven windows are more suitable
> for applications with bursty or high- volume data. Consider a network monitoring
> application—one possibly wants network statistics updated at regular intervals,
> independent of surges or lulls in traffic.

#### No Pane, No Gain: Efficient Evaluation of Sliding-Window Aggregates over Data Streams

This paper is mainly about an optimization that can be applied to windowing that helps computation and memory costs. Panes split windows into pieces, and those pieces can be shared across other windows to avoid recomputation. Directly from the paper, we see a good problem statement:

> Current proposals for evaluating sliding-window aggregate
> queries buffer each input tuple until it is no longer needed
> [1]. Since each input tuple belongs to multiple windows,
> such approaches buffer a tuple until it is processed for the
> aggregate over the last window to which it belongs. Each
> input tuple is accessed multiple times, once for each window
> that it participates in.

The authors go on to say:

> We see two problems with such approaches. First the
> buffer size required is unbounded: At any time instant, all
> tuples contained in the current window are in the buffer,
> and so the size of the required buffers is determined by the
> window range and the data arrival rate. Second, processing
> each input tuple multiple times leads to a high computation
> cost. For example in Query 1, each input tuple is processed
> four times. As the ratio of RANGE over SLIDE increases,
> so does the number of times each tuple is processed. Considering
> the large volume and fast arrival rate of streaming
> data, reducing the amount of required buffer space (ideally
> to a constant bound) and computation time is an important issue.

##### Motivation

A good example of where panes come in handy is derived from Query 4 of the
paper:

> Over the past 10 minutes, find the ids of the
> auction items on which the number of bids is greater than
> or equal to 5% of the total number of bids; update the result
> every 1 minute.

This is a relatively complex query that requires a lot of intermediate state
to progressively update the answer every minute. This query's internal representation
would get unwieldly without some form of optimization.

##### Implementation

When we use panes, we take the original query specification and split
it into a Window Level Query (WLQ) and Pane Level Query (PLQ). The PLQ
does *sub-aggregation*. The WLQ does *super-aggregation*. The number of
panes, the range, and slide values of each query are determined programmatically.
They are not tunable configuration values because the number of panes is
used to determine windows can share panes.

##### Types of Queries

Not all queries with panes should behave the same way. Some queries can be
even further optimized. This paper breaks queries down into:

- holistic
- bounded
- fully differential
- pseudo differential
- none of the above

Each of these have properties that allows varying levels of computational shortcuts,
so we should figure out which we can handle most easily and plan for future
implementations.

##### Drawbacks

Panes can fail to be useful and degrade performance when there aren't
"enough" segments in each pane. We should provide a configuration
switch to not use panes. This configuration option can be a secondary
feature, though.

#### The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing

This is probably the most important paper that we researched as it tied all of the ideas that we're using together into a good API. Few of the ideas presented are actually new, but their composition is what makes it an excellent contribution. I won't cite much of the paper here as I dove into the citations for further clarification, but it can be summed up with:

> We propose that a fundamental shift of approach is necessary to deal with these
> evolved requirements in modern data processing. We as a field must stop
> trying to groom unbounded datasets into finite pools of information that
> eventually become complete, and instead live and breathe under the assumption
> that we will never know if or when we have seen all of our data, only that new
> data will arrive, old data may be retracted, and the only way to make this problem
> tractable is via principled abstractions that allow the practitioner the choice of
> appropriate tradeoffs along the axes of interest: correctness, latency, and cost.

This piece of the paper is the primary driver that encourages us to optimize our streaming engine to support all types of computation.

#### How Soccer Players Would do Stream Joins

This paper doesn't emphasize the mechanics of how to construct windows, but instead zooms in on one particular windowing problem - joins. The paper presents a technique for leveraging all the cores in a machine to efficiently perform a windowed join.

> In this work we present handshake join, a way of describing and
> executing window-based stream joins that is highly amenable to parallelized execution.
> Handshake join naturally leverages available hardware parallelism, which we
> demonstrate with an implementation on a modern multi-core system and on
> top of field-programmable gate arrays (FPGAs), an emerging technology that
> has shown distinctive advantages for high-throughput data processing.

Some of the paper talks about how this can be implemented in hardware, which isn't useful for us. However, the general technique can be used in any language.

The idea can be easily visualized, hence the name:

> The enumeration of join candidates may be difficult to distribute efficiently
> over a large number of processing cores. Traditional approaches (including CellJoin)
> assume a central coordinator that partitions and replicates data as needed over
> the available cores. But it is easy to see that this will quickly become a
> bottleneck as the numbers of cores increase. The aim of our work is to scale
> out to very high de- grees of parallelism, which renders any enumeration scheme
> unusable that depends on centralized coordination. It turns out that we can learn
> from soccer players here. Soccer players know very well how all pairs of players
> from two opposing teams can be enumerated without any external coordination.
> Before the beginning of every game, it is custom to shake hands with all players
> from the opposing team. Players do so by walking by each other in opposite directions
> and by shaking hands with every player that they encounter.

We should dive back into this paper for more detail when we focus on particular aggregation operations.

#### Exploiting Predicate-window Semantics over Data Streams

This paper presents a relatively weak view of how windows can work, which is understandable given its fairly early publication date. It presents a basic for executing CQL queries over a data stream, and punts on issues such as stream disorder that are tackled in later papers by other authors. Perhaps the most useful piece of this paper is the query framework that it suggests, and its initial findings on how retraction can work with negative tuples:

> In the rest of this paper we assume that the pipelined query execution model with the negative tuples
> approach [L 2] is used to process window queries over data streams. The pipelined query execution model
> for data streams is a modification of the one used in traditional database management systems [2] where
> all query operators are connected via first-in-first-out queues. In the negative tuples approach, a special
> operator, termed EXPIRE, is added at the bottom of the query pipeline; one EXPIRE operator per
> data stream. EXPIRE buffers the input stream tuples, and outputs a negative tuple whenever a tuple
> expires from the window. The negative tuple is processed by the various operators in the query pipeline
> to negate the effect (if any) of the corresponding positive tuple. The output of the continuous query is
> a continuous stream of positive and negative tuples. A negative tuple is interpreted as a deletion of a
> previously produced positive tuple.

Otherwise, the other papers listed here do a better job of suggesting designs for how windows can be implemented, and their aggregates materialized.

### Windowing API characteristics

Onyx's windowing API should:

- [x] Be a low-level data structure that Continuous Query Language (CQL) can compile to
- [x] Support fixed (tumbling) windows
- [x] Support sliding windows
- [ ] Support session windows
- [x] Support global windows
- [x] Provide enough expressivity for the window to be created by features of the data itself
- [x] Support time-based windows (event timestamps, processing timestamps) and "tuple-based" windows (e.g. window of n tuples)
- [x] Support punctuation based triggers
- [x] Support timer-based triggers
- [x] Support watermark triggers
- [x] Support percentile triggers
- [ ] Support external event triggers
- [ ] Allow triggers to compose
- [ ] Be internally optimized to use panes
- [ ] Allow expression of predicates for when a segment should enter and exit a window
- [x] Allow triggers to be reused across different windows
- [x] Support different strategies to change data after trigger fires (e.g. discarding, accumulating, etc)
- [ ] Support retraction (e.g. negative tuples)
- [x] Support incremental aggregation for things like sums
- [x] Support buffered aggregation for things like windowed joins where all the data for a window is needed
- [ ] Provide *some* support for load shedding in the case of aggregation where data must be buffered
- [ ] Provide expressivity for merging windows back together.
