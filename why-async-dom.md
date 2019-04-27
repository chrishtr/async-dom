There have been multiple [recent proposals](https://github.com/chrishtr/async-dom/blob/master/current-proposals.md) to add asynchrony to the DOM. They are aimed at different subsets of the problems of slow rendering.
WorkerNode or WorkerDOM try to offload widget rendering lifecycle work onto a worker thread, in order to increase parallelism and avoid contention with other script and non-script work on the main thread. DOM changelists is similar.
asyncAppend provides a subset of functionality that DisplayLocking does. Both provide some form of transactional-like update of a DOM subtree.

Ultimately, rendering can be slow for three fundamental reasons:
1. Developer or browser code is poorly optimized for the use-case
2. Non-rendering main-thread tasks slow things down
3. There is too much work to do

Issue #1 can be solved with a sufficient amount of effort on the part of the developer or browser engineer to optimize the code to be sufficiently fast. The same goes for issue #2, by chunking up the non-rendering code, scheduling it appropriately,  or avoiding it entirely when not needed. Further, issue #3 can be solved by doing less work, or changing the web app UI to introduce affordances such as pagination.

1. Business incentives.
2. Layout will always be slow.
3. Developer extensibility for fast paths.
