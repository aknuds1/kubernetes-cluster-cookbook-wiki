A Kubernetes cluster generally has to be maintained like any traditional infrastructure. What does this mean? Generally, you must construct it in a way that it is highly available and self-healing. If you want to maintain 5-9 (%99.999) uptime, you must carefully consider the layout of the cluster. Multiple workers, multiple masters, multiple entry points. This page will break down the major parts of a Kubernetes from a system architect viewpoint. Some of these concepts may be counterintuitive to the agile concepts of containers- but in order to deliver your applications reliably- they must be adhered to.

***

###ETCD

ETCD is a distributed key/value store. In a Kubernetes cluster its main purpose is to hold data pertaining to node health, application location, desired states for applications, and user secrets and access/authentication information. ETCD is the brain of your cluster- possibly the most important single pice of the ecosystem. Without it, you will not- and can not deploy anything anywhere. None of the Kubernetes services will run, and anything that was running at the time of ETCD loss will remain in a detached state- doing nothing. This means that if you were to only be concerned with one thing in your cluster- it should be ETCD. If everything else in the cluster dies- you can actually restore them using the same ETCD data to a nearly identical- if not totally identical state. ETCD will act as your restore mechanism for the whole cluster.

***

First, lets look at the data itself:

An ETCD member stores all its data in a single directory on your filesystem. This is important to know for both security and performance. Conveniently, you are able to specify the ETCD base directory- meaning you can put the data wherever you want to. You want to store on local disk- I will explain why later, and you want to store your data on its own volume if possible. These are the two most important rules when it comes to performance. As for security; generally you want to make sure that the ETCD data is being written somewhere with proper permissions- not world readable, and not in any home directories.

Back to the local volume statement above. Generally your first idea for valuable data is to put the data on highly redundant, or thoroughly managed storage. But, that will do nothing but harm performance. Why? Because ETCD automatically heals itself with any other members of the ETCD cluster. The data will always be on a "majority" of the peers in a proper ETCD cluster. This means that if you have 3 masters in your cluster- one could be destroyed and you will maintain all your data. With 5 masters, you can lose two members and maintain all your data. With 7 masters- you can lose 3 members and still keep every last bit of data. This means that not only can you rolling maintenance/upgrades of your masters, but you can safely and quickly fix or replace broken members without loss of anything. There is no reason to add further redundancy or storage-level safety to your cluster because the purpose of ETCD is to distribute data across the network. For extra safety you can put the masters in different racks and maintain operation in partial power outages!

***

As discussed above, you can put ETCD on multiple nodes and distribute the data. This table will give you an idea around planning your cluster size:

| Members | Safety Margin | Extra considerations |
|---|---|---|
| 1 | 0 | No safety at all |
| 3 | 1 | Minimum for safe setup |
| 5 | 2 | Even better safety |
| 7 | 3 | Great safety |
| 9 | 4 | Some performance impact possible |
| 11 | 5 | Becoming impractical and slow |

You will see that the ideal size for an ETCD cluster is 3-7.  More members start eating up a lot of infrastructure as well as starting to become difficult to manage. On top of that, since ETCD writes the data to a majority of the nodes, adding more members equates to more writes and a longer time until the data is committed- thus safe. I strongly urge you to deploy 5 masters for a production type workflow- however in a pinch 3 can work. One master is totally unacceptable.

***

###API Server

If ETCD is the brain of your cluster- the Kubernetes API server is like the rest of the nervous system. The API server acts as the go-between for all data entering and exiting ETCD- from you and from the worker nodes that your application is deployed on. The great thing about the API server is that considerations for it are mostly driven by your considerations for ETCD. Since each master node on your cluster has an ETCD membership -and- acts as an API server, you will have high availability for both the data your API server consumes as well as the the functionality of the API server.

The API server in this cookbook setup communicates with ETCD through localhost. You will never have to worry about the API server coming disconnected from its data. On top of that- every worker node in your cluster that containers deploy to- access the API servers in a load balanced way though a local HAPROXY setup on the worker itself. If a master dies- the workers pretty much dont know about it at all! They don't skip a beat. If you are outside of the cluster and wish to access the API server in a similar manner, you would simply go through a proxy in a similar fashion- or just set up your kube config to point to all the masters- and it will choose a healthy one itself!

***

###Workers/Minions/Nodes

If ETCD is the brain and the API server is the rest of the nervous system- the workers are your arms and legs. Your workers are where your applications reside, what everyone else interacts with, and are in some ways- the least important part concerning the life of your cluster. Workers can die and be replaced with little- if any consequence to your cluster at all. If you have multiple workers, your application can retain functionality through most situations. You always want multiple minions, but keep in mind that scaling your minions is extremely quick and easy. Its generally safe to start with a small number- and add more in the future if you start running low on space/memory/cpu

***

###Sample scenarios

Lets take a look at a few typical cluster layouts and pick apart their reliability:

| # | Masters | Workers | Notes |
|---|---|---|---|
| 1 | 1 | 1 | Master and worker are the same system |
| 2 | 1 | 1 | Master and worker are separate |
| 3 | 3 | 1 | Slow, traditional infrastructure |
| 4 | 3 | 1 | Fast, agile infrastructure |
| 5 | 1 | 3 | None |
| 6 | 3 | 5 | None |

1. This is a very typical setup for people just starting with Kubernetes. Generally a user deploys it locally on their desktop or laptop just to play around- or just for development uses. This is fine. However- under no circumstance whatsoever should you "rely" on this setup for anything. You just can't. If anything goes wrong- you (and the "cluster") are hosed.

2. This setup is also very common for hobby setups and starters. Again, this works fine for local development or playing around- but for similar reasons, this is totally unreliable. Under this situation, if your worker dies, you can recover your cluster by spinning up a new worker- but if anything happens to your master- it is all wiped and you must start anew.

3. This is an acceptable setup in pretty much no situation as well. Lets say for example this is a setup on very traditional- bare metal only- manually provisioned infrastructure. Your masters are relatively safe, however any applications you deploy to the cluster have zero hardware resilience whatsoever. If your worker dies- so does your applications. Wonderful. How long does it take to get a replacement server? 3 months? Oh, goodie. I'm sure management is okay with this scenario.

4. Now this setup is not too bad. You have an HA setup for your masters, you can survive a minor outage, and replacing your master should be quick if one does die. There is a weakness with there only being one worker- however if you are able to dynamically provision more servers with an IAAS (Infrastructure As A Service) setup, you can not only replace a dead worker quickly- reducing application outage to perhaps a few minutes- but you can expand your cluster in the future to allow more capacity and safety for the applications as well! If you deploy a setup like this, I urge you to spin up a few more workers before really serving any serious purpose.

5. One master. Not okay for production. Period. This can be seen as an acceptable development or playground type environment, however. Lets say you have a handful of developers that just want to test rolling updates, HA application deployments, or distributed workloads. This works fine, but remember- the cluster is rendered wholly useless if your master dies.

6. Life is good. It could be better. But this is workable. Multiple masters for HA for your master services- Multiple workers for HA application deployments. You may want to think about a larger master cluster to add extra durability- but for the most part this is a safe deployment barring total loss.