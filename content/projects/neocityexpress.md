+++
author = "Isadora"
title = "Neo City Express"
date = "2023-05-01"
description = "My game entry for the Ludum Dare 53."
tags = [
    "extension",
    "projects"
]
+++

Neo City Express was my Ludum Dare 53 entry.

<!--more-->

I was so excited to do my very first Ludum Dare! I got together with [Pedro](http://saint11.org/), Davey and [Ryan](https://www.dualryan.com/) for 72 hours and we ended up with this game about texting and driving and uh... _stuff_.

The game can be played on 🎮 [itch.io](https://saint11.itch.io/neo-city-express). You can also play the original Ludum Dare game here: 🕹️ [Ludum Dare](https://ldjam.com/events/ludum-dare/53/neo-city-express). _And_ you can build it yourself with the 📝 [source](https://github.com/isadorasophia/neocityexpress).

<center>
<img src="/images/projects/neocityexpress.png" alt="Of course he's not texting and driving in this picture">
</center>
<br>

This it was our first official "release" of a game made with my engine, [Murder](https://github.com/isadorasophia/murder)! Pedro and I wanted to do this for a while and, after working on the engine for a year or so, we felt we were finally ready for a game jam.

It allowed us to prove not only that the engine works, but that it also allows us to code _fast_.

Have I mentioned how much I learned from Pedro? He's so damn good with game jams! As soon as the jam started, we made sure we focused on brainstorming (no coding) in these first hours. By saturday 1pm-ish, we had something more or less like this:

<p>
<center>
<img src="/images/projects/neocityexpress_prototype2.gif" alt="Gameplay was starting to take shape!">
</center>
</p>

## First 24 hours: game loop

For the first day, we focused on having a game loop "ready" - even with a menu! The idea was to have something mechanically ready by the end of the day. It didn't have to be good, as long as it was something. This helped us making sure we were keeping things in scope and we would have time to finish the game.

Something that I was particularly proud is how the game has auto-save implemented. This is built-in of the engine, which already handles all the serialization and consistency, so it wasn't a lot of work to quickly get something working during the jam.

## Day two: features!!!
<p>
<center>
<img src="/images/projects/neocityexpress_prototype3.gif" alt="Cellphone was now working.">
</center>
</p>

Ryan, who did all the music and sounds, was able to get onboarded by Sunday! It was Very Interesting because I never tested the engine on Mac before, so I just interrupted the jam to quickly update the MonoGame SDL assemblies to work on the latest Mac Version, so he could actually run the game. **BUT** it worked.

Davey started adding all the text and dialogue to the game, as well as helping Pedro to balance and design the levels.

I mostly worked on having a system for: adding dialogues, track how these dialogues are triggered throughout the gameplay, track the distance until you complete a delivery, enable dialogues to interrupt each other, figure out how to make blocking and non-blocking events, implement the actual typing in the keyboard and anything else along the way.

We didn't even implement the drugs system (lol) or Tinder yet. I thought the Tinder idea was so funny that I made sure we had something implemented to play with by Sunday.

The most important thing is that we had everything ready and all we had left to do was add as much content as we could on the next day.

## Day three: chaos

At this point, I accepted that I was going to take my day off at work (I was still in denial). We ported all the fmod sounds to the game, which Pedro and I didn't fully understand what it meant at the time.

<p>
<center>
<img src="/images/projects/neocityexpress_editor.png" alt="Cellphone was now working.">
</center>
</p>

I was pretty happy with our editor support for designing each of the days. We spent most of the day doing _a lot_ of playtesting and just balancing the game. At this point, I was starting to lose it and spend hours on bugs that I could quickly fix in 5 minutes the next day.

<p>
<center>
<img src="/images/projects/neocityexpress_final.gif" alt="Cellphone was now working.">
</center>
</p>

## End game
We were all really happy with the game, we even got a voice actor to do one of the endings! Pedro and I kept laughing (exhausted, at 3am) at all the things that Davey wrote throughout the game. I kept laughing at all the drawings that Pedro did, which were wonderful.

...And skip one month later, we actually got first place!!!! This was the insane part for me and I was so proud of all the work we did. I didn't think it was possible to do a game in ECS in such a short time, but we actually did it.

<p>
<center>
<img src="/images/projects/neocityexpress_results.png" alt="Cellphone was now working.">
<br>
<i>These were the final results for our game!</i>
</center>
</p>

Overall, I learned A Lot from this. I guess I could take a couple of takeaways on the whole thing:

1. Don't do an ECS system for tweens. It's not worth it. Just use a state machine.
2. Sleep and wake up early! Except on the last day. Anything goes on the last day.
3. Eat healthy and make sure you have all the meals throughout the jam. Except on the last day, see item above.
4. Each minute is precious. We started to measure tasks in minutes. It got to a point that we actually said stuff like: "I can do this, but it will take me 20 minutes, so it's not worth it".
5. I actually did copy and paste something so I didn't spend 1 minute doing a method for it, see item above.
