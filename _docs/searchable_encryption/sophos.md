---
title: Σoφoς (Sophos)
permalink: /docs/sophos/
---

Σoφoς (pronounced 'Sophos') is a symmetric searchable encryption scheme with an additional security property called *forward privacy* [proposed](https://ia.cr/2016/728) by Raphael Bost in 2016.

Forward privacy ensures that no information is leaked during the database updates.
This property is very important in practice, as it thwarts devastating adaptive attacks called file-injection attacks.
These attacks, designed by [Yupeng Zhang, Jonathan Katz, and Charalampos Papamanthou](https://ia.cr/2016/172), break the confidentiality of past queries by inserting fake documents in the database, and were namely enabled because the insertion protocol of the encrypted database revealed whether the newly inserted document matches previous queries.

Although Σoφoς is not the first forward-private construction, it is the first that is practically efficient -- actually asymptotically optimal -- and (relatively) scalable.