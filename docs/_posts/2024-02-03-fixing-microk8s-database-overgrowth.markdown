---
layout: post
title:  "Fixing microk8s database overgrowth!"
date:   2024-02-03 11:00:0 +0100
categories: microk8s sqlite k8s-dqlite
---
> **_NOTE:_**  *Before finishing this post I started to work on fixing k8-dqlite source code, I do not know how long it will take to be finished and published. Until then you can try this manual remedy.*

# Introduction
In this post I would like to describe how to perform manual cleanup of microk8s database.

Microk8s is great distribution of Kubernetes. It is very easy to setup and update. It is built around idea of `noOps` so it does not really need much maintenance. 

But it by default uses `DQLite` instead of `etcd`, this database is fast, but there is component `kine` which is used to emulate `etcd` interface above the `DQLite`. 
In current implementation kine can come to troubles.

Kine is basically `append only log` build above SQL database. When object is updated, a new version of an object is added into database and it has reference to the old version. In order for db to not grow, 
it is being continuously compacted, by removing old versions of objects. This compaction might work too slow and the database wil grow bigger and bigger. In addion to this the sql statements that has to deal with multiple versions of objects
are not optimal and when there is many versions, they start to be slow. Last but not least, each time kine encounters gap in index, it creats a record to fill it, this records are removed by compaction but I encountered situation in which i had
three hundred thousands of such record. In comparison after cleanup  the dba had eight thousand records, five thousands of which were probably some leaked ipam handles from `calico`.

# Solution

First we need to do some diagnosis. There are some debug options to be enabled, so we can observe valuable information in logs. Then we will connect to live database and perform some queries.
While doing maintenance you can turn off kubelite by `snap stop microk8s.dameon-kubelite`, this will keep dqlite running but shuts down kubernetes api server. This will ease the database, and it might recover on its own in few days.
Otherwise it will be impossible to perform manual deletes. But you should start with diagnosis first. 

Do *backup* of database first, do not do any other stuff as in this post, so you do not need to restore db.


Edit
`/var/snap/microk8s/current/args/k8s-dqlite`

and add `--debug` option, so it will look like that
`--storage-dir=${SNAP_DATA}/var/kubernetes/backend/
--listen=unix://${SNAP_DATA}/var/kubernetes/backend/kine.sock:12379
--debug`

`snap restart microk8s.daemon-k8s-dqlite`
Now you can see what sql statements are being executed. Do not allow this to run too long, it might fill up /var/log in few days.
to observer it in real time you can use `journal ctl -f -u snap.microk8s.daemon-k8s-dqlite`.

Now connect to the database, you need to use ip addres of server, because dqlite does not listen on localhost
 `/snap/microk8s/current/bin/dqlite -s <ip>:19001 -c /var/snap/microk8s/current/var/kubernetes/backend/cluster.crt -k /var/snap/microk8s/current/var/kubernetes/backend/cluster.key k8s`

this opens a shell, you can chcek if it works by entering simple query

{% highlight sql %}SELECT * FROM kine LIMIT 1;{% endhighlight %}
If it does not work output anything, it means client cannot connect to server, or there some changes in your configuration or newer version.

Then check the size of db, this is basically visible from storage directory, but it is harder to tell what is too much.
{% highlight sql %}SELECT count(*) FROM kine;
{% endhighlight %}

If there are hundreds of thousands of records something is wrong, depending on size of you clusters, there should be not more than tens of thousands of records.

Compaction lag is what we are after, it should be around 2000, in bad situations it might be 100k or more.
{% highlight sql %}SELECT TIME(),MAX(id)-(
    SELECT crkv.prev_revision FROM kine crkv WHERE crkv.name = 'compact_rev_key') 
FROM kine;{% endhighlight %}


If Compaction lag is too big, we can help by compacting things manually.

Next important issue are the aforementioned gaps. It might be the cause of problems, but manual compaction might also cause gaps to appear.

{% highlight sql %}SELECT COUNT(*) FROM kine WEHRE name LIKE 'gap-%';{% endhighlight %}

If number of gap records is too big, they can be deleted.
{% highlight sql %}DELETE FROM kine WHERE name LIKE 'gap-%';{% endhighlight %}

Now we can see if there are some problematic records, with more than  thousands records.
{% highlight sql %}SELECT name, COUNT(*) as cnt FROM kine GROUP BY name HAVING cnt >= 100 ORDER BY cnt;{% endhighlight %}

IT would take a long time for automatic compaction do compact record wich have 30000 records for example so we can delete it manually.
You will see dqlite trying to delete the records again, because it is doing it in batches of 500 i think, so you can restart the k8s-dqlite, to save several minutes.


for exaple to manually compact `/registry/leases/kube-system/kube-scheduler` use following statement,
{% highlight sql %} DELETE FROM kine WHERE 
  kine.id<(SELECT MAX(kv2.id) FROM kine as  kv2 WHERE kv2.name=kine.name) 
  AND kine.id<(SELECT max(mk.id)-1000 FROM kine as mk) 
  AND kine.name='/registry/leases/kube-system/kube-scheduler' LIMIT 10000;
{% endhighlight %}

You can increase LIMIT or remove condition to keep last 1000 records in database, or replace `WHERE kine.name=kv2.name` with `kv2.name=<problematic name>`,but it requires you to write same string twice, and it might result in error

If `databse is locked` is displayed, try again, until you succeed, you might sometimes receive timeout, but it does not mean query was not executed.
