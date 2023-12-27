---
title: "Authorization as a service"
description: "How and why to build an authorization microservice and make use of the policy as code paradigm"
tags: [access-control]
date: 2023-12-26
math: false
---

Currently I am working on an identity provider, a service that basically centralizes authentication for a range of applications.
Users can log in with their username and password, use Single-Sign-On via Google, and even log in passwordless with Passkeys.
Authentication is who you are. Authorization is what you can do. The question that I asked myself is, how about a centralized
service for authorization? Why would you need it and how would you build one?

# Why?

Let's first of all brainstorm about possible reasons why one would benefit from an authorization service.
I would say in an environment that needs to scale to accomodate a lot of traffic, and where therefore applications get broken
down into microservices, it makes sense to put authorization into its dedicated microservice as well.
This leads to consistency across different applications and allows applications to easily interoperate, because they can
rely on the authorization service to coordinate access control. Also, we save engineering costs, because we do not need to build custom authorization logic in each application. In contrast, each application makes a request to the authorization service to check permissions.
Because all data related to access control is centralized we achieve better auditability and can also potentially build useful common infrastructure on top of that.

# Challenges

A challenge of building an authorization service then is that authorization is on the hot path of a lot of requests coming from potentially a dozen high traffic applications. The service needs to be scalable with low latency. Also high availability is paramount. If the service went down completely, relying applications would have no other choice than to reply "Access Denied" to all of their requests, effectively causing a denial of service.

Additionally, the system needs to be flexible enough to express a wide range of access control policies that the applications need. And most important of all, the access decisions need to be correct, which is easier said then done if we talk about a distributed system that potentially could span over servers across multiple data centers and we therefore have to fight consistency issues when updates to access control policies and their underlying data are made concurrently.

# The CAP Theorem

From the CAP theorem we have learnt that under network partitions, we have to make a choice between consistency and availability. But even without network partitions, we still need to balance a tradeoff between consistency and latency. Eventual consistent databases like Amazon's DynamoDB are already worthy for a post of their own (see the Paper [Dynamo: Amazon's Highly Available Key Value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)). They prioritizes high availability and performance over strong consistency, allowing for a temporary inconsistent view of the data and reconcile later. But it is clear that for a database that stores our permissions, strong consistency is necessary.

# Zanzibar

Through the investigation of my initial question of how to build an authorization microservice, I pretty quickly arrived at a system paper by Google from 2019, named
["Zanzibar: Google's Consistent, Global Authorization System"](https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/). Zanzibar stores Access Control Lists (ACLs) and performs authorization checks. Google's wide range of applications like Mail, Drive, Cloud, Youtube, etc. all rely on Zanzibar for their authorization. Those applications are themselves microservices with billions of users with different authorization challenges like embedded content from other apps and search with permissions in the case of Google Drive. Zanzibar stores more than two trillion ACLs and performs millions of authorization checks per second, so low latency and high availability is required. Google's services are accessed globally, so Google makes sure that data is geographically distributed and replicated close to the user. So Zanzibar also replicates all ACLs to geographically distributed data centers. The need for global consistency of access control decisions despite frequent ACL and membership changes arises, which is the major implementation challenge behind Zanzibar.

# Spanner

The problem is mostly solved by building on top of Spanner (see the paper ["Spanner: Google's Globally Distributed Database"](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)) that provides globally distributed ACID transactions (i.e. in the scenario where data is sharded across different servers), linearizability of reads and writes (i.e. reads and writes, although concurrently, appear to be executed on a single machine) and consistent time stamps.

Spanner uses a collection of popular techniques to achieve this: State machine replication using Paxos, Two-Phase Commit for cross-shard atomicity, and Two-Phase Locking for Linearizability.

## Paxos

Paxos is a consensus algorithm used in distributed systems to achieve consensus among a group of nodes or processes, i.e. make them agree on a single value or sequence of values despite the possibility of failures and network delays.
State machine replication (SMR) is a technique used in distributed systems to ensure fault tolerance by replicating the state of a system across multiple nodes. Paxos ensures that replicas in the system agree on the order and content of operations to be executed, even in the presence of faults, ensuring consistency across all replicas. There are many papers written about Paxos by its creator, Leslie Lamport, e.g. ["The Part-Time Parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf) and ["Paxos Made Simple"](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf). But honestly, it can still be quite hard to understand. That's why another consensus algorithm, Raft, was developed. Its ideas are described in ["In search for an understandable consensus algorithm"](https://raft.github.io/raft.pdf). There is also a really nice visualization that eases understanding [here](https://thesecretlivesofdata.com/raft/).

## Two-Phase Commit

The Two-Phase Commit (2PC) protocol is a distributed algorithm used to ensure atomicity in transactions. It guarantees that either all participating nodes commit to a transaction or they all abort it, thereby maintaining consistency and preventing partial updates or inconsistencies across shards. It distinguishes two phases: In the prepare phase, each participant prepares the transaction and responds with an acknoledgement to the coordinator. Then we enter the second phase, the commit or abort phase. If all participants respond positively in the prepare phase, the coordinator sends a commit message to all participants. Upon receiving the commit message, participants apply the changes permanently. If any participant failed to prepare or the coordinator decides to abort for any reason, it sends an abort message to all participants, and they roll back the transaction, reverting to the pre-transaction state.

## Two-Phase Locking

Two-Phase Locking (2PL) is a concurrency control mechanism used in databases to ensure serializability and prevent conflicts among concurrent transactions. It operates in two phases: the growing phase and the shrinking phase. In the growing phase, a transaction can acquire locks on data items it accesses but cannot release any locks until it reaches a point where it needs no more locks. In the shrinking phase, a transaction releases all the locks it holds. Once a lock is released, it cannot acquire any new locks. Two types of locks exist: Write locks, which are exclusive, and read locks, which are shared.

## Consistent Snapshots

The interesting thing about Spanner is that it does not require read locks, which is a huge advantage. It does it by using consistent snapshots via Multi-Version Concurrency Control. That means the database stores different snapshots of the data for each update, associated with a timestamp that is consistent with causality. Read transaction can read from such a prior consistent snapshot without read locks. Obtaining globally valid commit timestamps though is definitely a challenge, because each physical clock has a certain drift and uncertainty. Spanner returns two timestamps representing an uncertainty window, guaranteing that the actual timestamp will be somewhere in that window. Transaction will wait the delta between those two timestamps before the commit, ensuring that there is no overlap of the uncertainty windows between two concurrent transactions. Google reduces the uncertainty window by deploying atomic clocks and GPS receivers in each datacenter.

Now that we understand how Zanzibar solves the consistency problem, lets move on to how permissions are stored in the database and how access control checks are performed. Then we will look at specific implementation challenges and performance optimizations.

# Graph Based Authorization

Zanzibar stores ACLs in the form of relationship tuples: (object # relation @ user).
For example: (document:123 # owner @ user:jill) means Jill is the owner of document 123. But we can also relate resources to other resources:
(group:engineering # member @ group:security) means all members of the security group are members of the engineering group. And we can relate resources to other relations. (group:engineering # manager @ group:leadership#member) means everyone that is a member of the leadership group is a manager of the engineering group. In this way we build a graph, where nodes are objects and users and relations form the edges between them. Checking for access becomes a graph traversal problem.

# The New Enemy Problem

There is an issue to watch out for though, and it is coined "The New Enemy Problem" in the Zanzibar paper. Namely, what if ACL updates and content updates happen concurrently, i.e. Alice removes Bob's access to the document and Charlie adds new content to the document. Upon doing the ACL check for Bob, is there a possibility that we missaply the old ACL to the new content, i.e., Bob can see the new content? Or is there a possibility that we somehow neglect the order of different ACL updates, causing issues? The answer is of course no, because Spanner guarantees us external consistency due to its timestamp mechanism. An access control check comes with a certain timestamp and observes all writes that come before it.

# Zookies and other performance optimizations

An important insight to optimize performance is that the access control checks don't have to be necessarily evaluated on the latest snapshot. Reading from older snapshots still guarantees consistency.

To avoid using stale ACLs one could try to always evaluate at the latest snapshot. That would require global data synchronization with high latency.
Instead, we can serve most checks at a default staleness with already replicated data. How?
Zookie is a concept within Zanzibar that opaquely encodes a timestamp with together with the content update, ensuring all ACL updates prior to that have lower timestamps. The client sends this zookie in subsequent ACL checks to ensure that the check snapshot is at least as fresh as the timestamp for the content version. Zookies reduce latency by being able to pick timestamps corresponding to already globally replicated data.

Other performance optimizations utilized in Zanzibar are request hedging to reduce tail latency (sending requests to multiple servers and use the first response that comes back), performance isolation to protect against misbehaving clients (limit on CPU usage, outstanding RPCs, etc.),
optimized processing of large and deeply nested sets through an indexing service, and hot spot mitigation through distributed caching.
Distributed Caching means we distribute subproblems to nodes likely to have the answer. Subproblems are things like is a user member of a certain group and so on. We can distribute subproblems evenly among servers using consistent hashing:

[Consistent hashing](https://www.youtube.com/watch?v=UF9Iqmg94tk) is a technique designed to address the problem of distributing data across multiple nodes in a scalable and efficient manner while minimizing disruptions caused by node additions or removals. Both data keys and servers are hashed onto a common hash space, usually represented as a ring. The data is assigned to the closest node looking clockwise.
The advantage of this method is that when a node is added or removed, only a portion of the keys needs to be remapped, reducing the amount of data movement required. The keys that were previously mapped to the failed or removed node are reassigned again to the closest available node on the ring looking clockwise.

# SpiceDB

There are different open source, Zanzibar-inspired databases for creating and managing security-critical application permissions. One of them is [SpiceDB](https://github.com/authzed/spicedb). Developers create a schema and use client libraries to apply the schema to the database, insert relationships and query the database to check permissions. It offers a pluggable storage system that has support for MySQL, CockroachDB, Spanner, etc. CockroachDB could be somewhat called the open source variant of Spanner. There are [sample applications](https://github.com/manaty226/sample-app-with-spicedb) and also a [Youtube video](https://www.youtube.com/watch?v=lXizkPvSbHU) introducing the project.

# OPA and OPAL

There exists another paradigm of building scalable unified authorization systems, that came up a lot during my research. [Open Policy Agent](https://www.openpolicyagent.org/) and its administration layer [OPAL](https://docs.opal.ac/). OPA is a policy engine that runs as a sidecar to the application and acts a decision point for access control. OPAL is a system on top of OPA that keeps each OPA instance up to date with respect to data and policy updates.
Policies are separete from the data and stored in a Git repository expressed in Rego, a Policy-as-code-Language specifically designed to express permissions. OPAL monitors the repository and can push updates to the OPA client. We can push new policies to the Git repository without changing application code and we can spin up an entirely new application that can also make use of the already existing policies, making it highly scalable.

# Policy as Code

Policy as Code is an interesting paradigm, that offers similar benefits to Infrastructure as Code.
We can use a dedicated languages to express policies in a very readable, flexible and reusable way.
It decouples policy from code, so we can make changes in those policies without needing to change the code. Also we achieve
minimal code repetition across services. We can version policies using Git, and can audit policy changes. The whole process enhances
security, because it reduces errors and bugs by having well documented and easily readable policies using a uniform language designed for that purpose.

Rego is an example of such a language right now primarily used for infrastructure configuration in the context of Kubernetes. [Cedar](https://www.cedarpolicy.com/en) by AWS is another such language with a focus on application-level authorization. It is readable and performant, enhances security and can potentially help with compliance to industry regulations. Another huge advantage of policy as code is that we can build other useful tooling on top of the centralized policies, e.g., to optimize policies and prove that those policies actually represent the intended security model. This is what AWS aims to do with Cedar, and there are also [demos](https://github.com/cedar-policy/cedar-examples) available.

# Conclusion

We have seen two approaches of unifying authorization into a dedicated service, offering both performance gains in a highly distributed and interdependent environment, and better security through better auditability and readability of access control policies.
Zanzibar was implemented by Google and is therefore designed to solve Google scale problems, which only a few companies have to face. Zanzibar-like systems are great for high-volume systems and to model complex relationships and permission hierarchies with frequent, dynamic permission changes, but struggle with (a few) policies that can't be expressed as relationships, e.g. those considering environmental attributes.

Policy-as-code systems can express various complex authorization policies, ABAC, RBAC, and so on. They offer the advantage of extracting policies from code to allow for better readibility, auditability and reuse of policies, arguably improving security. But they are only suited for environments with low and medium level of data changes and have problems to efficiently resolve hierarchical permissions where systems require reverse lookup, e.g., answering the question “who has access to this resource?” instead of just “can the user access this resource?”. It will be interesting to see if the policy as code approach will be widely adopted in the future also for some relatively simple, stand alone applications though.

# Further Resources

To learn more about databases and distributed systems in general, there are several high quality lecture series on Youtube:

- Andy Pavlo's [Introduction to Databases](https://www.youtube.com/watch?v=uikbtpVZS2s&list=PLSE8ODhjZXjaKScG3l0nuOiDTTqpfnWFf) (and Advanced Database Systems!) from CMU
- [Distributed Systems Lectures](https://www.youtube.com/watch?v=cQP8WApzIQQ&list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB) by Robert Morris from MIT, and yes that is [the guy that implemented the first computer worm](https://en.wikipedia.org/wiki/Robert_Tappan_Morris) and got 400 hours of community service (luckily did not go to jail)
- [Distributed Systems Lectures](https://www.youtube.com/watch?v=UEAMfLPZZhE&list=PLeKd45zvjcDFUEv_ohr_HdUFe97RItdiB) by Martin Kleppmann, the author of the book "Designing Data-Intensive Applications"
