What is quorum consensus and why every dev should know it.
==========================================================

Understanding quorum consensus to design your distributed database systems
--------------------------------------------------------------------------

[![Dinesh Kumar K B](https://miro.medium.com/v2/resize:fill:88:88/1*xilx79TPQvZOybmbWnyOKw.jpeg)](https://medium.com/@dineshkumarkb?source=post_page-----6da5783721a3--------------------------------)
[![Geek Culture](https://miro.medium.com/v2/resize:fill:48:48/1*bWAVaFQmpmU6ePTjNIje_A.jpeg)](https://medium.com/geekculture?source=post_page-----6da5783721a3--------------------------------)

[Dinesh Kumar K B](https://medium.com/@dineshkumarkb?source=post_page-----6da5783721a3--------------------------------)

![](https://miro.medium.com/v2/resize:fit:700/0*4Kn_rTOuvdMoVOyf)

Photo by [imgix](https://unsplash.com/@imgix?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

Introduction:
-------------

Every system design interview has a question of "how to optimise" given a specific scenario. Not a single design interview escapes this paradigm.Optimisation doesn't necessarily mean giving a "one-size fits all" solution. Well! technically, such a solution doesn't exist.

We propose a solution, analyse it's trade-offs for the given use-case and place them on the table to the interviewer. Any solution we come up with, should have a pros and cons comparison. To give such a solution, every developer should know what kind of a system they are dealing with.

For instance, if we asked to optimise a database storage system(Of course it's a distributed system), we can't blindly bring up a magic number that improves the performance of a database system. The questions you need to ask either yourself or to the interviewer is,

1.  Is the data-base read heavy/write heavy?
2.  What kind of application is using the database?
3.  Are we using the database for transactions(OLTP) or analytics(OLAP)?
4.  How consistent do we want our system to be? Do we fail a read/write operation during concurrent writes or are we ok to show a old data while the write operation is in progress?

Distributed data-base storage systems:
--------------------------------------

The first step to optimise/design any distributed system is to ensure we understand the CAP theorem.

-   Consistency
-   Availability
-   Partition Tolerance

When the data is distributed across a no-shared architecture i.e.storing data on independent systems (scaling out), the same data is stored in different systems or different data centers which are located in different geographies. This can be achieved by replication or partitioning but essentially a combination of both.

![](https://miro.medium.com/v2/resize:fit:700/1*mf2_w7J2cbMShTCDhzDjSg.png)

A typical distributed data base system

Above is a simple distributed system architecture diagram. Each replica is called a node which is typically a *Virtual or a physical machine.*

Both the replicas have the same data sets. Sometimes a single data-set is partitioned across multiple replicas which is also called as sharding. However, partitioning and replication are done together. They both go together predominantly.

Consistency:
------------

Having the same copy of data across all the replicas is consistency. This ensures that no two users get different sets of results while querying the data base.

Availability:
-------------

The data is always available even if one of the nodes are down for some reason.

Partition Tolerance:
--------------------

This means the data is still available even when the communication between the nodes is disrupted. The nodes are up and very much in a running condition. It's just that the nodes cannot communicate with each other.

The CAP theorem states that it is impossible to achieve all the above three qualities simultaneously in a distributed system.

Let's discuss the aspects of different combinations in CAP theorem.

CA (Consistency, Availability) --- These systems are highly available and the data across them is consistent and up to date.

CP(Consistency, Partitioning) --- They support consistency and partition tolerance while sacrificing availability.

AP(Availability, Partitioning) --- Systems that support high availability and partitioning whereas they don't support consistency.

Since network failures are unprecedented and unavoidable, it is technically impossible to achieve a CA system. Therefore network partitioning cannot be avoided. Only in an ideal world, does the network partition not happen and the data is replicated as is to other nodes.

> A system continues to function despite the existence of a partition. This is not about having mechanisms to "fix" the partition, it is about tolerating the partition, i.e. continuing despite the partition --- Partition Tolerance

However, practically like I said, network partitioning is indispensable in a distributed system because if a system is tolerant to partition it should shut itself down as the data gets distributed instead of partitioning. Technically, only single node systems are partition tolerant. But that defeats the whole purpose of distributed systems. Doesn't it? So considering partitioning, we should choose between consistency and availability.

![](https://miro.medium.com/v2/resize:fit:511/1*NwQhToP46je_rT8i3eWFqA.png)

Database nodes with partitions

Let's take the above example, where we have a distributed system of 3 nodes with partitions. We have to make a choice between consistency and availability as we already have partitioned the data.

Choosing availability over consistency:
---------------------------------------

Let's say now node2 goes down.Data cannot be replicated to node2. If we happen to choose availability, the system then accepts read/write operations. Having said that, the node 2 is still down whilst node 1 and node 3 have updated data. This makes node 2 outdated with stale data.This is a AP system.

Choosing consistency over availability:
---------------------------------------

Now, let's choose consistency. This means that all 3 nodes should have the same data in response to a query. In this case, we must block all new write operations. However, in case of handling sensitive data like financial transactions, the systems are expected to show the latest transactions and the balance.So banking systems can choose to display an error message for new transactions until the errors are fixed and show the balance before the new transaction was made. This makes the user less insecured as his transaction is not left high and dry. The only obstacle is he/she is not able to make new transactions until the error is fixed. Having said that, we have to consider all these factors while choosing the appropriate system with right CAP based on the requirement.

Quorum Consensus:
-----------------

This is one of the distributed lock manager based concurrency control protocol in distributed database systems. It can guarantee the read and write operations.

Let's establish few ground rules.

N = Number of replicas in a distributed system

W = A write quorum of size W. For a write operation to be considered successful, the write should be acknowledged from W replicas.

R = A read quorum of size R. For a read operation to be considered successful, read must wait for responses from atleast R replicas.

> Generally, if there are N replicas, every write must be confirmed by W nodes to be considered successful and we must query at least R nodes for each read.

Therefore N=3, W=1 means that, the write must be confirmed by at least one node for the operation to be successful. The parameters N, W and R are configurable. Typically N is chosen to be an odd number and we set W=R=(N+1)/2. Usually W+R > N makes the system tolerable to unavailability and ensures consistency.

With N=3, W=2, R=2 we can tolerate 1 unavailable node.

With N=5, W=3, R=3, we can tolerate 2 unavailable nodes and so on.

Read and Write trade-offs:
--------------------------

Sometimes, your use-case may demand your system to be either read savvy or write savvy.

If R=1 and W=N, then the system is optimised for faster reads.

If W=1 and R=N, then the system is optimised for faster writes.

Choose the quorum values wisely based on your requirement.

Consistency models:
-------------------

A consistency model defines the degree of data consistency.

Strong Consistency:
-------------------

Any read operation by any user always returns the latest written data. A user would never see a stale data.

Weak Consistency:
-----------------

The read operations may not see the most updated value.

Eventual Consistency:
---------------------

As the name implies, the updates are propagated to all nodes given a period of time(eventually).

Strong consistency is achieved by not accepting new reads/writes when a node is down.However, this compromises on availability. Therefore databases like Amazon Dynamo DB and Cassandra have eventual consistency.

Summary:
========

In the quorum, you could consider R and W as the minimum number of votes required for a read and write operation(respectively) to be considered successful. Banking systems demand a high consistency as any loss in data would result in a heavy financial loss.

Having a clear understanding of the quorum would help you fine-tune your distributed system as per your requirements.

References:
===========

-   <https://stackoverflow.com/questions/47539213/how-ca-distributed-system-according-to-cap-theorem-can-exist>
-   <https://codahale.com/you-cant-sacrifice-partition-tolerance/>
-   <https://stackoverflow.com/questions/36404765/why-isnt-rdbms-partition-tolerant-in-cap-theorem-and-why-is-it-available/64427972#64427972>

[Quorum](https://medium.com/tag/quorum?source=post_page-----6da5783721a3---------------quorum-----------------)

[System Design Interview](https://medium.com/tag/system-design-interview?source=post_page-----6da5783721a3---------------system_design_interview-----------------)

[System Design Concepts](https://medium.com/tag/system-design-concepts?source=post_page-----6da5783721a3---------------system_design_concepts-----------------)

[Distributed Systems](https://medium.com/tag/distributed-systems?source=post_page-----6da5783721a3---------------distributed_systems-----------------)

[Database](https://medium.com/tag/database?source=post_page-----6da5783721a3---------------database-----------------)

[![Dinesh Kumar K B](https://miro.medium.com/v2/resize:fill:144:144/1*xilx79TPQvZOybmbWnyOKw.jpeg)](https://medium.com/@dineshkumarkb?source=post_page-----6da5783721a3--------------------------------)

[![Geek Culture](https://miro.medium.com/v2/resize:fill:64:64/1*bWAVaFQmpmU6ePTjNIje_A.jpeg)](https://medium.com/geekculture?source=post_page-----6da5783721a3--------------------------------)

[Written by Dinesh Kumar K B
---------------------------](https://medium.com/@dineshkumarkb?source=post_page-----6da5783721a3--------------------------------)

[518 Followers](https://medium.com/@dineshkumarkb/followers?source=post_page-----6da5783721a3--------------------------------)

-Writer for

[Geek Culture](https://medium.com/geekculture?source=post_page-----6da5783721a3--------------------------------)

Python Back-End Developer, CKAD |AWS | Django | Flask | Fastapi |Azure | [www.linkedin.com/in/dineshkumarkb](http://www.linkedin.com/in/dineshkumarkb) | [https://dock2learn.com](https://dock2learn.com/)