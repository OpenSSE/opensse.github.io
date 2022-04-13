---
layout: post
title:  "OpenSSE in Rust I - Restarting from scratch (or so)"
date:   2022-04-06
author: Raphael Bost
---

I have never used this blog to post anything about OpenSSE or the new works I
contributed to these past few years. Mainly for (bad) reasons. But, as I am
working on my spare time on a new exciting step (at least for me) for this
project, I thought this was a good occasion to change this.

Indeed, as you might have guessed with the title, I am starting to re-implement
OpenSSE in Rust. Only part of it, though: I really doubt about the practicality
of schemes like Janus (a forward and backward-private scheme using bilinear
maps), and probably will not add it in the portfolio. Similarly, Σoφoς (a.k.a.
Sophos), although being quite elegant and theoretically optimal, is in practice
superseded by Diana, which only uses symmetric cryptography primitives, while
Σoφoς uses RSA (more precisely a trapdoor permutation) as a key building block.

Hence, this post is the first of a series of ... a few, that will try to explain
my thoughts, the issues, and the solutions I encountered during this
re-implementation. As such, it is definitely more of a developer notebook, than
it is a scientific article, or a documentation of the project. However,
researchers interested in searchable encryption might find interesting (or at
least debatable) topics discussed in these posts. And I will be happy to have
some feedback about them (and the series in general).

So let's start...

## Why re-implementing?

The current implementation of OpenSSE in C++ is quite substantial. My work on
the code spanned over 5 years: the first commit dates back from March 2016, for
the Σoφoς implementation accompanying the [CCS paper](https://ia.cr/2016/728),
and the last one from June 2021, when I was finishing the implementation of the
[Tethys page-efficient scheme](https://ia.cr/2021/716). The result of those 5
years is around 26k lines of code and tests, a project that has continuous
integration with sanitizers to check for memory errors, a good code coverage
(even though this metric is not perfect), static analysis, and, most important
of all, a few users, who are using it as the reference implementation of Σoφoς,
Diana, Janus and probably in the future, Tethys.

However, I feel quite unconformable with this code, for many reasons. Let me try
to list some of them:

1. It is in C++. On the one hand, the code is very efficient, and some of the
   latter schemes could not have been implemented in any other language (except
   for C), due to its reliance on low-level libraries for asynchronous IOs. On
   the other hand, using C++ is dangerous: I have no formal background in C or
   C++ (read, I learned it all by myself), especially on the security side. And
   even though I use unit testing and sanitizers to make sure the code is not
   *too* buggy, there is still a security risk with using C or C++, especially
   when developing a security application.

2. The code has been written over a long period: meaning it can be of quite
   uneven quality, with different APIs. It is not uniform. And this is an issue
   if I want to make the project more usable. I need good and well-documented
   APIs, as well as completed implementations (for example, there is no
   protocol runner -- i.e. the network-facing components -- for Janus and
   Tethys).

3. Despite being re-used, it is getting less and less "plug-and-play" for the
   user. During a development phase, as the dependencies I use are up-to-date,
   the CI gives me the guarantee, that compiling the project works without any
   quirks (at least on a new install). Unfortunately, I am getting more and more
   problems with the variety of "compatible" platforms (MacOS and Linux) as well
   as older distributions. It seems to me that I am starting to lean towards
   these open sourced implementations of papers that cannot be realistically
   recompiled and reused by people other than their original developer (and I
   hate that).

4. The build system and the package management is terrible. Sure, using CMake is
   better than writing my own Makefile. Sure, I can use git submodules to embed
   some of the dependencies (i.e. Google Test). But this should not have to be,
   and when I am using out-of-tree dependencies, I regularly have issues. For
   example, the build instructions changed in the last two years for
   [gRPC](https://grpc.io), the RPC framework I have been using in OpenSSE.
   Using CMake to build the library is officially supported for Windows only:
   [Bazel](https://bazel.build) is the recommended build system for gRPC on
   Linux. But I do use CMake right now, and I *de facto* rely on CMake being
   able to find the gRPC headers and library somewhere on my system, usually
   through CMake install files. Files which are not produced anymore with a
   Bazel build. I also have issues with RocksDB, but this time with linking, and
   I have not been able to fix these.

Now what? What are my options?

I feel I only have a few. First, I could go on, and let this project die. I find
this depressing and a waste. I believe this work can be used in the future,
either by me or someone else, and not touching it anymore prevents any serious
future work. Then, I could try to fix things and implement new schemes on the
existing basis. This would not solve the above problems, at least not their
roots. And this is essentially what I have been doing lately with the
implementation of Tethys, making things quite worse. Last, I could start from
scratch, by using tools I would be more comfortable with, with a newly found
motivation, redesigning the APIs, the code, assessing the purpose and the
usefulness of each component.

And this is this last option I chose. It is time to pick the tools.

## What are the objectives of this new implementation?

Now that we are building something from the ground up, we can chose our
objectives, the goals we will try to tend to, and from there chose our tools. So
let's make a list, order by importance:

1. **Secure and safe code.** We are coding a security application that will
store users' data, and we cannot mess with the security. This means that the
server must be secured, even though the security guarantees of the SSE schemes
we will implement tell us that the server cannot have access to *too much*
sensitive data (see all the discussions and papers about leakage functions and
leakage-abuse attacks for the definition of 'too much'), as the server can act
as a point of entry for an attacker into the users' network. It actually also
implies that the client side of the SSE protocols must be secured. The safety
must not be forgotten either. By safety, I mean the possibility of misusing the
APIs (e.g. by sending a query using the wrong protocol).

2. **Easy, understandable and usable APIs.** The goal of this project is to be
   used, both as a reference implementation serving as a comparison point to
   other works, and as a library for other larger applications. Hence, the
   interfaces must be consistent across the different schemes, easy to use, and
   hard to misuse. The trendy term for that in the cryptographic world is
   *boring*: the APIs should come with no surprise. However, this does not mean
   they cannot be complex or only involve simple operations; it implies they are
   not complicated.

3. **Efficient and fast implementations.** The ultimate goal is to be
competitive with modern, unencrypted key-value stores, and this cannot be
achieved without offering a fast implementation of the schemes. Also, we want to
be both fast *and* efficient: many of the schemes implemented in OpenSSE are
IO-bounded, and spawning dozens of threads to respond to a query might be a
waste of resources or might scale poorly. Note however, that it is only third on
our list: in our case, performance is useless without security, and the most
performant system is the one that is not used.

4. **Reusable and understandable code.** I really do not want to be alone to
contribute to this project, and I hope that sometimes someone will be able to
work on it in order to improve its performance, fix a bug or implement a new
scheme, without me (do you know about the [bus
factor](https://en.wikipedia.org/wiki/Bus_factor)?). And even for me, it would
be better to have a healthy ground for future implementations or uses. To do so,
this project needs a well-documented, and well-designed codebase. Easier said
than done.

5. **Having a fun time.** Duh! I really don't have a lot of time, so it will be
   better if I enjoy working on this new project. Maybe, I should put that
   first.

These objectives are not great: they are hardly measurable, probably incomplete,
and surely naïve. That is why they are merely goals. As stated in the last item,
I am doing this for myself, not for a company, or for a future start-up. This is
just for me, as a hobby (yes, I do have strange hobbies).

## Choosing the right tools

The first and main tool to consider is of course the programming language. The
ones I took into consideration are the following:

* C/C++
* Java
* Rust
* Go
* Swift
* Zig

### Language properties

Which were the criterion? I wanted a language that is widespread, and well
known. This forbids options like OCaml, Elixir, or Scala. Not that any of these
are bad, they are just lesser known that the ones in my list. Another criterion
was to pick languages I know and/or I wanted to work with. For example, I don't
know Go, nor Swift, but I definitely am curious. On the other hand, I really do
not want to use Python (besides the fact that I don't believe it can fit any of
the objectives above). Next, I need a "systems programming language". This kind
of rules out Java, but not quite as it is a very good language when it comes to
networking, and the portability is amazing. By 'portability', here I mean the
ability to reuse essentially the same codebase for different operating systems,
my targets being Linux, BSD, MacOS (mainly for development, but also because, in
the Academia, a lot of people use Apple computers), and maybe, some day,
Windows. This is definitely Java's strong point, and excludes *de facto*
languages like Objective-C (which I know quite a bit), or C# (which I do not
intend to know). Finally, efficiency is also an important criterion, although I
could choose to implement the cryptographic primitives in a very performant
language and then use this library from an other language, fitting the schemes'
implementation better.

From that point of view, Java is definitely behind all the other picks, being
the only non-compiled language. However, with Java, you have access to a huge
array of very useful libraries when dealing with databases, for example, and not
the least, Lucene. Java is also garbage collected, as is Go.

### Tooling

An other important element is the tooling associated with the language. This
includes the package managers, build systems, static and dynamic analysis tools.
I do exclude more generic toolings like code formatters, code coverage,
continuous integration as they tend to be now available to every language.

From that point of view, C and C++ are grim. There is no widespread and easily
usable package manager: [vcpkg](https://vcpkg.io/) is definitely a good tool,
but as dependencies are usually distributed as dynamic libraries through the
package manager of the distribution (for Linux, but Homebrew or MacPorts have
essentially the same role on MacOS for that matter), it makes things hard when
it comes to binary distribution and dynamic libraries (how do you make sure that
your local build of RocksDB will behave similarly to the binary distributed by
`apt` on an old Ubuntu LTS?). The only real solution is to re-compile every
dependency and hope they are all managed by your code package manager. Not
ideal. My experience with vcpkg (or Conan) is essentially zero, so I might (read
surely) be mistaken here.

Also, CMake is... I have mixed feelings about CMake: it is super powerful, but
not easy to use, and quite verbose. There is no surprise why it is used for
projects like LLVM: it is fast, precise, cross-platform, has easily configurable
builds, and multiple backends (`make`, Ninja, Visual Studio,...). In the past, I
have also used [SCons](https://scons.org): being able to write you build script
in Python is great, but SCons is super slow, and not really easier to use than
CMake, at least for C/C++ projects. But, in the end, I am not entirely convinced
by CMake, especially when compared to modern languages like Rust, Go, Swift, or
Zig, which embed their build systems, and do it well, often together with a
package manager.

On the static analysis side, the picture is different: C/C++ and Java being the
oldest languages of the list, one has access to a lot of interesting resources
when it comes to finding code smells, evaluating code quality, or just finding
bugs. Static analysis tools for C and C++ are particularly interesting and
powerful, even the free ones, like `clang-tidy`. I also had the chance to use
[Coverity Scan](https://scan.coverity.com), and it has been super helpful. Both
tools allowed me to fix numerous bugs, even though I feel like most of these
bugs would not have existed with a memory-safe language like all the other
languages of the list, except Zig. So it feels like a zero-sum game: we have
nice tools to find bugs which would not have had existed with other languages.

For dynamic analysis, we have a similar picture: C and C++ have access to great
tools such as Valgrind or the sanitizers (ASan, UBSan, TSan, MemSan, and many
others). Rust also has access to very interesting tools, like
[Miri](https://github.com/rust-lang/miri), which can help finding bugs in
`unsafe` code, as well as the aforementioned sanitizers (in particular ASan,
which can find memory leaks, those being allowed in safe Rust). I was not able
to find any interesting or relevant information about the other languages, so I
considered that the dynamic analysis tooling was lacking for those.

### Performance

Even though performance is not the top priority, it definitely is something
important. And from that point of find, I considered three main criterions:

* Is the language compiled to machine code?
* Does it use garbage collection to manage memory?
* Does it support asynchronous code is a nice way?

I don't have much of a problem with interpreted languages or languages using
bytecode in themselves. They can be very useful and practical for many tasks.
But it is hard to use them for high-performance disk IOs, mainly because it
requires close interactions with the OS's kernel and a small latency.
Unfortunately, languages that are not compiled to machine code cannot
efficiently use such interfaces: the interpreter or the VM adds a lot of latency
on the one hand, and as these languages are cross-platform, it is hard to use a
system-specific API. Sure, native extensions can be built, but these are often
hard to develop and maintain (and it does not really solve the latency problem).

Garbage collection is a huge debate point. It saves you from having to manage
memory and also prevents bugs that can have a huge impact on performance (I am
looking at you, memory leaks). On the other hand, it often works in a
non-predictable manner and cause lag spikes at moments, during which we would
like to avoid them. It is not a huge deal-breaker in my use case, but it is
definitely something to take into account for low-latency applications.

Finally, asynchronous programming. When implementing Thetys, I started using
asynchronous APIs to access the disk in a controlled and precise way. This
allowed me to reach a level of performance that had not been seen before with
searchable encryption scheme. Unfortunately, due to the lack of support for
asynchronous programming in C++, I had to fully commit to the *callback hell*,
and the result is far from being amazing from the API point of view (even with
the use of [`std::future`](https://en.cppreference.com/w/cpp/thread/future)). I
would like to have something more elegant. And, unfortunately, executors will
not be introduced in the standard until C++23, and the coroutines introduced in
C++20 are not really usable IMHO. Zig seems to be in a similar state. On the
other side of the spectrum, Go and Rust have a very good async ecosystem: it is
a key feature of Go from its inception, and, for Rust, it is something quite
new, but well tested and with a lot of libraries, tools and documentation (and
hype).

### Expressiveness

Finally, as I am writing a library with (tentatively) well-thought APIs, I
really need a language in which I can express complex types, genericity, logical
dependencies, all of which can be checked with a strong typing. In the old
french academic training in computer science, I have been taught OCaml and
lambda calculus, and it still shows today.

As such, the introduction of [concepts in
C++20](https://en.cppreference.com/w/cpp/concepts) is a great addition to C++,
especially as it has been closely combined with templates (a.k.a. generics). We
can find similar features in most of the other languages in my list, with the
notable exception of Go (well, it has been introduced very lately, in December
2021, and still looks a bit fresh).

Note that this expressiveness requirement barred Python from my list for good:
it does not even have types...

### Why Rust?

In the end, I chose Rust, because I feel like it fits all the needs I have, on
every aspect. The tooling is the best I have experienced in any language so far:
no annoying package dependencies, easy builds, integrated testing and
documentation (and testing *inside* the documentation). The performance is
great. The async ecosystem is both efficient and very elegant (despite being
difficult to access if you want to go beyond just using it). I love the type
system (although I encountered annoying limitations -- see the next post). When
I code in Rust, I feel like I am in charge (like in C++), but that I am helped
in the meantime (unlike in C++): the compiler tells me when I make non-trivial
mistakes and helps me to fix them. And when I am stuck, I can call for help a
very welcoming and open community.

Java's performance overhead is too much for this particular application, and, as
I wrote before, it is too high level for what I have to do (interacting with the
OS).

Using C or C++ still is a huge liability, despite what can be said about "modern
C++" and its security (no, you cannot always use smart pointers; yes, from the
moment you start using raw pointers, there is a security risk).

Swift looks very exciting: automatic reference counting is a good compromise
between garbage collection, and manual memory management. It is also
cross-platform now. However, it looks like it is a bit too high level, and it
does not integrate very well with C and C++ libraries (e.g. the key-value stores
I will have to use for Diana). Also, even though the support and the community
is great for MacOS (and iOS for that matter), I am not sure that other *nix
platforms are as well supported.

Zig is an interesting case. It looks super efficient, and is a clear contender
to C. It produces very efficient and very small binaries, and is an amazing
language for HPC and embedded systems. Its compatibility with C is very
appealing. Unfortunately, it is a new language for me, and it does not attract
me a lot, especially given that I know Rust, which has not a massively different
spirit, and which syntax I prefer.

Finally, Go would be my second pick. The goroutines look very nice: I like the
concurrent and imperative approach a lot. I would (have) miss(ed) generics,
though. The tooling looks great, too. Of course, I am also very interested in
the language because of its spread (if Docker is implemented in Go, it *has* to
be for a reason). But in the end, I chose Rust for its better performance,
better safety (in particular for concurrent code) and my pre-existing knowledge
of the language (meaning I can start implementing with the right idioms right
away).

## Conclusion

So re-implementing OpenSSE in Rust it is. What are the next steps?

First, I will explain the choices I made for the APIs. Then, I will, in an
independent post, quickly describe the work I did for the cryptographic toolkit
I will need later on. After that, I might write something about some
meta-programming that I found useful to generically test code (Rust has a basic
testing framework, not as rich as, for example, Google Test). Hopefully, we will
quickly get into the actual implementation of the SSE schemes, starting with
Diana and then Tethys.
