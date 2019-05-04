There have been multiple [recent proposals](https://github.com/chrishtr/async-dom/blob/master/current-proposals.md) to add asynchrony to the DOM. They are aimed at different subsets of the problems of "slow rendering". "Slow rendering" generally falls into three categories:

1. "Slow rendering" could happen because the browser or web page is inefficient. For example, maybe the browser has a bug leading to slow layout in certain cases, or it scales poorly with the number of composited layers.

2. "Slow rendering" could also happen because (user input and) rendering are delayed due to other work happening that uses up the CPU. Examples of this include time spent running GC, decoding an image, or running a slow ad analytics script.

3. Finally, "slow rendering" can happen because there are simply too many things changed in the DOM or application state since the last render to expect it to update quickly. A good example is a global change of font size or page width that requires a complete re-layout of the entire DOM.

If we're in one of the above situations, what should be done? One course of action coud be to try to address part of the problem. If #1 is the problem, it's always nice if the inefficiency can just be fixed directly. At a high level, that sounds straightforward. If #2 is the problem, then the solution may be to do non-rendering work on a different thread, or run it at a lower priority than rendering work. Seems harder than #1, but potentially doable.

If #3 is the problem, then the solution becomes more murky. If the problem is "DOM changes too much", one possibility is to simply make the DOM smaller, to limit the scope of the problem. Another is to try to structure the DOM into pieces that can be processed independently, and avoid touching too much at a time. If the problem is "application state changes too much", then similar approaches can be applied to the script code that renders views of the state (make it smaller, break into independent pieces).

However, these aren't the only solutions to problem #3! Other ideas include:
 a. Using more CPU cores to do work in parallel
 b. Predicting future work and doing it in advance
 c. Breaking the DOM into pieces, and updating the pieces asynchronously from each other

** Now what?

There are a bunch of issues bound up here that make the above ideas challenging to follow up on. The issues include that there is a single monolithic DOM for each frame, that the timing & implementation of its rendering to the screen is implemented by the browser (not the developer), that JavaScript is a single-threaded programming language, that the DOM APIs are not designed for multi-threaded access, and that CSS is a very expressive, wide and powerful API, and therefore quite challenging to implement efficiently.

But these are not all the issues! This does not yet touch issues on the developer side, such as that web applications use frameworks which [provide value](https://rendering.chrishtr.org/2019/01/what-is-value-of-virtual-dom.html) by abstracting *away* from the details of the DOM, that business realities limit how much work can go into optimizing a single web app, that there are multiple browsers -- not to mention multiple versions & variants of the same browser -- to contend with, that browser performance bugfixes or APIs may not be deliverable on timeframes that are acceptable to a business, and that features sell (or are perceived to sell) web apps more than performance does.

What is needed is solutions that address the problem of "slow rendering", but that strategically minimize the issues mentioned above. Desirable properties of solutions include:

* Minimimal difference in API and behavior from the existing web APIs
* Does not preclude any use cases
* Easy enough to adopt
* Admits an incremental path of launch & iterate from a first MVP version to much more advanced ones
* Compatible with established and popular ways of writing web apps

