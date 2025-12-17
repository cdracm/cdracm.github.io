---
layout: post
title: "Conc"
published: false
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

2003.2 We made a text cache a bit smarter. Now we could ask it "give us all files containing word `MySuperClassName` in the `extends` position, meaning right after 'extends`/`implements' keyword.
   That optimization allowed the cache to return only more-or-less relevant files 
2005 Highlighting daemon appeared which worked in the background, parallel to other work, so this method was made synchronized on static global object PsiLock.LOCK 
2015 During sublasses search, all intermediate candidates were not held in memory all at once, it put memory under too much stress. Instead, just tokens (PsiAnchor) were stored and then PsiClass restored from them as necessary.
2016 Queries for subclasses cached, to avoid expensive recalculation in each call
     Caches were fixed, made weak because of memory leaks
     LocalSearchScope: separate processing
2016.04 New data structure was introduced: concurrent lazy collection. Separate threads might access this collection to return cached results immediately. 
     If the result was not calculated yet, the thread computed the result and cached it in this collection, so that all other threads would return this cached value immediately.
2016.04. Deadlock fixed (due to inconcsitent lock order: readlock/psilock)
2016.04 made this concurrent lazy collection was made cooperative. All these threads participated in discovering more subclasses, added them into this collection, cooperatively helping each other to speedup the process.
2016.04 made collection store PsiAnchor isntead of PsiClasses
2016.05 fixed bug when the exception inside computation didn't release the semaphore
2016.05 fixed bug when PCE led to losing the current element being analyzed
2016.06 fixed deadlock with inconsistent lock order
2016.06 fixed Fork-join-pool starvation
2016.06 fixed bug when PCE corrupted classBeingProcessed
2018.02 fixed bug when waiting for all cooperationg threads to finish hanged

Moral corner.
Never again the concurrent cooperating threads. It's too crazy complex to do right in Java.
