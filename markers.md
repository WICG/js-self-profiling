# JS Self-Profiling Markers

## Introduction

It is difficult to analyze a trace without visibility into the context around it and the events competing for the UI thread such as rendering or painting.
By adding markers to the captured samples, web developers will be able to correlate slow traces with the browser activity.

End-users will benefit from faster and more efficient websites by giving the tools to web-developers to understand how their application performs on conditions encountered on real user devices.

## Goals 

* Being able to identify the browser activity at the time the sample was captured
* Being able to differentiate scripting, rendering, painting and GC related activities.

## Context

### Javascript profilers

Javascript profilers offered by User Agents give detailed insights to the browser activity during the trace,  common categories include: scripting, rendering and painting. The level of details and the events themselves can vary per browser, see [Gecko](https://github.com/mozilla/gecko-dev/blob/master/devtools/client/performance/modules/markers.js#L12) and [v8](https://source.chromium.org/chromium/chromium/src/+/main:third_party/devtools-frontend/src/front_end/panels/timeline/TimelineUIUtils.ts;l=1252?q=TimelineUIUtils.ts&ss=chromium).

In addition to highlight browser activity around task execution, profilers also show events that may pause stack execution like [Garbage Collection](https://source.chromium.org/chromium/chromium/src/+/main:third_party/devtools-frontend/src/front_end/panels/timeline/TimelineUIUtils.ts;l=1339?q=TimelineUIUtils.ts&ss=chromium). This type of browser activity is not detectable with stacks alone and make trace analysis more difficult.

### Trace

Details about the Javascript Self-Profiling API trace format can be found [here](https://wicg.github.io/js-self-profiling/#the-profilertrace-dictionary).

Generally, a trace is represented as a collection of sample with a **timestamp** and **stack id** fields:

```
{
    "stacks" : [
        {"frameId" : 0},
        {"frameId" : 1,"parentId" : 0},
        {"frameId" : 2,"parentId" : 1}
    ], 
    "samples" : [
        {"timestamp" : 1551.73499998637,"stackId": 2},
        {"timestamp" : 1576.83999999426,"stackId": 1},
        {"timestamp" : 1601.90499993041}
    ],
}
```

## Format proposal


The proposal is to add a **marker** member to the sample dictionary:

* `marker`: optional string, if member is missing idle state is assumed

The marker represents the type of work being executed by the User Agent. Since some work like GC may interrupt stack execution it is necessary to add a marker to a sample with a stack to differentiate script execution from any other type of work.
 
The marker can be one of the following strings:

* `script`: script related activity as specified in the [script processing model](https://html.spec.whatwg.org/#script-processing-model)
* `gc`: garbage collection related activity
    For trace analysis developers need to be able to identify frames that were not executing on the main thread due to a garbage collection.
    The W3C design principle [recommends](https://w3ctag.github.io/design-principles/#js-gc) not exposing the timing of a garbage collection and APIs depending on GC must set clear expectation about their interaction with the GC, see [WeakRef](https://tc39.es/ecma262/#sec-weak-ref-objects).
    Considering the GC timing is exposed after the complete trace is captured with a resolution 10 ms, this marker type should not be considered a weak reference to a GC event. 
* `other`: all other activities that do not fit the other classification


Rendering stages are not fully specified in the event loop processing model, see [Update the rendering](https://html.spec.whatwg.org/#update-the-rendering).
But we can envision 3 markers corresponding to the stages described in draft document [resize-observer](https://www.w3.org/TR/resize-observer/#html-event-loop):
* `style`
* `layout`
* `paint`

Additionaly UAs may support additional markers they consider relevant for performance analysis on their platform. Examples of those markers are:

* `script-parser`:  script related activity, converting a script into an intermediate representation like an Abstract Syntax Tree
* `script-compile`: script related activity, converting a script into a bytecode representation for execution.

```
enum ProfilerMarker { "script", "gc", "style", "layout", "paint", "other" };

dictionary ProfilerSample {
  required DOMHighResTimeStamp timestamp;
  unsigned long long stackId;
  [CrossOriginIsolated] ProfilerMarker? marker;
};
```

## Key scenarios

### Browser activity

In the current implementation a trace captured by a web application developer shows gaps between each stack that cannot be interpreted.
The addition of markers to samples helps developers understand what are the browser operations that made the trace slow outside of javascript and enables them to minimize style or layout impact for example. 

trace with markers:

```
  "samples" : [
    {
      "timestamp" : 100,
      "stackId": 2,
      "marker": "script"
    },
    {
      "timestamp" : 110,
      "stackId": 1,
      "marker": "script"
    },
    {
      "timestamp" :120,
    }
    {
      "timestamp" :130,
        "marker": "style"
    },
    ....
    {
        "timestamp" :150,
        "marker": "layout"
    },
    ....
    
    {
        "timestamp" :170,
        "marker": "paint",
        "stackId": 3
    },
    {
      "timestamp" : 180,
      "stackId": 3,
      "marker": "script"
    }
```

### Report layout thrashing.

Some properties or methods may trigger the browser to recompute the style or layout. 
Without a marker tagging a sample for a render operation, there is no easy way to differentiate script execution from a rendering operation.
This is especially useful considering that operations triggering a re-flow are UA dependent and are subject to change.

trace without markers:

```
  "samples" : [
    {
      "timestamp" : 100,
      "stackId": 2, // operation recomputing the layout
    },
    {
      "timestamp" : 110,
      "stackId": 2, 
    },
    {
      "timestamp" : 120,
      "stackId": 1,
    },
    {
      "timestamp" :130
    }
```

trace with markers:

```
  "samples" : [
    {
      "timestamp" : 100,
      "stackId": 2,
      "marker": "layout"
    },
    {
      "timestamp" : 110,
      "stackId": 2,
      "marker": "layout"
    },
    {
      "timestamp" : 120,
      "stackId": 1,
      "marker": "script"
    },
    {
      "timestamp" :130
    }
```

### Long trace due to GC activity

A sample was captured indicating 30 ms spent on stack 2. A major GC happened at timestamp 105 that took 20 ms to complete.
With the addition of markers, a web application developer will be able to attribute the longer duration to a GC event.
This can be used to dismiss longer samples due to random GC events or to focus on the memory allocation pattern of his application if this type of traces increases over time.

Trace without browser markers:

```
  "samples" : [
    {
      "timestamp" : 100,
      "stackId": 3
    },
    {
      "timestamp" : 110,
      "stackId": 2
    },
    {
      "timestamp" : 120,
      "stackId": 2
    },
    {
      "timestamp" : 130,
      "stackId": 2
    },
    {
      "timestamp" : 140,
      "stackId": 1
    },
    {
      "timestamp" : 150
    }
```

Trace with markers:

```
  "samples" : [
    {
      "timestamp" : 100,
      "stackId": 3,
      "marker": "script"
    },
    {
      "timestamp" : 110,
      "stackId": 2,
      "marker": "script"
    },
    {
      "timestamp" : 120,
      "stackId": 2,
      "marker": "gc"
    },
    {
      "timestamp" : 130,
      "stackId": 2,
      "marker": "gc"
    },
    {
      "timestamp" : 140,
      "stackId": 1,
      "marker": "script"
    },
    {
      "timestamp" :150
    }
```
## Privacy and Security

Careful consideration must be taken to avoid leaking top level UA work performed on a cross-origin document. UAs must only expose a marker if the responsible document for the work is same-origin with the profiler.

There is a risk to introduce a new source of side channel information through this API. Specifically, the timings of cross-origin opaque resources owned by the document that do not pass a Timing-Allow-Origin check or the timings of cross-origin documents hosted by the same process. To mitigate this risk, a Sample's marker attribute may only be accessible when the current Realm's settings objects's cross-origin isolated capability boolean is set to true.

Additional checks may also be required by user agents to implement this feature.
## Considered alternatives

Instead of setting a marker at the sample level, it is possible to attach the marker to a stack element instead.
A single JS stack may be duplicated to represent the different markers that it’s been associated with during the sampling.
Browser activity not linked to a script may be represented by a top level entry with a marker field.

This approach has the disadvantage of making the stack resolution logic more complex and breaks the one to one mapping between the script’s stack and the trace’s stack.
For example the `parentId` attribution is no longer clearly defined. In case of a duplicate stack we have multiple candidates and defaulting to the stack marked `script` is not an option since that stack is not guaranteed to exist.

Using the same GC example the trace would be presented like this:

```
{
    "stacks" : [   
        {"marker" : "idle"}, // id: 0
        {"marker" : "script"}, 
        {"marker" : "parse"},
        {"marker" : "gc"},
        {"marker" : "paint"},
        {"marker" : "other"},
        {"frameId" : 1, "marker": "script"}, // id: 6
        {"frameId" : 2, "parentId" : 6, "marker": "script"}, // id: 7
        {"frameId" : 2, "parentId" : 6, "marker": "gc"}, // id: 8
        {"frameId" : 3, "parentId" : 8, "marker": "script"} // id: 9
    ], 
    "samples" : [
        {
            "timestamp" : 100,
            "stackId": 9,
        },
        {
            "timestamp" : 110,
            "stackId": 7,
        },
        {
            "timestamp" : 120,
            "stackId": 8,
        },
        {
            "timestamp" : 130,
            "stackId": 8,
        },
        {
            "timestamp" : 140,
            "stackId": 6,
        },
        {
            "timestamp" :150,
            "stackId": 0
        }
}
```

## Security and Privacy questionnaire

`01. What information might this feature expose to Web sites or other parties,`
` and for what purposes is that exposure necessary?`

This feature exposes user agent work on the main thread during a trace captured with JS Self Profiling API. This information is limited to high level categories of work that help web developers understand and improve the performance of their application.

` 02. Do features in your specification expose the minimum amount of information`
` necessary to enable their intended uses?`

Yes.

` 03. How do the features in your specification deal with personal information,`
` personally-identifiable information (PII), or information derived from`
` them?`

Does not expose PIIs.

` 04. How do the features in your specification deal with sensitive information?`

Does not expose sensitive information.

` 05. Do the features in your specification introduce new state for an origin`
` that persists across browsing sessions?`

No.

` 06. Do the features in your specification expose information about the`
` underlying platform to origins?`

Does not expose data about the underlying platform.

` 07. Does this specification allow an origin to send data to the underlying`
` platform?`

No.

` 08. Do features in this specification enable access to device sensors?`

No.

` 09. What data do the features in this specification expose to an origin? Please`
` also document what data is identical to data exposed by other features, in the`
` same or different contexts.`

There is concern for a class of user agent work that may be shared across origins. Currently only Garbage collection event is concerned by this and may be mitigated by requesting cross-origin isolation.
Other events are only exposed if the active profiler belongs to the same origin.

` 10. Do features in this specification enable new script execution/loading`
` mechanisms?`

No.

` 11. Do features in this specification allow an origin to access other devices?`

No.

` 12. Do features in this specification allow an origin some measure of control over`
` a user agent's native UI?`

No.

` 13. What temporary identifiers do the features in this specification create or`
` expose to the web?`

No temporary identifier created.

` 14. How does this specification distinguish between behavior in first-party and`
` third-party contexts?`

No distinction.

` 15. How do the features in this specification work in the context of a browser’s`
` Private Browsing or Incognito mode?`

Semantics should be unchanged.

` 16. Does this specification have both "Security Considerations" and "Privacy`
` Considerations" sections?`

Yes.

` 17. Do features in your specification enable origins to downgrade default`
` security protections?`

No.

` 18. How does your feature handle non-"fully active" documents?`

The JS Self Profiling API only capture samples from the Main thread and does not interact with non-fully active documents.

` 19. What should this questionnaire have asked?`
