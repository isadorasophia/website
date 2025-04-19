+++
author = "Isadora"
title = "Visual Studio Toolbox - Why is async hard to debug?"
date = "2021-06-10"
description = "Series explaining how async works and how to debug in Visual Studio."
tags = [
    "series",
    "youtube",
]
+++

Series of videos explaining about async debugging in C#.

<!--more-->

This was a talk recorded with Channel 9, from Visual Studio. We divided the talk in three parts so we could go in-depth more easily with the other topics.

The source code for this talk is available [here](https://github.com/isadorasophia/ReadMySongs).

## Part 1
In this first part, I explain what happens under the covers of an "async" and "await" in C#. We start from scratch with a synchronous application and make it asynchronous.

{{< youtube 7POKQgdkrA4 >}}

## Part 2
In the second part, we explain the nitty details of the runtime information available for async methods with a state machine. This covers how the debugger is able to fetch the debug information necessary to build the "async call stacks".

{{< youtube jfxGk5rdj-0 >}}

## Part 3
On the third part, I get a real application and debug an issue with the "async call stacks". I go over Visual Studio features (which I worked on!) and explain how to debug it in practice.

{{< youtube QstdWSQMBQ4 >}}

<br>
