---
layout: post
title: "Conc"
hidden: true
---

I need to tell a story of one particularly complex piece of concurrent code.
The I'm working on an IDE, and the code I'm writing analyzes the text in the editor and tries to highlight some problems. 
For example, inconsistent Java code constructs.
To do that, we need to make a model of Java code the user is trying to write in our editor.
For example, user writes 
```
final class X {}
class Y extends X {}
```
and the IDE responds with "I'm afraid you can't do that Fred, you must not extend the final class".
To understand this, we should build a hierarchy of all classes the user managed to write: X <- Y.
To compute this hierarchy, we need to find all classes that extends this class, fast.

This is a rought `git` history of what was done to make this method work faster:

1. The dark ages: pre-2003. The first version enumerated all classes containing our class name and checked if the one does really extend the other.
   Then the search was repeated again for the found direct subclasses recursively, to eventually find all (non-direct) subclasses.
   We had a text cache at that moment already, so we could find all files containing these names, fast.
   But that also meant we had to process a lot of irrelevant classes containing this word by accident.
1. The text index became a bit smarter. Now we could ask "*give us all files containing word `MySuperClassName` in the `extends` position, meaning right after 'extends`/`implements' keyword*".
   That optimization allowed the analyze only more-or-less relevant files. 
1. Highlighting daemon appeared which worked in the background, parallel to other work, so this method was made synchronized on static global object `PsiLock.LOCK`. 
1. During sublasses search, all intermediate candidates were not held in memory all at once, because it put memory under too much stress.
   Instead, just the light-weight references (`PsiAnchor`) were stored and then `PsiClass` restored from them as necessary.
1. Queries became cachedeable, to avoid expensive recalculation in each call.
1. New data structure was introduced: concurrent lazy collection. Separate threads might access this collection to return cached results immediately. 
   If the result was not calculated yet, the thread started to compute the result and cached it in this collection, so that all other threads would return this cached value immediately.
   After that the subclasses calculations might proceec in parallel with the collection iteration, given that the majority of client only requested one inheritor.
   
   <div class="svg-anim-js" data-delay-ms="2000" aria-label="Animated diagram">
    <img class="svg-anim-js__img"
       src="/assets/anim/concCollectionStep1.drawio.svg"
       alt="Animation frame"
       decoding="async">
   </div>

   <style>
     /* Optional sizing — tweak as you like */
     .svg-anim-js { width: 100%; max-width: 1200px; }
     .svg-anim-js__img { width: 100%; height: auto; display: block; }
   </style>

   <script>
   (() => {
     // Frames (absolute paths)
     const frames = [
    "/assets/anim/concCollectionStep1.drawio.svg",
    "/assets/anim/concCollectionStep2.drawio.svg",
    "/assets/anim/concCollectionStep3.drawio.svg",
    "/assets/anim/concCollectionStep4.drawio.svg",
    "/assets/anim/concCollectionStep5.drawio.svg",
    "/assets/anim/concCollectionStep6.drawio.svg",
   ];

     // Support multiple animations on one page
     document.querySelectorAll(".svg-anim-js").forEach((root) => {
       const imgEl = root.querySelector(".svg-anim-js__img");
       const delayMs = Number(root.dataset.delayMs || 2000);

     // Preload all frames first to avoid white flashes
     const loaded = new Array(frames.length).fill(false);

    function preloadOne(i) {
      return new Promise((resolve) => {
        const im = new Image();
        im.decoding = "async";
        im.onload = () => { loaded[i] = true; resolve(true); };
        im.onerror = () => { loaded[i] = false; resolve(false); };
        im.src = frames[i];
      });
    }

    Promise.all(frames.map((_, i) => preloadOne(i))).then(() => {
      // Start animation only after preload attempt
      let idx = 0;
      imgEl.src = frames[idx];

      setInterval(() => {
        const next = (idx + 1) % frames.length;

        // Switch only if next frame is loaded; otherwise keep current
        if (loaded[next]) {
          idx = next;
          imgEl.src = frames[idx];
          }
        }, delayMs);
      });
    });
  })();
  </script>
  
  **💀Nightmare started here in earnest💀:**
  
7. Deadlock fixed (due to inconsistent lock order: `readLock`/`PsiLock`)
1. Made this concurrent lazy collection cooperative. All these threads participated in discovering more subclasses, added them into this collection, cooperatively helping each other to speedup the process.
   
**💀Nightmare intensifies💀**

9. Made collection store `PsiAnchor` isntead of `PsiClass`es.
1. Fixed bug when the exception inside computation didn't release the semaphore correctly.
1. Fixed bug when `ProcessCanceledException` led to losing the current element being analyzed.
1. Fixed bug when we were asked for `java.lang.Object` inheritors and tried to cache all possible classes as a result (since every class is an `java.lang.Object` inheritor).
   In addition to insane memory usage, we tried to distribute the search among all `ForkJoinPool` threads and saturate them all, causing freezes.
   We disabled caching and parallelization in this case, having decided that everyone who tried to query all `java.lang.Object` inheritors deserved to be punished.
1. Fixed deadlock with inconsistent lock order (`readLock`/internal lock)
1. Fixed `Fork Join Pool` starvation. We needed to spawn new FJP threads when the clients block.
1. Fixed bug when `ProcessCanceledException` corrupted `classBeingProcessed` collection.
1. Fixed bug when non-physical classes were cached and incorrectly returned afterwards. These are special light elements that can't be cached but recomputed each time.
1. Fixed performance issue when the subclasses were queried from `EventDispatchThread`. This is a special UI-realted thread and any delay in this thread would cause visible freezes. It was decided to not cache subclasses in this case at all.
1. Fixed bug when waiting for all cooperationg threads hanged indefinitely.

Moral corner.

1. Never again the concurrent cooperating threads. It's too crazy complex to do right in Java.
   - How would several threads help each other?
   - How to distribute work between them safely and non-blockingly?
   - How to ensure the work is distributed to as many CPU cores as possible?
   - After we distributed work to all available CPU, how to avoid thread starvation when some threads waiting for other to complete and not letting other threads to do any work?
   - After we distributed work to all available CPU, and every thread does meaningful work, how to make sure that everything else, especially more important stuff, is not slowed down?
   - After we saturated all CPU resources, how to make it all safely intrruptible?
   - and the remaining 90%
