---
layout: post
title: "-"
hidden: true
comments: true
---
The hardest part in my work is people.
# Intention description test

A long time ago, I wrote a test which checks that every intention has a description.
The IDE I work on has a number of quick fixes, or "intentions" that help solve an issue at hand or just do some useful.
E.g., to introduce a variable when the user's caret is on an unknown symbol, or delete an unused method, or just invert the condition inside an `if()` statement.
I figured it would be helpful to supply a description for every quick fix so the user would understand what was happening.

So I 
- added the mising descriptions and wrote the test which checked **`assert (description != null)`**.
- Some time after I noticed a couple of new quick fixes without descriptions.

  The test passed however, they all did have a description, but the empty one.
  So I added missing descriptions and improved the test with **`assert (description.isNotEmpty())`**.
- A week after I noticed yet another quick fix lacking the description.
  The test passes, and the description was non-empty all right. It contained the single space character.
  I added missing description and improved the test with **`assert (description.trim().isNotEmpty())`**.
- A short time later I noticed yet another quick fix lacking meaningful description.
  The test passed brilliantly; the description was perfect. It’s just that it consisted of the lonely symbol "x".
  I added missing description and modified the test to **`assert (description.trim().length() > 6)`**,
- and since then I keep ignoring all subsequent quick fixes with descriptions `"description"`, `"lorem ipsum"` and `"blahblah"` out of pure desperation.

# An Annotation API migration

A long time ago, I refactored the annotation subsystem to be more responsive. (Annotators are little code nuggets that support various language features and highlight some problems).
This was a substantial refactoring involving the removal of old methods and the introduction of new classes.
I realized that migrating to this new API was not straightforward, so I deprecated the old methods, ported as much code as I could, and asked the authors of the remaining annotators to port their code when they had the chance

- One year passed, and the deprecate methods were still in use.
  I realized they have more important tasks to do, so I inserted the log message inside the deprecated method:

  `LOG.info('please port to the new API')`
- One more year passed, the deprecated methods were still in use.
  I thought they must have needed more time to figure out the port, so I upgraded the message to a warning to increase visibility:

  `LOG.warn('please port to the new API')`
- One more year passed, the deprecated methods were still in use.
  I figured they needed more time to complete the port. So I added more urgent warning:

  `DeprecatedUsage.report('please port to the new API')`
- One more year passed and the deprecated methods were still there.
  I finally understood they needed more time. I added the stacktrace to the warning mesage to help.
  
- Eventually, all annotators in our codebase were migrated to the new API.

  The last message in the log was
```2022-10-27 23:21:20,759 [ 946049]   WARN - #c.i.c.d.i.AnnotationHolderImpl -
CLion developers promised to fix their annotator 0 centuries 5 years 6 months 2 days ago
com.intellij.diagnostic.PluginException: 'AnnotationHolder.createErrorAnnotation()' method
(the call to which was found in class com.jetbrains.cidr.lang.daemon.OCAnnotator) is slow, non-incremental and thus can cause unexpected behaviour (e.g. annoying blinking),
is deprecated and will be removed soon. Please use `newAnnotation(...).create()` instead [Plugin: com.intellij.cidr.lang]
	at com.intellij.diagnostic.PluginProblemReporterImpl.createPluginExceptionByClass(PluginProblemReporterImpl.java:23)
	at com.intellij.diagnostic.PluginException.createByClass(PluginException.java:92)
	at com.intellij.codeInsight.daemon.impl.AnnotationHolderImpl.doCreateAnnotation(AnnotationHolderImpl.java:189)
	at com.intellij.codeInsight.daemon.impl.AnnotationHolderImpl.createErrorAnnotation(AnnotationHolderImpl.java:79)
	at com.jetbrains.cidr.lang.daemon.OCAnnotator.addErrorAnnotation(OCAnnotator.java:205)
	at com.jetbrains.cidr.lang.daemon.OCAnnotator.addErrorAnnotation(OCAnnotator.java:163)
	at com.jetbrains.cidr.lang.daemon.OCAnnotator.addErrorAnnotation(OCAnnotator.java:173)
	at com.jetbrains.cidr.lang.daemon.OCAnnotator.addErrorAnnotation(OCAnnotator.java:136)
```

# The Right to be Forgotten. Article 17. 
Speaking of deprecation. 
The eldest and wisest deprecation notice I could find in our codebase - which still lives happily today - was made twenty-something years ago with this awesome commit:
![deprecated](/assets/people/deprecated.png)
