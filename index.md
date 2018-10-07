---
layout: default
title:  "Message Modelling in IBM WebSphere Message Broker"
tagline: "Tutorials on modelling messages IBM WebSphere Message Broker"
---

## Message Modelling in IBM WebSphere Message Broker

> _Despite the tons of examples and docs, the MRM is voodoo. Damned cool voodoo, but still voodoo._

_paraphrased from Apaches mod_rewrite_

### Message modelling?
Message modelling is the art of describing messages in formats that are not self-describing. Examples of such formats are CSV, COBOL Copybooks and other kinds of fixed length and delimited formats. To enable WebSphere Message Broker to parse these formats it must be told how to interpret the data. Message modelling is the process of crafting such an description - called a Message Set.

* [Repeating key value rows](repeating-key-value-rows/)
* [Tagged fixed length data](tagged-fixed-length/)

----

#### About
I wrote these two tutorials sometime during 2008 while working at [Zystems](http://www.enfo.se/en/services-solutions/integration-solutions/) and they where later published and "announced" in [this thread on MQSeries.net](http://www.mqseries.net/phpBB2/viewtopic.php?p=256009). They've since lived on my now defunct blog and on [my homepage](http://marten.gustafson.pp.se). While doing some autumn cleaning on my website I stumbled on them and decied to move them here instead. Everything has been migrated pretty much as is. I've ported the tutorial texts from PDF to Markdown, but that's about it.
