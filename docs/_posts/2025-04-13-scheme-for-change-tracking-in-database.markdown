---
layout: post
title:  "Scheme for change tracking of entities in database"
date:   2025-04-13 11:00:0 +0100
categories: database architecture
---

# Introducion

A traditional databases are not designed to keep a history of objects (or rows). There are situations you might want to see what user changed a row and what was the change. Being it for regulatory, security or business requirements.
There are many ways how to have this aspect done.

At high level event-sourced databases achieve this as a consequence of their design. Let's just focus on ordinary sql or document databases like mongodb.
Here we can divide solution into two classes, either we add separate change log, or we might add auditing information into rows and then create new, possibly creating append only log, in which we create new version of object instead of overwriting it.

# My solution

Each implementation have its drawbacks. I devised a particular implementation that suited my desires very well.
The problem with canonical representation i had were.
* irrelevant records in a table
* id of object changes
* no id of particular version

To avoid of all this problems i devised a two table solution. The only drawback is more complicated add and edit function 
and that ve store one more version of object than is necessery.

First table contains objects that can be mutated. Second table contains all versions of objects. At start there is same id for one object in both tables(main and history).
After an object is chaged, it is mutated in main table and resulting row is added to history table, gaining new id. Which i call revision id.
Af this the main table is updated with the revision id. The last update is to allow keeping a chain of histories for an object. Each object thus holds a reference to object from history table that it was created from.
What I like about this approach is that it has history prefilled. This allows me to reference particular version using revision id, but also keeps main id for other book keeping tasks.

After imlementing this solution I found out that it would be good to be able to exempt portion of object attributes(*state attributes*) from this history tracking. 
For example some operational attributes,like user account blocking. This can be easily achieved by storing this kind of attributes in different table, possibly with its own veresion.

Blocking and unblocking user would create 2 new version, but the second would be the same like the one before blocking, but it would have different id.
Another solution to this issue would be to include *object hash* in addition to seqential revision id (if we replaced revision id with object hash we might get cycles).
This will be usefull if an object would by modification get to previous state. In that case we would be able to connect multiple versions using one id, for example if we would be doing some data analysis, this would give as better data. 


