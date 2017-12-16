# Performance problems with the single DOM and a responsive UI thread

A responsive UI thread is critical for a good user experience. If
the app is not listening and responding quickly to all user input,
users quickly become frustrated, or end up not being able to complete
tasks because their interactions with the app are misunderstood due
to a mismatch between internal state and the state the user thought
they were seeing.

Today there is only one DOM tree for an entire web page<sup>1</sup>
When mutated, and once script yields, the DOM updates to a new
rendering state and presents to the screen atomically.  The rendering
update involves updating as necessary the style, layout, paint,
animation, compositing and raster phases for the page. Due to the
many powerful declarative markup primitives in DOM, it is easy to make
changes that cause non-local updates to all of these phases.

This power, combined with atomic update, make it difficult to develop
scalable web apps with complex layouts and interaction models. In
particular, it makes it very difficult and in some cases impossible
for developers to implement a fully responsive UI in their app.


This document outlines many of those problems, and how they map
to the current implementations in browsers. Possible solutions
to them are in separate documents that make reference to the problem
spaces outlined here.

## *Long-Render*: Long render updates cause jank

Sometimes the mutations made to the DOM are simply too large to
process quickly. Examples including injecting complex new content,
or making changes which necessitate time-consuming layout such as
grids, tables or flexbox. While those updates are processing,
script cannot run.

## *Widget-Coordination* Updating widgets with no coordination to present together

Multiple widgets may be present on the page that use Web Components-
style communication though DOM elements and properties, come from
different frameworks, or are otherwise third-party script.

Making sure these widgets can update at the same time to the screen
is hard without running all of the updates synchronously.

## *Sync-Script*: Script must make all DOM updates synchronously, causing jank

Because the rendering pipeline updates atomically, script is forced
to make all DOM updates which need to present together to the
screen before yielding to the UI event loop. To reduce jank due
to these computations running long, apps need to put in a lot of
effort to separate rendering-related but pre-DOM-update work into
separate asynchronous tasks, then do the DOM mutation work in one
batch. A common architecture update path is:

1. Update model state
2. Map model state to view state in virtual DOM or its equivalent
3. Map the virtual dom to real DOM mutations.

The need to separate work from the 1->2 transtions from 2->3 makes
virtual DOM more attractive, and is a much harder implementation 
approach for apps which suffer from the *Widget-Coordination* problem,
as the steps above are then intertwined.

## *Main-Thread-Script*: DOM-updating script must run on the main thread

All Javascript-exposed objects other than SharedArrayBuffer must be on
only one thread a time, and the DOM classes are not transferrable.
Therefore the code that actually updates the DOM must be on the main
thread. As a result, only script which does not mutate DOM (e.g. steps
1 and 2 above) can move to a worker thread. to a worker thread. This
is is made harder by *Widget-Coordination*.

Further, sending computations back and forth from workers reduces
performance and can be cumbersome.





