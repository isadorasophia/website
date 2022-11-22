+++
author = "Isadora"
title = "Your very own ringtone!"
date = "2022-09-29"
description = "Visual Studio extension to add sounds to IDE commands."
tags = [
    "extension",
    "projects"
]
+++

A Visual Studio extension to add sounds to Visual Studio called "Your very own ringtone!".

<!--more-->

You can check the [source code here](https://github.com/isadorasophia/your-very-own-ringtone) or download it through the [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=isainstars.yourveryownringtone). 

This was a very fun project, I basically add hooks for all sorts of commands within Visual Studio, which you can check at [VSListener.cs](https://github.com/isadorasophia/your-very-own-ringtone/blob/main/src/VSListener.cs). I also learned a bit about NAudio to support multiple audio formats and sample rates. I love doing tools that I can use myself!

![](/images/projects/your-very-own-ringtone.png)

<video width="100%" controls="yes">
  <source src="/images/projects/vs-sounds-demo.mp4" type="video/mp4">
</video>