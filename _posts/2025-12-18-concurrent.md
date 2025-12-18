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
   
   [anim](/assets/concCollectionAnim.html)

<!-- Embeddable SVG frame animation (CSS-only, loop-correct, order-stable) -->
<div class="svg-anim-wrap">

  <!-- Hidden preload -->
  <div class="svg-anim-preload" aria-hidden="true">
    <img src="/assets/anim/concCollectionStep1.drawio.svg">
    <img src="/assets/anim/concCollectionStep2.drawio.svg">
    <img src="/assets/anim/concCollectionStep3.drawio.svg">
    <img src="/assets/anim/concCollectionStep4.drawio.svg">
    <img src="/assets/anim/concCollectionStep5.drawio.svg">
    <img src="/assets/anim/concCollectionStep6.drawio.svg">
  </div>

  <div class="svg-anim" aria-label="Animated diagram">
    <img class="frame f1" src="/assets/anim/concCollectionStep1.drawio.svg">
    <img class="frame f2" src="/assets/anim/concCollectionStep2.drawio.svg">
    <img class="frame f3" src="/assets/anim/concCollectionStep3.drawio.svg">
    <img class="frame f4" src="/assets/anim/concCollectionStep4.drawio.svg">
    <img class="frame f5" src="/assets/anim/concCollectionStep5.drawio.svg">
    <img class="frame f6" src="/assets/anim/concCollectionStep6.drawio.svg">
  </div>
</div>

<style>
  .svg-anim-wrap { margin: 1rem 0; }

  .svg-anim {
    position: relative;
    width: 100%;
    max-width: 1200px;
  }

  .svg-anim::before {
    content: "";
    display: block;
    padding-top: 100%;
  }

  .svg-anim > img.frame {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    opacity: 0;
    z-index: 0;
    will-change: opacity, z-index;
    transform: translateZ(0);
  }

  /* Preload container */
  .svg-anim-preload {
    position: absolute;
    width: 0;
    height: 0;
    overflow: hidden;
    opacity: 0;
  }

  /* Animation: 6 frames × 2s = 12s */
  .svg-anim > img.frame {
    animation: frame 12s infinite linear;
  }

  /* NEGATIVE delays = seamless looping */
  .f1 { animation-delay:  0s; }
  .f2 { animation-delay: -2s; }
  .f3 { animation-delay: -4s; }
  .f4 { animation-delay: -6s; }
  .f5 { animation-delay: -8s; }
  .f6 { animation-delay: -10s; }

  /* The key part: opacity + z-index together */
  @keyframes frame {
    0%   { opacity: 0; z-index: 0; }
    5%   { opacity: 1; z-index: 2; }
    45%  { opacity: 1; z-index: 2; }
    50%  { opacity: 0; z-index: 0; }
    100% { opacity: 0; z-index: 0; }
  }

  @media (prefers-reduced-motion: reduce) {
    .svg-anim > img.frame { animation: none !important; }
    .svg-anim > img.frame:first-of-type { opacity: 1; z-index: 1; }
  }
</style>


1. Deadlock fixed (due to inconcsitent lock order: `readLock`/`PsiLock`)
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
