---
title: OpenSSE
permalink: /docs/home/
redirect_from: /docs/index.html
---

OpenSSE is a project gathering the implementation of several single-keyword symmetric searchable encryption schemes, with different tradeoffs between security and efficiency. Even though production-quality is an objective, be aware that OpenSSE is for now a research project and should not be trusted nor used for sensitive data.

## Getting started

If you are not aware of searchable encryption, its guarantees and its weaknesses, we encourage you to take a look at the [dedicated introduction]({{ "/docs/searchable-encryption/" | prepend: site.baseurl }}) in the documentation.


## Testing & Contributing

We are looking for people ready to help at fixing, improving and securing the code, at implementing existing schemes, at improving the performance, and even at designing new constructions... Use GitHub's issues manager to propose new ideas and advertise a bug.

The code is freely available on [GitHub]({{ site.git_address }}), separated in three repositories, `crypto-tk` for the cryptographic tools, `db-parser` for a multiformat database parser, and `opensse-schemes` for the schemes' implementation *per se* (you are most likely to be interested by the latter).
Follow the [instructions]({{ "/docs/getting/" | prepend: site.baseurl }}) to properly build and use this code.

If you are interested in contributing, we highly recommend that you read the documentation, in order to really understand what were the design principles and motivations. 


<!-- Look at the [projects](https://github.com/orgs/OpenSSE/projects) page to see what we are focused on, or propose new ideas . -->