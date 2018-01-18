# Explainer: display locking

We introduce a new concept for elements: locked for display.

Locked for display means that (a) the pixel output of that element and its
subtree, including descendant shadow roots, is fixed and cannot change, and
(b) the subtree is inert, so it may not receive any browser events. Locking
for display applies only to elements with contain:contents style. The user
agent may force unlocking at any time if the UI appears frozen for too long.

There are four associated new methods, one new event, and one new class
(Lock):

* Lock.requestUpdate(<function>): registers a request for an async callback
into <function> in the future. The callback will always be called, but if the
commit has already started, will be called once commit and unlock have
finished. If the lock is finished, then requestUpdate behaves like
requestAnimationFrame.

* Lock.requestCommit(): Registers a request for the user agent to do the work
to commit the element’s subtree contents to the screen. The commit begins
after all outstanding requestUpdate callbacks for this lock complete. Returns
a promise which is resolved when the request is fulfilled. This happens after
the work is done and the promise is resolved/rejected, but is otherwise not
timed to rAF.

* Lock.isCommitting(): returns true if so.

* Lock.onUnlock: called when a lock becomes unlocked, perhaps due to the user
agent forcing it and perhaps due to a commit.

* Element.requestLock(<function>): Locks the element for display, then calls
<function> with the lock as a parameter. Returns a promise which is resolved
once the callback function and any for a child element, has returned and the
resulting DOM below the element is ready for display to the screen. If an
ancestor is already locked, the child element is locked together with teh
ancestor. Likewise, if a descendant is already locked, the lock is the same,
and this means the lock is extended to the ancestor’s subtree.



### Example:

~~~~
function readyToUnlock() {
  // Last chance to avoid displaying the new widget state if abort is needed.
  // Options are to re-lock the element, detach its DOM element, or otherwise
  // mutate it in some way.
}

function updateWidgetContents(lock) {
  if (moreWorkToDo()) {
    doSomeWork();
    lock.requestDisplay(updateWidgetContents());
  }
  lock.commit().then(readyToUnlock);
}

widgetRoot.requestLock(updateWidgetContents).then(readyToUnlock);
~~~~

### Why this proposal vs another?

There are several other related/similar proposals in this area, including
asyncAppend, DOM changelists, async documents, and likely others. The
advantages of this proposal over others stem from designing the APIs and
concepts to map very closely to existing Blink implementation-internal
concepts, and to how developers update the DOM.

In particular, this proposal:

* Is straightforward to implement utilizing the DOM-tree-walk nature of Blink
rendering, and adds only one new concept (caching display output). It is
expected that other user agents have very similar internal implementations and
therefore can also use the same implementation strategy. Further, state
complexity is minimized because under this proposal, the DOM as viewed by
script is always in a consistent state.

* Admits an incremental launch-and-iterate style, since Blink need not perfectly
chunk its work into <=16ms pieces right away, and can instead iteratively
approach this goal through iterative improvement, implementing in parallel
among the teams & launching these improvements when ready. Allows for all
performance optimizations of Blink rendering to be retained. For example,
repaint-after-layout, style caching, layout root optimizations, transform fast
paths, and image decode caching strategies are all retained in a natural way.

* Is compatible with the ways existing web apps update themselves. Well-behaved
existing web apps (e.g. React or Polymer apps) need only wrap their model ->
view rendering update code in requestLock() + requestCommit(), plus an
affordance to indicate the DOM is updating. Web Apps which desire to run their
rendering code on a worker can do so, then serialize the result into a locked
subtree in a chunked manner. Can be aborted by the user agent without a broken
UI.

asyncAppend, for example, fails short in items (c)  and (d) because it does
not allow mutating existing DOM. Also note that asyncAppend can be built on
top of this proposal.

## Implementation sketch

**Definition**: an element subtree
that is locked for display can be in one of two modes, (a) quiescent or (b)
committing (if requestCommit has been called and started).

### Rendering 

The Blink lifecycle update proceeds in a series of DOM/Layout tree walks
starting at the root. These tree walks all maintain dirty bits about which
elements need some sort of data recomputed, and whether any descendants do.
This implementation naturally admits an incremental algorithm for chunked
work.

Each lifecycle tree walk would proceed as normal. When it arrives at an
element which is the root of a locked subtree:
* If the root locked element is
in a quiescent subtree, most tree walks will early-return from recursion at
the root locked element. Dirty bits for the locked subtree will be retained.
Exceptions: 
  * Layout should use the specified size of the root locked element,
if provided, or else the layout size it had before becoming quiescent, or else
empty.
  * Paint should early out at the root locked element but output the
display list it had before becoming quiescent, if any.
* Otherwise, the root
locked element is in a committing subtree. In this case:
  * Keep doing work including update to the subtree, until a certain quota
  of subtree work has been done, then yield. Clear dirty bits for work
  completed.
  * Even if some paint work happens for the subtree, ignore it and output
   the display list it had before becoming quiescent, if any.

### Input events

  While locked, the subtree is inert. The inert attribute is currently being
implemented. Also, the current spec encourages the user agent to still allow
text search into inert elements, which is likely not feasible for locked
subtrees which are in the process of being mutated. Therefore the spec should
require that text search not work for quiescent or committing subtrees.

## Notes

### Are there any gotchas for developers?

Display locking requires making
the existing subtree inert during update. This results in some complications
and new potential failure modes for developers that they need to take into
account:
1. Developers must take into account the fact for long-running commits
the subtree will appear frozen to the user for some time, and show loading
/grayout-style affordances or similar, and potentially also queue up input
events for later replay.
2. If a subtree remains locked indefinitely due to
programmer error, users will be unable to click on links or select text in
that subtree, breaking page functionality. The UI pattern proposed here
involving the required use of requestUpdate is meant to mitigate this
possibility. The user agent should also implement timeout behavior that forces
unlock after a suitable timeout delay.
3. Nested locking requires some care from
developers. For example, nested widgets might want to update in overlapping
time periods. The API guarantees that they share a lock in this case. Before
the lock is committing, requestUpdate() callbacks keep the lock alive and not
yet committing, allowing easy coordination among widgets (good). But if the
lock is already committing at the time it is acquired, then requestUpdate will
happen after commit, and unlocked. It may be a good idea to request a new lock
later.
4. It is generally a design (though not correctness!) error to mutate a
locked subtree outside of requestUpdate, because that may lead to duplicated
work.

### What happens if a developer calls a style, layout or hittesting-inducing
method?

In these cases, the user agent should force a layout, just like is
done today. Developers should be advised not to call these methods at the
wrong time or expect jank. Devtools warnings can be added to help prevent
these situations.

Examples include getComputedStyle (requires style), offsetTop (requires
layout), and elementFromPoint (requires hit testing).

### What happens for custom
elements?

Custom event created-callbacks, microtasks, etc. all happen as usual
when mutating the dom (i.e. during or just after one or more of the
requestUpdate callbacks).

### When does a ResizeObserver run?

These run after layout is complete during commit.

### Isn’t it a lot of work to provide good affordances and input event queueing during lock to preserve the user experience?

Yes and no. If a developer uses
display locking without any affordances, they are replacing one kind of user
experience failure with another:
  * Stale events: Without display locking, UI-
thread-processed user events will not be processed until style+layout+paint
complete, and then will be executed in batch. These events may be stale in
that the user intended them for a widget state which no longer exists (input
box disappeared, click target moved). Bad.
  * Ignored events: With display
locking, the user events will be sent to the UI thread, and if the script
doesn’t listen for them, they will be lost. Thus events which ended up not
being stale will instead be ignored. Bad.

It is unclear which is worse. It seems fair to say that clicking on an
unintended target can be worse than ignoring a click, since the unintended
click may cause a navigation or other strange behavior.

Note that without display locking, the stale events problem cannot be solved
because script cannot run during style+layout+paint update. With display
locking, a simple user affordance such as a semi-transparent gray retangle, is
easy to implement and compatible with user expectations around ignored UI
events during “updating…” states. Further, it is possible (though more work)
to queue up the events and replay them, if so desired.

### What happens if a widget embeds a badly-behaved subwidget?

If this happens, the sub-widget may delay the commit. In these cases there
should be a way for the parent widget to commit itself, even if the sub-widget
remains committed.
