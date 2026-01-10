---
layout: post
title: "How Many Optimizations"
hidden: true
comments: true
---

It's a list of all optimizations in the highligting I can recall. 
It's just for the sake of completeness, for me when I need to implement something that horrendous in the future again.

Highlighting is a process of analyzing the current editor text and highlighting all the problems/suggestions/things that IDE wants to highlight.
Highlighting is divided into small granular pieces of code, called "inspection" or "annotator" each of which analyzes its own thing.
For example, there are inspections that highlight too long method names, too convoluted method body, unused variables, inconsistent REST endpoints, spelling mistakes or inefficient API usages.
Each of these inspections need to be run, all problems it found highlighted, and all problems fixed need to be de-highlighted.

Highlighting needs to be: fast, responsive and has low-latency
And thus, never-ending battle for performance of all these inspections has begun!

1. Make inspections/annotators run in parallel.
   Since each inspection is a piece of code analyzing its own problem indepedently of others, running alll them in parallel is an obvious optimization.
   And yet it never fails to amaze me how many requests I'm getting that trying to restrict this parallelism.
   "Can I run my 'inspection X' after 'inspection Q' please?"
   "No no no, my inspection can't be run in parallel because it requires stopping the whole world and let it watch holding its breath how my code is run!"
   "Fine, you can run my inspection in parallel. But only if it computed BronculatorZ value in advance. Otherwise, pause everything for ten minutes while BronculatorSupplier completed computation"
   
1. Make inspections/annotators process every PSI element in parallel.
   The current file opened in the IDE is parsed by its corresponding language and a number of PSI elements are returned.
   E.g. the java file `class C { int field; }` is parsed by Java language parser and a handful of PSI element objects are returned, like
   PsiClass(name="C"), PsiField(name="field"), PsiTypeAnnotation("int"), PsiWhiteSpace(text=" ") etc.
   Each of these objects are handled by an inspection, one by one, and it retruns the problem, if there is any around this element.
   It would be faster to do this in parallel, right?
   
   It proved to be so much harder than the previous optimization.
   Turned out, people hate parallel code.
   It's almost impossible to force them to write a piece of code which is
   - thread safe (can be safely called from several threads simultaeously)
   - reentrant (can be called from any context, even recursively or from other threads)
   - efficient (only analyze this freaking white space, not iterate all files on disk and upload each file to the remote LSP server in India saturating all available CPUs and blocking all of them immediately after that)
   So many existing inspections were just not ready for this parallel execution.

1. Analyze the visible part of the file first.
1. Fertile elements first
1. Change scope locality detectors
1. Remove obsolete highlighters faster
1. API for discovering PSI elements only once for all inspections/annotators
1. Lazy quickfixes
