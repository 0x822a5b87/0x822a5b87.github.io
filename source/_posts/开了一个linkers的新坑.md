---
title: 开了一个linkers的新坑
date: 2025-01-15 11:37:54
tags:
  - rust
  - compiler
---

> 早上逛 [reddit rust](https://www.reddit.com/r/rust/) 时偶然看到了 [Ian Lance Taylor](https://www.airs.com/blog) 介绍 [linkers](https://a3f.at/lists/linkers) 的系列文章，觉得很有意思，决定开一个新坑来看看能不能自己实现一个简单的 linkers，目前找了一些资料，看起来工作量并不小。这里先做个简单的记录，参考的文章和书籍：
>
> - [linkers by Ian Lance Taylor](https://a3f.at/lists/linkers)
> - [byo-linker](https://github.com/andrewhalle/byo-linker)
> - [Linkers and Loaders](https://www.amazon.com/Linkers-Kaufmann-Software-Engineering-Programming/dp/1558604960/ref=sr_1_1?crid=24DJPGT6TCPSU&dib=eyJ2IjoiMSJ9.blypyivja49kN_oMLeGvLUYdmuxfh0xHvWmmz6FjaF-covkmqayhb5XmNBBIwDAbisFkJaB2k580X2V6D8nWuopCyzyijCWFM05AuAiNzMgaERzu_oxeZ8xEEIpl8vXed5zSGn5h8GXGtkBQX7THg0RltzVDWOPkMugTdtoG-9F-9i_407kB3K0tgLCeSnNq2B6MimVn2yD1KmgLnL-XlE5JnCQcS8qh1C0h6KY4hWA.uWjN7FnLv144NGBHfuKQYTUYvP0gr-bw8yJMFd3qvO4&dib_tag=se&keywords=linkers+and&qid=1736912625&sprefix=linkers+and+l%2Caps%2C430&sr=8-1)
>
> 可能会涉及到的基础知识包括：
>
> - [**SPARC** (**Scalable Processor ARChitecture**)](https://en.wikipedia.org/wiki/SPARC)
> - [x86 instruction listings](https://en.wikipedia.org/wiki/X86_instruction_listings)
> - [Executable file](https://en.wikipedia.org/wiki/Executable): `ELF`, `PE`, `OMF`
> - [Symbol table](https://en.wikipedia.org/wiki/Symbol_table)
> - 其他待总结
