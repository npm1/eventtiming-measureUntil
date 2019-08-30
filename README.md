# EventTiming: measureUntil()

## Overview

The [EventTiming API](https://github.com/WICG/event-timing) enables developers to measure event latency.
However, the current API does not allow measuring event handling when it happens asynchronously.
It exposes the synchronous duration of the event handlers and the next time a paint happens after the input event is processed, but that does not always include actions a developer would expect from the event.
In fact, some frameworks choose to execute event handling asynchronously with respect to event handlers.
In this case, the browser cannot tell when the associated tasks have been completed.
This explainer contains a proposal for augmenting Event Timing with a way to annotate async work associated with events.

## Motivation

`measureUntil()` could be somewhat polyfilled by keeping a map of events and their timestamps.
The rendering timestamp (next paint) cannot be calculated accurately but it can be approximated.
However, there are key advantages of having an explicit `measureUntil()`:

* Make it easier to associate events to their async work.
* Enable obtaining the appropriate 'long events' in EventTiming. The way events are filtered can now depend on developer annotations so that events which do not perform any synchronous work but perform long asynchronous work can be surfaced.
* Disincentivize usage of heuristics to approximate 'next paint' timestamps. These are less efficient and more inaccurate than the browser calculations.

## Proposed API

```
partial interface Event {
  void measureUntil(Promise<any> workPromise);
};
```

Any JavaScript context that has access to an Event can add a work promise to the Event by calling `measureUntil()`.
When the event dispatching algorithm is about to be completed (and we're computing EventTiming's `processingEnd`), the list of promises associated with the event are awaited.
Once all of those promises are resolved or rejected, we compute the timestamp of the next [update the rendering step](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering).
We then set the `duration` attribute of [PerformanceEventTiming](https://wicg.github.io/event-timing/#sec-performance-event-timing) as the difference between the computed timestamp and the `startTime`.

Note that this is consistent with the current definition of `duration`.
Without any promises associated to Events, the end-time is the timestamp of the first [update the rendering step](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering) after the event has been processed.
This is precisely how `duration` is currently being computed.

## Example usage

Suppose that we have a button which, when clicked, adds an image to the website.
A developer can now include the time it takes for the image to be fetched in the 'event handling time' as follows:

```javascript
function onclick(event) {
  event.measureUntil(new Promise(function(resolve, reject) {
    fetch('resources/image.jpg').then(function(response) {
      if (!response.ok) {
        reject('Could not fetch image');
        return;
      }
      const container = document.getElementById('container');
      const imgElem = document.createElement('img');
      container.appendChild(imgElem);
      const imgUrl = URL.createObjectURL(responseAsBlob);
      imgElem.src = imgUrl;
      resolve();
    }
}
```

## Problems

There are a few potential concerns with this method:

* It likely requires event listeners. This seems acceptable: if async duration is required, presumably that means that there are already events listeners in place.
* `PerformanceEventTiming` entry dispatch can be delayed arbitrarily by associating a promise which is never fulfilled.
More generally, associating more work to the entry means that it has to be dispatched later. This seems better than dispatching the entry right away without `duration`, arguably the most important attribute.
* Different calls of `measureUntil()` on the same event will affect each other because the paint timestamp is only computed once all of the promises have been fulfilled.

## Alternatives considered

One alternative would be supporting a map of async work durations.
A name can be added to `measureUntil`, and the `PerformanceEventTiming` entry would contain a map of names to durations.
This way, multiple `measureUntil()` calls will likely not affect each other very much.
We did not choose this alternative because:

* It's unclear what to do when a name is used repeatedly, as this would produce collisions.
* This enables spamming queries to a 'next paint' timestamp, so it would need to be throttled.
* There is no clear value which justifies the added complexity.

Another alternative would be to add `asyncDuration` instead of repurposing the existing `duration`.
However, given that the threshold needs to account for this new async work, and given that we expect this method to be called when there is not much synchronous work happening, it does not make sense to have both.
