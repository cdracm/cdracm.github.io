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

To make it work faster, we needed to think smarter.
