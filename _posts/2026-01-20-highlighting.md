---
layout: post
title: "Efficiency by Thousand Cuts"
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
   When the highlighting is restarted, only PSI elements visible onscreen got to be processed first, then all the remaining elements.
   This way the visible highlighters are shown faster.
   However, this obvious idea is not so simple really.
   Again, people are to blame.
   Turned out, there are quite a few inspections written in so convoluted way that when they process some PsiElement1, they show the result highlighters for some completly unrelated PsiElement2 in completely opposite screen corner.
   So running these inspections for visible elements make them do some unnecessary work on other invisible elements.
   Inspections, being a piece of random imperative code, are free to do some crazy stuff, and, according to mister Murphy, are bound to do some crazy stuff.
   
1. Fertile elements first
   Battle for reducing latency between typing in an editor and showing highlighters onscreen led to realization:
   Some inspections in some situations like some PSI elements in some files more than other.
   For example, some annoying inspecton could complain about methods being too negatively named.
   This inspection is naturally leaning on PSI elements being method identifiers, and ignore everything else.
   So however many whitespaces or curly braces you feed this inspection, you'll get nothing in return.
   Until you pass PsiIdentifier - then it has a chance to return a warning "This identifier is too negative, come to your senses".
   What if we keep track of these elements - I call them "fertile PSI elements" - and feed them first, to reduce the latency?
   Of course, it only works when PSI elements stay more or less stable on typing.
   Otherwise, when almost all PSI gets rebuilt on every key press, all accumulated knowledge on PSI element fertility is lost.
   This is precisely what happens in not-so-well written languages, where incremental reparse and other optimizations are not yet implemented.
    
1. Change scope locality detectors
   This optimization tries to compute the smallest set of PSI elements to restart the highlighting for.
   Some languages have lexical structure scoped, so that the change inside one scope does not affect the other scope.
   For example, in Java when something change inside one method code block it should not affect the other method code block.
   So we can avoid (expensive) re-highlighting that other method altogether.
   
1. Remove obsolete highlighters faster.
   This was a big optimization which required rewriting the whole highlighting engine.
   Before this optimization, the following algortighm was used:
   - run all inspections, see what highlighters they generated
   - compare all generated highlighters with all highlighters already exising in the file
   - remove the obsolete highlighters (i.e., the ones that are no longer generated)
   Please note that the interval between running the inspection and removing the obsolete highlighter it no longer generated is huge.
   It's essentially the total time of all inspections being ran for this file.
   This is a very long time.
   Can we do better?
   The algorithm above doesn't know that inspections generate highlighters when being fed with some PSI element.
   If we remember which PSI element caused generation of which highlighters, we'll be able to remove the obsolete highlighter
   as soon as the inspection doesn't generate it when processed this particular PSI element the second time.
   Again, this optimization is heavily dependent on PSI reparse preserving identity of unrelated PSI elements.

   What went wrong with this optimization? A couple things:
   - Turned out that PSI reparse is not that stable, even in mature languages supported since Old Ages.
   - People really are eager to write stupid things.
     I found several inspections went bonkers when called with the same PSI elements.
     Some returned a highlighter when was called with PsiIdentifier. The second time it returned this highlighter when was called with PsiClass.
     And so on, it alternated PSI elements as if unsure which one to pick.
     What's wrong with you people?

1. Lazy quickfixes
   Some inspections have to do heavy work to compute and present fixes/improvements to the code.
   For example, "unresolved symbol" inspection highlights incorrect symbols in the text and suggests to fix them, either by adding imports, qualifiers or renaming them.
   This can be very expensive because for example finding the correct "assert" methods among dozens is hard. 
   This optimization tries to defer computing these expensive quick fixes when they are not needed.
   What went wrong:
   - Spawning and canceling all these additional background calculations proved to be complicated
   
1. API for discovering PSI elements only once for all inspections/annotators
1. Immediate highlighter creation
1. smart parallelization 
