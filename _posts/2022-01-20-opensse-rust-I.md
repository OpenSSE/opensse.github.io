---
layout: post
title:  "OpenSSE in Rust I - Restarting from scratch (or so)"
date:   2022-01-12
author: Raphael Bost
---

I have never used this blog to post anything about OpenSSE or the new works I contributed to these past few years.
Mainly for (bad) reasons. But, as I am working on my spare time on a new exciting step (at least for me) for this project, I thought this was a good occasion to change this.

Indeed, as you might have guessed with the title, I am starting to re-implement OpenSSE in Rust. Only part of it, though: I really doubt about the practicality of schemes like Janus (a forward and backward-private scheme using bilinear maps), and probably will not add it in the portfolio. Similarly, Σoφoς (a.k.a. Sophos), although being quite elegant and theoretically optimal, is in practice superseded by Diana, which only uses symmetric cryptography primitives, while Σoφoς uses RSA (more precisely a trapdoor permutation) as a key building block.

Hence, this post is the first of a series of ... a few, that will try to explain my thoughts, the issues, and the solutions I encountered during this re-implementation.
As such, it is definitely more of a developer notebook, than it is a scientific article, or a documentation of the project. However, researchers interested in searchable encryption might find interesting (or at least debatable) topics discussed in these posts. And I will be happy to have some feedback about them (and the series in general).

So let's start...

## Why re-implementing?

The current implementation of OpenSSE in C++ is quite substantial. My work on the code spanned over 5 years: the first commit dates back from March 2016, for the Σoφoς implementation accompanying the [CCS paper](https://ia.cr/2016/728), and the last one from June 2021, when I was finishing the implementation of the [Tethys page-efficient scheme](https://ia.cr/2021/716).
The result of those 5 years is around 30k lines of code and tests, a project that has continuous integration with sanitizers to check for memory errors, a good code coverage (even though this metric is not perfect), static analysis, and, most important of all, a few users, who are using it as the reference implementation of Σoφoς, Diana, Janus and probably in the future, Tethys.

However, I feel quite unconformable with this code, for many reasons. Let me try to list some of them:
1. It is in C++. On the one hand, the code is very efficient, and some of the latter schemes could not have been implemented in any other language (except for C), due to its reliance on low-level libraries for asynchronous IOs. On the other hand, using C++ is dangerous: I have no formal background in C or C++ (read, I learned it all by myself), especially on the security side. And even though I use unit testing and sanitizers to make sure the code is not *too* buggy, there is still a security risk with using C or C++, especially when developing a security application.

2. The code has been written over a long period: meaning it can be of quite uneven quality, with different APIs. It is not uniform. And this is an issue if I want to make the project more usable. I need good and well-documented APIs, as well as completed implementations (for example, there is not protocol runner -- i.e. the network-facing components -- for Janus and Tethys).

3. Despite being re-used, it is getting less and less "plug-and-play" for the user. During a development phase, as the dependencies I use are up-to-date, the CI gives me the guarantee, that a compiling the project works without any quirks (at least on a new install). Unfortunately, I am getting more and more problems with the variety of "compatible" platforms (MacOS and Linux) as well as older distributions. It seems to me that I am starting to lean towards these open sourced implementations of papers that cannot be realistically recompiled and reused by people other than their original developer (and I hate that).

4. The build system and the package management is terrible. Sure using CMake is better than writing my own Makefile. Sure, I can use git submodules to embed some of the dependencies (i.e. Google Test). But this should not have to be, and when I am using out-of-tree dependencies, I regularly have issues. For example, the build instructions changed in the last two years for gRPC. Using CMake to build the library is officially supported for Windows only: Bazel is the recommended build system for gRPC on Linux. But I do use CMake right now, and I *de facto* rely on CMake being able to find the gRPC headers and library somewhere on my system, usually through CMake install files. Files which are not produced anymore with a Bazel build. I also have issues with RocksDB, but this time with linking. I have not been able to fix these.

Now what? What are my options?

I feel I only have a few. First, I could go on, and let this project die. I find this depressing and a waste. I believe this work can be used in the future, either by me or someone else, and not touching it anymore prevents any serious future work.
Then, I could try to fix things and implement new schemes on the existing basis. This would not solve the above problems, at least not their roots. And this is essentially what I have been doing lately with the implementation of Tethys, making things quite worse.
Last, I could start from scratch, by using tools I would be more comfortable with, with a newly found motivation, redesigning the APIs, the code, assessing the purpose and the usefulness of each component.

And this is this last option I chose. Now, I need to pick the tools.

## Why Rust?

