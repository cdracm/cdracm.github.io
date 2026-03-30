---
layout: post
title: "Intention to Lorem-Ipsum, Sir!"
hidden: true
published: false
comments: true
---
The hardest part in my work is people.

### Intention description test story

A long time ago, I wrote a test which checked that every *intention action* had a description.
The IDE I work on has a number of quick fixes, or "*intentions*" that help solve an issue at hand or just do something useful.
E.g., introduces a variable when the user's caret is on an unknown symbol, or deletes an unused method, or just inverts the condition inside an `if()` statement.
I figured it would be helpful to supply a description for every such quick fix so the user would understand what was happening.

So I went and
- added the missing descriptions and wrote the test which checked
  
  **`assert (description != null)`**.
  
- Some time after I noticed a couple of new quick fixes without descriptions.

  The test passed however, because they all did have a description, but the empty one.
  So I added missing descriptions and improved the test with

  **`assert (description.isNotEmpty())`**.
  
- A week after, I noticed yet another quick fix lacking the description.
  The test passed, and the description was non-empty all right. It contained the single space character.
  I added the missing description and improved the test with

  **`assert (description.trim().isNotEmpty())`**.
  
- A short time later I noticed yet another quick fix lacking a meaningful description.
  The test passed brilliantly; the description was perfect. It’s just that it consisted of the lonely symbol "x".
  I added missing description and modified the test to

  **`assert (description.trim().length() > 6)`**,
  
- and since then I kept ignoring all subsequent descriptions `"description"`, `"lorem ipsum"` and `"blahblah"` out of pure desperation.

