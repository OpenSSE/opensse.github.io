---
title: What is Searchable Encryption?
permalink: /docs/searchable-encryption/
---

You want/need to outsource some data in the Cloud?
You want to encrypt those data because you know that there are bad guys out there, who might try to steal those data, or at least learn some information about you, your family or your company?
You still want to enjoy the nice features offered by Cloud services, and in particular search through the outsourced documents?

Searchable encryption aims at solving this problem efficiently.
The main problem is actually that it is impossible to construct an encrypted database that is both totally secure and as efficient as its unencrypted counterpart. 
For example, an ideally secure encrypted database would not leak the number of results of a search query. Yet, hiding this information would require to increase the communication between the client and the server beyond the size of the actual result list that an unencrypted database would respond.
This is a simple example of the tradeoffs between security and efficiency of searchable encryption.


Indeed, on one end of the spectrum, we have very secure databases constructed with very powerful tools such as [fully homomorphic encryption (FHE)](https://en.wikipedia.org/wiki/Homomorphic_encryption#Fully_homomorphic_encryption) or [oblivious RAM (ORAM)](https://en.wikipedia.org/wiki/Oblivious_ram), and on the other end, what is called *legacy-compatible databases*, *i.e.* encrypted databases built on top of a regular, non-encrypted database.
For the latter case, the encryption scheme can be seen as a proxy transforming keywords in documents into cryptographic tokens, computed deterministically, which are then placed in the database.
Yet, as one might expect, these legacy-compatible constructions, despite being almost as efficient as unencrypted databases, are very unsecure because they are subject to attacks based on frequency-analysis, compromising the secrecy of the encrypted dataset.


The searchable encryption community hence tries to construct schemes whose performance are practical, and with a good level of security.