---
layout: post
title: "Conc"
hidden: true
---

Before I forgot I want to tell the story of one particularly complex piece of concurrent code.
The product is IDE, and the code I'm writing analyzes the text in the editor and tries to highlight some problems. 
For example, inconsistent Java code constructs.
To do that, we need to make a model of Java code the user is trying to write in our editor.
For example, user writes 
```
final class X {}
class Y extends X {}
```
and the IDE responds with "I'm afraid you can't do that Fred, you must not extend the final class".
To understand this, we should build a hierarchy of classes the user managed to write: X <- Y.
To compute this hierarchy, we have some API method `computeAllSubClassesOfThisClass`.

This is what we did to make the method `computeAllSubClassesOf("MySuperClassName")` work faster.

(dark ages pre-2003). The first version enumerated all classes containing word `MySuperClassName` and checked if they really extend the `MySuperClassName`.
   Then the search was repeated again for the found direct subclasses recursively, to eventually find all (non-direct) subclasses.
   We had a text cache at that moment already, so we could find all files containing this text, fast.
   But that also means we had to process a lot of irrelevant classes containing the word `MySuperClassName`.

1. We made a text cache a bit smarter. Now we could ask it "give us all files containing word `MySuperClassName` in the `extends` position, meaning right after 'extends`/`implements' keyword.
   That optimization allowed the cache to return only more-or-less relevant files 
1. Highlighting daemon appeared which worked in the background, parallel to other work, so this method was made synchronized on static global object `PsiLock.LOCK` 
1. During sublasses search, all intermediate candidates were not held in memory all at once, it put memory under too much stress. Instead, just tokens (`PsiAnchor`) were stored and then `PsiClass` restored from them as necessary.
1. Queries for subclasses cached, to avoid expensive recalculation in each call
     Caches were fixed, made weak because of memory leaks
     `LocalSearchScope`: separate processing, instead of iterating all classes and filtering out the scope.
1. New data structure was introduced: concurrent lazy collection. Separate threads might access this collection to return cached results immediately. 
     If the result was not calculated yet, the thread computed the result and cached it in this collection, so that all other threads would return this cached value immediately.   
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
  
6. Deadlock fixed (due to inconcsitent lock order: `readLock`/`PsiLock`)
1. made this concurrent lazy collection was made cooperative. All these threads participated in discovering more subclasses, added them into this collection, cooperatively helping each other to speedup the process.
1. made collection store `PsiAnchor` isntead of `PsiClass`es
1. fixed bug when the exception inside computation didn't release the semaphore
1. fixed bug when `PCE` led to losing the current element being analyzed
1. fixed bug when this method tried to cache all classes (`java.lang.Object` inheritors).
   in this case the (parallel) method will try to distribute itself among `FPJ` threads and saturate all threads, which led to freeze.
1. fixed deadlock with inconsistent lock order
1. fixed `Fork-Join-Pool` starvation
1. fixed bug when `PCE` corrupted classBeingProcessed
1. fixed bug when non-physical classes were cached and incorrectly returned afterwards. Do not cache these elements, recompute them instead.
1. fixed performance issue when the subclasses were queried from `EDT`. This query led to the saturation of all threads which led to freezes. Instead, it was decided to not cache subclasses in this case at all.
1. fixed bug when waiting for all cooperationg threads to finish hanged

Moral corner.

1. Never again the concurrent cooperating threads. It's too crazy complex to do right in Java.
