+++
author = "Isadora"
title = "dotnet2md"
date = "2022-09-29"
description = "Tool that converts .NET metadata and XML documentation to markdown files."
tags = [
    "extension",
    "projects"
]
+++

A tool that converts .NET metadata and XML documentation to markdown files.

<!--more-->

I was very inspired by [mdBook](https://rust-lang.github.io/mdBook/) and wanted something similar to generate documentation to the game engine I am working on. I was aware that C# can generate [XML files from documentation comments](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/), so I thought about leveraging both of these.

Of course, I went from a straighforward XML to markdown generator to a tool that extracts metadata information of all public types exposed by a target assembly. You can find the [source code here](https://github.com/isadorasophia/dotnet2md) or even try it out.

![](/images/projects/murder_demo.png)
_Deploying mdBook after generating the markdown files._

The general structure of the files is:
```
└── src
    └── pre_SUMMARY.md
    ├── summary.md
    └── <AssemblyName>
        ├── <ClassName>.md
        └── <NamespaceName>
            └── <ClassWithinNamespaceName>.md
```

I did a lot of things that I am particularly proud here:
- "Decompile" the members signature to display code in the documentation.
- Link to an external documentation source. So far, I support MonoGame and Microsoft documentation websites.
- Link to other members within the same assembly at the documentation.
- Automatically generate files for mdBook.
- Enabled GitHub Actions to automatically generate release binaries.

Anyway! I can't wait to actually release the engine and its documentation powered by this tool.
