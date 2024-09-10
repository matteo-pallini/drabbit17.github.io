---
layout: posts
comments: true
author_profile: true
title: Philosofies of Software Design
excerpt: Thoughts on Ousterhout's "Philosophy of Software Design" and Raymond's "The Art of Unix Programming"
---

**TL;DR**


### Who needs Philosophy? or Art?
This blog post is not a summary of the two books or a comparison. It's just a collection on thoughts on the two perspectives and ultimately a recommendation to read them.

Most software engineering teaching material seems to be focused on showing you how to solve specific problems, rather that helping you to develop a general understanding on why a solution is appropriate. it will help with how to solve a specific problem with a technology/language, but you may still have no clue on the why the solution makes sense and then go on tackling a different problem by re-using the same solution with some tweacks, or adding patches on top of it. Some other books will do a slightly better job by giving you general heuristics or rules, Clean Code by Martin is a good example of this, then depending on your experience you may consider it the gospel, or take some of it, or reject it completely. 

There is not many books that will give you some principles you can use when organising your code given a specific context/problem. Ousterhout's "Philosophy of Software Design" and Raymond's "The Art of Unix Programming" are two exceptions in this regard. They deliberately focus on giving you high-level principles that will help you make design decisions, such as how large interfaces should be, how many layers to have (few), etc. Whether someone should agree with the principles is up for debate, but at least it will force the reader to think on what should be at the core of a decision. If you agree with the ideas provided you will now have some way to justify the organisation of your code next time you have a debate, if you disagree you can still leverage the argument provided to position yourself on the opposite of the spectrum. The section below on Ousterhout's narrow interfaces VS Raymond's transparent ones should give a better idea of what I am referring to here. The two positions seem at opposite ends of the spectrum, but each one provides arguments for them, based on what they think is the better design philosophy. Up to you to pick one, reading the book will make it easier for you to articulate your preference.

### Narrow Interfaces VS Transparent ones

### What else besides these two books
Another reading that really helped me in terms of code design are the first two chapters of the Gang of Four book (one of them is particularly good). I think OOP only started really clicking to me after having read these two chapters (practicing the SOLID principles also helped).

### What if you don't like reading books
Ultimately the point of internalising principles to support your design choices is to make better (or more informed) choices. Other non-books approaches that could get you there: 
1. If you are lucky enough you will join a company with experienced and good developers who will not only help you to develop a taste, but also to articulate it (ie this is good because X, Y and Z). 
2. If you are less lucky, you can learn quite a bit by reading code from good open source repositories.
3. You can also read RFCs for language features and the related discussions. Although, I think the ROI from open source is much smaller, because you have to get lucky to find good codebases or good contributors. Also, you will need to read a lot, and some of it may be quite hard (I am thinking about RFCs specifically here). Reading open source code with a good reputation would help here (eg Redis codebase)
4. Trying code competition and reading others' solutions (Advent of code or Code Katas). Although also here you are going to get low ROI probably.

Time taken to write post: 6 hours
