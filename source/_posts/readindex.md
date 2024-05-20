---
title: Exploring Raft Algorithm - How Does readIndex Work 
date: 2024-05-14 12:23:23
author: xinhua
tags:
- raft 
- distributed system
- algorithm 
---
The Raft algorithm ([1]) is designed to ensure linearizability. It achieves this by ensuring that all replicated logs are applied to the state machine in a consistent order, thereby making the system appear as if it is operating in a single, sequentially executed environment.

In many applications where high data consistency is required, the number of read operations usually far exceeds that of write operations. If the same processing procedure is used for both read and write operations in the Raft algorithm, this can become a bottleneck in performance.

By employing “readIndex” (section 6.4 in reference [2]), a client can directly read the state machine data from any server, not solely from the leader. This not only enhances performance but also maintains the system’s linearizability.

Let’s first discuss the senario without using readIndex: if a client reads from any server (assume it is a follower) directly after its write, a stale read may occur:

![Figure-1](/images/readindex-1.png)
1. The client assigns the value of x as 4 through the leader (write operation).
2. The leader creates a log entry and propogate it to all followers
3. Followers write the log entry and respond
4. Once the leader receives a majority of responses, it commits the log entry to the state machine and informs all followers to commit.
5. Server2 commits to the state machine, but Server3 has not yet committed.
6. The client reads the value of x from the state machine on Server3.
7. Since Server3 has not yet committed the log entry value to the state machine, the client reads stale data.

In etcd, this type of read, which does not utilize readIndex, is allowed as Serializable read ([4]). It offers low latency and high throughput, suitable for scenarios where data consistency requirements are not stringent.

If readIndex is used, the process is as follows:
![Figure-2](/images/readindex-2.png)
1. After committing the value x, the leader assigns the current commit ID to a local variable readIndex.
2. the client starts to read x from server3
3. Server3 first requests the readIndex from the leader
4. Since server3 has not committed x yet, its commitIndex is less than the readIndex. Server3 wait till its commitIndex ≥ readIndex
5. Server3 commits x in state machien, and its commitIndex increases by one.
6. As server3’s commitIndex matches the readIndex, it returns the value x

Here we see that server3 must catch up with the leader’s last committed state in order to return the value of the read operation.

According to the key safety property of Raft: state machine safety (section 5.4 in reference [1]), The readIndex process described above ensures linearizable read consistency.

We also need to consider two scenarios about the Leader

1. The leader has just been elected before the client’s first read operation comes.
2. The leader was deposed (stale leader)

### Scenario 1: New elected Leader

At the beginning of a new Term, before the first client read operation comes ( no write operation ), the leader must ensure its commit index will be at least as large as any other followers’ during its term.

But the leader “never commit log entries from previous terms by counting replicas”.(section 5.4.2 in reference [1]) Otherwise, the rollback issue depicted in Figure 8 of the Raft paper ([1]) will occur. (A Breach of the State Machine Safety Property)

Here, I quote a sentence that is often cited on the internet, which explains how Raft handles this situation.

    If the leader has not yet marked an entry from its current term committed, it waits until it has done so. The Leader Completeness Property guarantees that a leader has all committed entries, but at the start of its term, it may not know which those are. To find out, it needs to commit an entry from its term. Raft handles this by having each leader commit a blank no-op entry into the log at the start of its term. As soon as this no-op entry is committed, the leader’scommit index will be at least as large as any other servers’ during its term. (Section 6.4 in reference [2])

This process can be described as follows:
![Figure-3](/images/readindex-3.png)

### Scenario 2: The Leader is deposed

Raft ensures Election Safety, meaning that no more than one leader can exist within a single term. However, in the case of a network partition, it is possible for two leaders to emerge, though they will be from different terms.
![Figure-4](/images/readindex-4.png)

For read requests, a stale leader may return stale data. For instance, under the read-after-write consistency requirement, a client writes to the leader of term 3, Server1, but the read request is sent to Server5 or Server4.

Due to this reason, the leader needs to initiate a round of heartbeats before issuing its requestIndex to ensure it is not outdated.

In a geo-distributed scenario, this extra round of heartbeat can lead to performance challenges: high latency. To avoid this extra heartbeat round, there is a method called leader leases. Here is an excellent introduction from yugabyteDB: [low-latency-reads-in-geo-distributed-sql-with-raft-leader-leases](https://www.yugabyte.com/blog/low-latency-reads-in-geo-distributed-sql-with-raft-leader-leases/) ([3])


### References

[1] In Search of an Understandable Consensus Algorithm: https://raft.github.io/raft.pdf

[2] Consensus: Bridging theory and practice: https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf

[3] Leader leases : https://www.yugabyte.com/blog/low-latency-reads-in-geo-distributed-sql-with-raft-leader-leases/

[4] Etcd Api: https://etcd.io/docs/v3.6/learning/api/#range

