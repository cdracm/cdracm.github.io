---
layout: post
title: "The Continuation of Extension of Annotation Migration"
hidden: true
comments: true
---
The hardest part in my work is people.

### An Annotation API migration story

A long time ago, I refactored the annotation subsystem to be more responsive. 
(Annotators are these little code nuggets that support various language features and highlight some problems).
This was a substantial refactoring involving the removal of old methods and the introduction of new classes.
I realized that migrating to this new API was not straightforward, so I deprecated the old methods, ported as much code as I could, 
and asked the authors of the remaining annotators to port their code when they had the chance.

- A half year passed, and the deprecate methods were still in use.
  I realized they have more important tasks to do, so I inserted the log message inside the deprecated method:

  `LOG.info('please port to the new API')`
- An another half year passed, the deprecated methods were still in use.
  I thought they must have needed more time to figure out the port, so I upgraded the message to a warning to increase visibility:

  `LOG.warn('please port to the new API')`
- Yet another half year passed, the deprecated methods were still in use.
  I figured they needed more time to complete the port. So I added more urgent warning:

  `DeprecatedUsage.report('please port to the new API')`
- A half year more passed, and the deprecated methods were still there.
  I finally understood they needed more time. I added the stacktrace to the warning mesage to help them.
  
- Some halves of a year passed, and, eventually, all annotators in our codebase were migrated to the new API.

  The last message in the log was
```2022-10-27 23:21:20,759 [ 946049]   WARN - #c.i.c.d.i.AnnotationHolderImpl -
CLion developers promised to fix their annotator 0 centuries 3 years 8 months 21 days ago
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

### The Right to be Forgotten. Article 17. 

Speaking of deprecation. 
The eldest and wisest deprecation notice I could find in our codebase - which still lives happily today - was made twenty-something years ago with this awesome commit:
![deprecated](/assets/annotation/deprecated.png)
