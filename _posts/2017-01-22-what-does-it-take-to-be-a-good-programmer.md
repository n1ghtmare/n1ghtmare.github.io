---
layout: post
title: "What does it take to be a good programmer?"
description: "Sharing some experience"
date: 2017-01-22
tags: [lifestyle-programming]
comments: false
share: true
---

I’ve been writing code now for almost 10 years, I’ve worked on different projects in size, with different teams (some solo projects) on various platforms. Although I still feel I have a lot to learn and I do suffer from an [impostor syndrome](http://www.hanselman.com/blog/ImAPhonyAreYou.aspx), I think I’ve learned a few things.

I had a discussion recently with a fellow developer on what does it mean to be a good programmer, and what is “good code”.

Well, this is not going to be a revelation to anyone, and it might even incite a few "duh" responses, or might even be different than the popular opinion, but it’s simply my thoughts written down based on my personal experience in the hopes that someone might find them useful.

## Be Humble

Even when you think you know everything, or you think you’re hot shit/"rock star"/"ninja" (or whatever they call it these days) – I guarantee you you’re not (or at least not as much as you might think). I believe in order to work well with others or just being good programmer in general – you need to be humble and realize the fact that don’t know everything. That is of course not to say that you need to blindly follow what others in the industry say, or not to argue/defend your opinion, no far from it, but  be willing to hear people out, be kind and realize you *might* be wrong. Being wrong, or rather realizing that you’re wrong is a good thing, because this is the moment you learn new things.

## Write Good Code

Ok, I know – gee thanks, great advice. No shit, Sherlock! Also what is good code anyway, right? Well, unfortunately, this is a debate, and it’s also something that comes with experience, you develop a feel for it, but I think we can all agree on the basics. Be consistent with the coding style you and your team have agreed on. Even if it’s a solo project, be consistent, be obsessive about it. Stop and think about naming things. Go back to code you’ve written earlier and see if you can improve it. Don’t write write smart one-liners that no one can understand, or if you do – leave a comment, explain your intent. I have a rule of thumb – always try to write the code in a way that is easy to read and understand. Try to keep your functions/methods short. Same goes for classes. I promise you that a class with 7000 lines of code is almost always ***bad***.

Some of my favorite principles that I try to follow are DRY (Don’t Repeat Yourself) and YAGNI (You Ain’t Gonna Need It). I believe these principles push you to write cleaner and better structured code. Of course like with everything else in our industry, you need to be pragmatic. Don’t take things too far, and **never be dogmatic about anything**.

YAGNI helps a lot with "analysis-paralysis". I think we’ve all been there, where you over-architect something so much that you can’t move forward. Be balanced, anticipate things, but move forward, write code, don’t be afraid to change things around, don’t try to write every-single thing before you even need it – requirements change, platforms change, you might be wasting your time.

DRY helps in writing code that is more maintainable. When you do a change to your codebase you don’t want to change the same code in 15 different places, or worse change it in 14 and forget 1 that causes a bug 3 months down the line. Having said that (and I hate saying that because it makes me feel like one of those cliched "consultants") **it depends**, *sometimes* it’s ok to have duplicated code. If the cost for removing the duplication leads to an overly complex abstraction that is not worth it, it’s fine to leave the code the way it is.

Write unit tests, they could prove to be invaluable, especially when you introduce changes to your codebase. When you discover a bug or an edge case, write a unit test for it, trust me you will thank your past-self later at some point. Sometimes I use TDD for certain components of the system I’m working on. I believe certain types of problems lend themselves nicely to Test-Driven Design, others don’t. Again, try to be balanced, don’t be dogmatic – be pragmatic. Always try to use the best tools for the job.

Try to leave the codebase cleaner than you found it, remove dead code, or the thing I really loathe commented-out code, you have source-control for that, just remove it.

## Move Faster

I know this has been said many times before, but it’s just now after so many years *really* understood it. I’ve developed a new technique (methodology if you will) of development. I divide and conquer and get things done. Let me clarify.

I think about the problem (give it some thought, but don’t dwell too much on it), then I divide said problem into sub-problems, make them work, make the larger problem work, integrate the rest of the codebase (if needed) – *then and only then* go back and clean-up/refactor/optimize/goldplate your code. I guess this might be something that is pretty obvious, but it took me many years to reach this mindset. I use to spend so much time on the smallest details upfront that I (in many cases) lose the big picture. I would write things that I end up not needing, or I would need them but in a different form and I would discover this only when I put the pieces together. I wasn’t as efficient as I am now. Don’t be afraid to go back and change things around. Try to move fast! Of course that is not to say you should be sloppy and throw shit together. Also by “fast" I don’t mean rush, like I said before think about what you’re solving. Be balanced, be pragmatic.