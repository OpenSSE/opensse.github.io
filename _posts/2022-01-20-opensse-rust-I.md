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
The result of those 5 years is around 26k lines of code and tests, a project that has continuous integration with sanitizers to check for memory errors, a good code coverage (even though this metric is not perfect), static analysis, and, most important of all, a few users, who are using it as the reference implementation of Σoφoς, Diana, Janus and probably in the future, Tethys.

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

## What are the objectives of this new implementation?

Now that we are building something from the ground up, we can chose our objectives, the goals we will try to tend to, and from there chose our tools.
So let's make a list, order by importance:
1. **Secure and safe code.** We are coding a security application that will store users' data, and we cannot mess with the security. This means that the server must be secured, even though the security guarantees of the SSE schemes we will implement tell us that the server cannot have access to *too much* sensitive data (see all the discussions and papers about leakage functions and leakage-abuse attacks for the definition of 'too much'), as the server can act as a point of entry for an attacker into the users' network. It actually also implies that the client side of the SSE protocols must be secured.
The safety must not be forgotten either. By safety, I mean the possibility of misusing the APIs (e.g. by sending a query to the wrong server).

2. **Easy, understandable and usable APIs.** The goal of this project is to be used, both as a reference implementation serving as a comparison point to other works, and as a library for other larger applications. Hence, the interfaces must be consistent across the different schemes, easy to use, and hard to misuse. The trendy term for that in the cryptographic world is *boring*: the APIs should come with no surprise. However, this does not mean they cannot be complex or only involve simple operations; it implies they are not complicated.

3. **Efficient and fast implementations.** The ultimate goal is to be competitive with modern, unencrypted key-value stores, and this cannot be achieved without offering a fast implementation of the schemes. Also, we want to be both fast *and* efficient: many of the schemes implemented in OpenSSE are IO-bounded, and spawning dozens of threads to respond to a query might be a waste of resources or might scale poorly.
Note however, that it is only third on our list: in our case, performance is useless without security, and the most performant system is the one that is not used.

4. **Reusable and understandable code.** I really do not want to be alone to contribute to this project, and I hope that sometimes someone will be able to work on it in order to improve its performance, fix a bug or implement a new scheme, without me (do you know about the [bus factor](https://en.wikipedia.org/wiki/Bus_factor)?). And even for me, it would be better to have a healthy ground for future implementations or uses.
To do so, this project needs a well-documented, and well-designed codebase. Easier said than done.

5. **Having a fun time.** Duh! I really don't have a lot of time, so it will be better if I enjoy working on this new project. Maybe, I should put that first.

These objectives are not great: they are hardly measurable, probably incomplete, and surely naïve. That is why they are merely goals. As stated in the last item, I am doing this for myself, not for a company, or for a future start-up. This is just for me, as a hobby (yes, I do have strange hobbies).

## Choosing the right tools