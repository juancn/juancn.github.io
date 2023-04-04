---
layout: post
title:  "Paper Programming"
date:   2011-04-04 12:46:00 -0300
categories: post
---

When I was a kid, all I wanted was a computer. Finally when I was twelve I made a bargain with my dad. I would give up the graduation trip in exchange for a Commodore 64 (graduation trips are customary in Argentina when you finish primary and secondary school).

![A Commodore 64](/images/2011-04-04-paper-programming/c64.jpg)

We bought a "Segundamano" (lit. second hand) magazine and found a used one for U$S 200. My dad contacted the seller and we went to pick it up.

You have to keep in mind that this was 1989 and the social and economic landscape in [Argentina was a mess](http://en.wikipedia.org/wiki/1989_riots_in_Argentina). That year the [inflation rate was over 3000%](http://www.wolframalpha.com/input/?i=argentina+inflation+1989) (it is not a typo) and those 200 dollars was a lot of money, so my dad really made an effort.

If you are still paying attention, you might have noticed that I never mentioned a [Datasette](http://en.wikipedia.org/wiki/Commodore_Datasette) nor a disk drive. All I got was a Commodore 64, plain and simple. But this was not going to stop me.

After we got it, my dad was concerned that I might get too obsessed with the computer, so he placed some additional constraints on when I could use it (the fact that we had only one TV might have also been a factor in his decision). I was allowed to use it only on Saturday mornings before noon.

In retrospective, I think that this two factors made me a better programmer.

At that time I had some experience programming in Logo mostly. It was a Logo dialect in Spanish that we used at school ("HAL Logo en Español", do not confuse with HAL Laboratories). We had a Commodore-128 laboratory at school (about eight machines with disk drives and monitors). I started learning Logo when I was 8, by the time I was twelve I could program a bit of BASIC also, but not much since literature was scarce.

One great thing about the Commodore 64 was that it came with BASIC, but most importantly, it came with a manual! The Commodore's Basic manual was my first programming book.

What happened was that I was forced to work as if I had punched cards. I would spend most of my spare time during the week writing programs in a notebook I had. Thinking about ways to solve problems and reading and re-reading the C64 manual.

On saturday mornings I would wake up at 6 am and hook-up the computer to the TV and start typing whatever program I was working on that week. Run it and debug it, improve it and around noon my mom would start reminding me that time was up. So at that point I began listing the BASIC source code and copying it back to my notebook.

It was during this time that I rediscovered things like an optimized Bubble-sort, although I didn't know its name then (and I wouldn't learn it for many more years). I still vividly remember the moment. It was one afternoon that I was trying to figure out a way to sort an array, so I started playing with a [deck of cards](http://en.wikipedia.org/wiki/Baraja_(playing_cards)). I finally figured out that I could do several passes of comparison and exchanges on adjacent cards. And if I did exactly as many passes as the number of elements the array would be sorted. I also noticed that the largest element would be at the end of the array after the first pass, and the second largest would be in place after the second pass and so on. This allowed me to save about half the number of comparisons.

The most valuable thing that I learned during this time is that when it comes to programming, thinking hard about the problem at hand before doing anything pays off, big time!

I eventually was able to save enough for a Datasette (it costed 40 dollars, my dad payed half of it) and I was able to build much larger programs.

The biggest one I wrote was an interpreter with a multitasking runtime. It was very basic. I had no notion of a lexer, let alone a parser. Commands were composed of a single letter (the first one of the line) and several arguments. You could declare procedures with names and call them. It was like pidgin-basic (since my exposure was to basic) but the procedure structure resembled Logo.

Programs were stored in memory in several arrays. There was a main one that contained statements, and couple to hold procedure names and offsets into the main array.

The runtime divided the screen in 4 areas (this was a 40x25 text screen, so not much room was left for each), and each window could run a separate program. The runtime would do a round robin on each running program, executing a single statement of each on each pass. For this it had to maintain a separate program counter and variables for each. I even planned to add windowing calls to create custom windows but I never got to finish it.

It was at this time that I also got interested in electronics, so I built a a few contraptions controlled by the C64, but that's a tale for another post.