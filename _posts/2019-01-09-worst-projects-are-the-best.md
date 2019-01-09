---
layout: post
title: "Worst Projects Are The Best"
description: "Looking back at the worst project I've worked on and the lessons I've learned as a result"
date: 2019-01-09
tags: [programming, projects]
comments: false
share: true
---

It's one of those days at work where I had to deal with a horrible API that's being held together by duct tape and prayers. At first I was really frustrated, but lately I've been trying to take a more stoic approach to things, so I tried to look at it in a more positive way.

I realized that through the years I've learned the most from the shittiest systems. I mean sure, in most cases it was more of an "anti-example" rather than a concrete lesson. It was more of - "In the future don't do this or you'll end up with shite! P.S. - Remember this pain!".

Now, don't get me wrong, of course I've learned plenty from well-designed software, however I can't help but think that the bad ones were a lot more visceral and made me appreciate the good ones a whole lot more. I understand that the definition for shitty software differs from one person to the next, I also find it interesting that throughout the years my view on this matter changed, however some things in software design/development are just universally shitty.

As I reflect on past projects I start to wonder. Seriously, what is the shittiest system I worked on? In my case one particular project comes to mind.


### The Big Brown Wonder

Years ago, I was assigned to work on a well-established "flagship"/"core" web app that was kinda doing ... well - everything. Honestly, it started as some sort of a correspondence system and over the course of 10 or so years grew to a monster that everyone was afraid of. Everything about it was horrible, from the absolutely crazy (and ever-growing) scope, down to the technical details. On my first day with the system, I was sitting with one of developers when he received an "urgent" requirement to add a field on a form. He opened a stored procedure that had ~80,000 LOC did some voodoo search and found the HTML that needed to be worked on. Yes, HTML - in a stored procedure. Actually, that was the whole system, well that and 3-4 more stored procedures, the smallest of which was ~50,000 LOC (!!!!!!). The developer looked at me at this moment with a "yeah I know, what can you do" expression on his face and proceeded with adding the field. At this point I wasn't really sure if I should be impressed or disgusted. Naturally all of the developers on the team hated it and the legends of how this system came to be were lost throughout the years. Some attempts were made for a re-write at one point or another, but for one reason or another none were successful.

I had to marvel at this wonder for a bit and then I wanted to dig a bit deeper and see where this goes.



It was a web app hosted on Apache using something called (if I remember correctly) `mod_plsql` with an Oracle DB backend, also I'm not sure "hosted" is the exact word since almost everything, including the application itself was stored in the DB (not just the data).

It had (at least) 4 different configuration sources - an XML config file, config in DB, config on a flat text file, config in a web service (SOAP). The config files were structured completely different although many of the key/values were duplicated and it wasn't clear when one configuration source is used over another (sometimes it's the XML config, sometimes it's the one from the DB). The code itself was even worse. Although I don't remember details, I remember one particular instance that I wanted to "refactor" - something in the lines of:



```
-- performance, don't touch ! 
IF {some-condition} THEN
    {do a thing}
ELSE
    {do the *exact same thing!!!*}
END IF;
```



Every single "page" of the system was located in the aforementioned giant stored procedure (or one of its 3-4 siblings), with absolutely no structure in place. There wasn't really a concept of a "page layout" or a "master page", which meant that if a developer wanted to, for example add a link in the footer of a page he had to modify something like 50 different locations in the stored procedure! Fun times.

I forgot to mention - there was no documentation, there were no unit tests, no integration tests, no tests of any sort. Oh, and ... no source control. I know people like to shit on CVS or SVN (which was the hotness at the time), but I would've killed for any form of source control. Making any change to the system almost always guaranteed to break something elsewhere. Working with this beast was a huge challenge and not a pleasant experience to say the least. Due to the way the management worked, the team size and all the previous decisions that lead to this monstrosity, the project was beyond "fixing". You had to accept it for what it is, do your best, and just go with the flow. The best you could do was clean up some of the code, add a helpful comment here and there and move on with your life.



I was later assigned to another project and after a while I completely changed my job. I do wonder sometimes what happened to the system. Needless to say, the experience has thought me a lot of things. Mainly, how *not* to structure a web application, the value of tests and source control and the importance of documentation . So in a way, I'm really thankful for the opportunity to work on this software and all the other (less) shitty systems before or after it.