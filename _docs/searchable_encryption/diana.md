---
title: Diana
permalink: /docs/diana/
---

A major downside of [Σoφoς]({{ "/docs/sophos/" | prepend: site.baseurl }}) is its update throughput.
Indeed although the OpenSSE implementation of Σoφoς is multithreaded, it can only insert roughly 4300 entries per seconds on a high end CPU (Core i7 4790K).
This is due to the use of asymmetric cryptographic primitives (namely RSA) as a fundamental component of the scheme.
And, although the server only performs public key operations that are reasonably fast during search, the client uses private key operations that are a lot slower to encrypt and insert new data.


In an [article](https://ia.cr/2017/805) published in 2017, Raphael Bost, Brice Minaud and Olga Ohrimenko presented a new forward-private searchable encryption scheme that only uses symmetric key primitives.
Their construction, called Diana, derives tokens using a binary tree. 
Although Diana is not asymptotically optimal (the insertion operation is no longer constant time, but logarithmic in the number of documents matching the updated keyword), the computational overhead for both the searches and the updates is less than the one of Σoφoς.