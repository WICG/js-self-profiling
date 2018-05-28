# JavaScript Self-Profiling API Proposal

## Motivation

Currently it is difficult for web developers to understand how their applications perform in the wide variety of conditions encountered on real user devices. A programmable JS profiling API is needed to make it possible to collect JS profiles from real end-user environments.

A native self-profiling API for JS code would also allow web developers to efficiently find hotspots in their JS code during page loads and user interactions, to assign CPU budgets to individual JS-implemented features on the page, to find unnecessary work being done on the client, and to find low-priority JS code executing in the background and wasting device power.

Currently JS self-profiling can be done by instrumenting individual JS functions with timing code but this is cumbersome, bloats JS size, changes the code (potentially altering performance characteristics), adds overhead from timing calls, and risks missing out on hotspots in unexpected corners.

## Facebook's Profiler Polyfill

In an attempt to polyfill the missing self-profiling functionality, Facebook built and deployed its own in-page JS profiler implemented using JS and SharedArrayBuffers. This JS profiler was implemented using a worker thread that signaled to the main thread when it needed to record its current stack. The worker thread would toggle a “capture stack now” flag in a SharedArrayBuffer every few milliseconds, and the value of this flag was read by instrumentation code that was inserted at transpilation time into the beginning of interesting JS functions running on the main thread. If the instrumentation code saw that the flag was set, it would capture the current JS stack (using the Error object) and add it to the running profile.

This JS profiler was enabled for only a small percentage of Facebook users, and only instrumented functions of 10 statements or more in order to limit performance impact and to limit the quantity of profiling data collected. Nevertheless, we found it extremely valuable for understanding Facebook.com's performance in the field and for finding optimization opportunities. 

This polyfill implementation had some downsides:

* The size overhead of the added instrumentation required limiting the fraction of functions instrumented (resulting in incomplete coverage)
* The sampling “interrupt” was not instant so stacks were collected from the **_next_** instrumented function which added a lot of noise to the dataset and made it difficult to reason about
* The performance overhead limited the sampling frequency and fraction of pageloads profiled
* Our design depended on SharedArrayBuffers, which have been disabled on the Web for the time-being, so the profiler is disabled for now

A browser-implemented profiling API would avoid these downsides.

## Wider Industry Interest

It is cumbersome to manage instrumentation for performance measurement, regardless of whether it is inserted by build-time tooling (like Facebook's polyfill above) or inserted by hand in functions of interest. This API eliminates the need to maintain performance instrumentation and therefore allows smaller sites to deploy in-page JS profiling.

We also expect that third-party analytics providers will offer libraries or infrastructure to record, ingest, aggregate and automatically analyze the collected profiles for optimization opportunities, thus further lowering the barrier to entry.

Finally, several other Web properties with large codebases have expressed interest to us for using this API to better monitor their webapps performance in the field.

## API Requirements

Minimum of functionality:

* Ability to profile same-origin JavaScript executing on the main thread of the current page frame. The collected execution stacks should be similar in format to Error object stacks. Note that the profiler would not need to record native DOM calls.
* Some way to communicate the accuracy of the collected profile: either timestamps for each sample or a calculated “effective” profiling interval.

Wishlist features:

* Add performance.mark() data to collected profiles
* Ability to profile arbitrary threads or all threads from the same origin, including WebWorker, SharedWorker and ServiceWorker threads
* Ability to have multiple profiling requests happening at the same time (e.g. both mousedown and mouseup issue a startProfiling command before either of them stops profiling)
* Ability to specify a circular buffer size
* An API to get the capabilities of the browser's profiling system. For example, a browser might only support profiling intervals higher than 2ms in order to defeat timing attacks if they don't support site isolation

Future ideas:

* Support for returning source-mapped profiles
* Tracing instead of sampling
* Incorporating DOM API calls into traces
* Incorporating timing information for non-JS events (e.g. time spent in layout or styling)
* Ability to profile across iframes from the same origin

## Minimal API Proposal

* performance.startProfiling(options) => Promise that resolves with a Profiler object
    * “options” is a JS object containing the profiling configuration:
        * options.interval: profiling interval in milliseconds (float)
        * options.version: profiling format version (integer)
    * The call returns immediately with a Promise. The promise is resolved to a Profiler object when the browser starts profiling the target thread(s) with the provided configuration. The promise is rejected if the profiler failed to start.
        * Example: performance.startProfiling().then(profiler => doInterestingWorkload()).catch(/* failed to start profiling or other error */)
        * The Profiler object has the methods stopProfiling and cancelProfiling
* Profiler.stopProfiling() => Promise that resolves with a ProfilingResult object
    * Stops profiling, returns a Promise immediately that resolves to a ProfilingResult object (see below) or rejects with an error (e.g. if profiling is not active)
* Profiler.cancelProfiling() => Promise
    * Cancels profiling started by performance.startProfiling(). Returns a Promise that resolves after profiling is cancelled or rejects if there is an error.
    * Unlike stopProfiling, cancelling does not require the browser to do any extra processing before returning the collected samples (e.g. gathering samples from multiple threads, calculating effective sampling rate).

## Possible JSON Profile Format

Space-efficient encodings are possible given the tree-like structure of profile stacks and the repeated strings in function names and filenames. A trie seems like a natural choice. A trie is also desirable because the raw uncompressed memory representation causes huge memory pressure and is more expensive to analyze (e.g. to count stack frequencies).

There are examples of trie-based approaches in browsers today:

* Chrome's Trace Event format, specifically the stackFrames field: [https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview#heading=h.yr703knxre9f](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/)
* Firefox's Gecko Profiler format, specifically the stackTable field: https://github.com/devtools-html/perf.html/blob/master/docs-developer/gecko-profile-format.md#source-data-format

An idea for a possible format:

```
{
    version: 1,
    options: {
        <configuration actually used for profiling>
    },
    profile: {
        strings: [
            "start @ main.js:12:34"
            "render @ newsfeed.js:56:78",
            "getLikes @ stories.js:90:12",
            "getComments @ stories.js:11:22"
        ]
        stackFrames: {
            ...
            101: { name: 0 },
            102: { name: 1, parent: 101 },
            103: { name: 2, parent: 102 },
            104: { name: 3, parent: 102 }
        },
        main-thread: [
            {
                "type": "stack",
                "timestamp": 12345.67890000,
                "stack": 103,
            },
            {
                "type": "stack",
                "timestamp": 12346.00000000,
                "stack": 104,
            },
            ...
        ]
    }
}    
```

Open questions:

* How to represent source locations for <script> tags, inline event handlers, etc.
* How to represent a call to native DOM like innerHTML or "style recalc" or JS-builtins like Math.random()
* How should async events between threads be annotated?

## Visualization

Mozilla's perf.html visualization tool for Firefox profiles or Chrome's trace-viewer (chrome://tracing) UI could be trivially adapted to visualize the data produced by this profiling API.

## perf.html

As an illustration, Mozilla's perf.html project shows the JS stack aggregation and timeline. It is able to show gaps where JavaScript was not executing, areas where there were long running events (red), and an aggregate view of the samples in the selected timerange such as the 15 contiguous samples in function 'user.ts' highlighted in the screenshot below.
