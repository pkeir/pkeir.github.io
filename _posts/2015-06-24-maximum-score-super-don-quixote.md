---
layout:   post
title:    "The Maximum Score in Super Don Quix-ote"
category: videogames
tags:
 - C++
 - computer vision
 - retrogaming
 - videogames
---

![](/assets/images/000000_bats_01_right.jpg)

During the golden era of arcade videogames, a remarkable game appeared in one of my beloved local video arcades: Super Don Quix-ote. A laserdisc game, the graphics and gameplay of Super Don Quix-ote basically offered an interactive cartoon; a stark contrast to 1984 raster-graphic contemporaries such as Kung-Fu Master, Pac-Land or Marble Madness.

Another laserdisc game, Dragon's Lair, was released one year prior to Super Don Quix-ote; in 1983. Both used a similar gameplay mechanism, requiring the player to respond to numerous moments of fleeting danger, in a timely fashion. In contrast to Dragon's Lair, Super Don Quix-ote was kind enough to display a visual cue at such moments; an icon, inviting either **up**, **down**, **left**, **right** or **button** input. Years later, Shenmue used a variation on this mechanism, then dubbed Quick Time Events (QTEs).

I must admit I was pretty handy at Super Don Quix-ote back then; once I was even paid to play it to completion. It seemed obtaining the very best score was conceptually a simple matter of completing the game without losing a life; and this I could do. One day however I was surprised to find a stranger in the arcade; a stranger with a higher score than expected; given his loss of life. The outsider was good enough to explain.

A selection of the QTEs of Super Don Quix-ote would accept an input contrary to the on-screen icon. Choosing such *alternative* moves would ultimately result in a better score. The visitor showed me all that he knew, and in the weeks that followed, I found a couple more. Soon I was able to complete the game using all the QTE alternatives; again without losing a life.

A few years ago I was delighted to learn of the [Daphne](http://www.daphne-emu.com) emulator: the First Ever Multiple Arcade Laserdisc Emulator; led by Matt Ownby. Its support for Super Don Quix-ote was the icing on the cake. I had always wondered if I did find *all* the alternative QTE moves, and noted there was no discussion of these online. Around this time I had the idea that computer vision software engineering methods could be used to automatically play Super Don Quix-ote, and then explore all possible QTE responses.

![](/assets/images/all_sdq_alternative_moves.jpg)

Well, to keep a long story short, I recently got round to it. Using a C++ program I developed (the source code is [here](https://bitbucket.org/pgk/sdq_explorer), running external to the Daphne emulator, I exhaustively tried all possible QTE responses while also tracking the score; using hand-rolled computer vision routines. So, there are 14 alternative moves. Contrary to the icons shown above, a player can use **Button; Left; Button; Left; Left; Button; Down; Left; Left; Right; Left; Button; Left & Button** (lexicographical ordering) and will be rewarded with 10,000 bonus points each time.

Of course, suitably armed, I then wanted to attempt the maximum high score. Surprisingly, this was the hardest part of the project. I set a camera and tripod up in my living room, and set aside around an hour a night for what grew to a period of a couple of months. The maximum score in Super Don Quix-ote is 776500. The video of me getting the score is below (and [here](https://youtu.be/ZpzWhfh92F4); a video of the sdq_explorer program doing the same is [here](https://youtu.be/5lKqUU2qiwU). I'll be presenting a paper on this in more detail at the [xCoAx](http://xcoax.org) Conference at the [CCA](http://www.cca-glasgow.com) in Glasgow this coming Thursday 25th June (2014).

[![Video of perfect SDQ playthrough by the author](https://img.youtube.com/vi/ZpzWhfh92F4/0.jpg)](https://www.youtube.com/watch?v=ZpzWhfh92F4)
