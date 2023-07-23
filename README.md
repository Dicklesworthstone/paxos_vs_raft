_**Disclaimer**: Although I came up with the overall structure of the tables and the various section topics, GPT4 created all the contents, and then I used Claude2 to review it for accuracy._

# Background

## The Problem Consensus Algorithms Attempt to Solve:

First a couple definitions:

**Distributed System:** This refers to a system composed of multiple independent components (often referred to as nodes, servers, or machines) that interact with each other. These components may be spread across different geographical locations, and they communicate with each other over a network.

**Consensus:** Consensus is a process of agreement. In the context of a distributed system, the goal is to ensure that all functioning nodes in the system can agree on a particular value or state. This is important for maintaining the consistency of data across the system, among other uses.

Now, let's take a non-technical analogy to explain this further:

Consider a scenario where you have several friends trying to decide on a restaurant for dinner. Each friend represents a node in a distributed system, and the restaurant choice represents the value that needs agreement.

Here are the challenges you'd face, which are similar to the challenges in a distributed system:

_Agreement:_ All friends must agree on the same restaurant. This is equivalent to all nodes in a distributed system agreeing on the same value.

_Single Value:_ The group of friends can't split up and go to different restaurants - they need to agree on one place where they all will dine. Similarly, in a distributed system, we want to ensure that all machines agree on a single value.

_Termination:_ The decision-making process must eventually conclude. If your friends keep debating indefinitely, they'll never get to eat. In the same way, a consensus protocol should ensure that given enough time, all functioning nodes in the system can come to a decision.

_Fault Tolerance:_ It's possible that some of your friends might not respond to messages, maybe because they're busy or their phone is off. Despite this, the rest of the group still needs to make a decision. In a distributed system, the consensus protocol should be able to handle failures of some nodes and still reach a decision among the remaining ones.

Consensus protocols aim to ensure that all functioning nodes in a distributed system can agree on a single value, despite potential communication delays or node failures. They are a critical part of maintaining data consistency and reliability in distributed systems.

## Paxos:
Paxos is a family of protocols for solving consensus in a network of unreliable or fallible processors (nodes). Conceptually, the Paxos protocol is a conversation between a proposer, acceptors, and learners. The proposer suggests a value, and the acceptors decide whether to agree or disagree. The learners then learn the chosen value. The critical point in Paxos is that any proposed value is chosen if a majority of acceptors agree. This means that if a value has been chosen, any majority of acceptors will include at least one acceptor that has agreed, preventing another value from being chosen. This makes it resilient to network partitions and node failures.

## Raft:
Raft, on the other hand, simplifies the process by electing a leader in the first phase, who then makes all the decisions until it fails. The election ensures that there's always a majority agreement on who the leader is. The leader then dictates the log entries (which correspond to state changes in the system). Other nodes replicate the leader's log, and the leader ensures that a log entry is safely replicated on a majority of the servers before considering the entry committed.

______________________________


# High Level Comparison of the Paxos and Raft Consensus Algorithms:

| Step | Paxos  | Raft  |
|------|--------|-------|
| **Initialization** | No leader. Any node can propose a value. | Nodes start in a follower state. If a follower receives no communication, it becomes a candidate and starts an election. |
| **Election** | Not applicable. | Candidates request votes from all nodes. Any node will vote for the first candidate that asks. The candidate becomes a leader if it receives votes from a majority. |
| **Proposal** | A proposer picks a unique number n and sends a 'prepare' request with number n to a majority of acceptors. | The leader starts sending append entries (heartbeats) to all other nodes. |
| **Promise** | If an acceptor receives a 'prepare' request with number n greater than that of any 'prepare' request to which it has already responded, it responds to the request with a promise not to accept any more proposals numbered less than n. | Not applicable. |
| **Acceptance** | If the proposer receives a response to its 'prepare' requests from a majority of acceptors, it sends an 'accept' request to each of those acceptors for a proposal numbered n with a value v. | If followers don’t hear from a leader then they can become a candidate and start a new election. |
| **Commitment** | If an acceptor receives an 'accept' request for a proposal numbered n, it accepts the proposal unless it has already responded to a 'prepare' request having a number greater than n. | Once a follower learns that its log entry is stored on a majority of servers, it applies the entry to its local state machine. |
| **Learn** | A value is chosen when a proposer issues an 'accept' request and a majority of acceptors respond. | Once the leader learns that a log entry is stored on a majority of servers, it applies the entry to its state machine and returns the result of that execution to the client. |
| **Reconciliation** | If a proposer's 'prepare' request fails because some acceptors have already accepted another proposal, the proposer can issue a new 'prepare' request with a higher number and the value of the highest-numbered proposal among the responses. | The leader ensures that entries committed on former leader are copied to the new leader's log entries. |


______________________________


# More Detailed Comparison:

**Initialization**

| Detail | Paxos | Raft |
|--------|-------|------|
| Initial state | Each node is in a passive state, waiting for proposals. Nodes have no particular roles and there is no designated leader. | Each node starts as a follower in a given term. A term is a period of time during which leadership transfers may occur. |
| Triggering action | Any node can decide to propose a value. It does not need to receive any special permission or signal to do so. | If a follower doesn't hear from a leader for a certain duration (known as the election timeout), it self-promotes to a candidate and starts an election. |
| System setup | Paxos nodes need to know the list of participants but do not need any particular configuration. | In Raft, nodes need to know about each other and have a configuration that specifies the maximum time to wait before starting an election. |

**Election**

| Detail | Paxos | Raft |
|--------|-------|------|
| Election triggering | There is no formal leader election in Paxos. Any node can propose a value at any time. | A follower transitions to a candidate and requests votes from all other nodes in the cluster. |
| Voting | Each node independently decides which proposals to accept based on the proposal number. There is no coordinated voting process. | Nodes vote for the first candidate that requests their vote in a given term. Any votes for other candidates are ignored in that term. |
| Election result | Paxos doesn't elect a leader, so there is no election result. | A candidate becomes a leader if it gets votes from a majority of nodes in the cluster. |

**Proposal**

| Detail | Paxos | Raft |
|--------|-------|------|
| Proposing a change | A proposer selects a unique proposal number \(n\) and sends a 'prepare' request with \(n\) to a majority of acceptors. | The leader starts the proposal process by logging the new entry in its own log. |
| Proposal delivery | The 'prepare' request is sent to a majority of nodes (acceptors) in the system. | The leader sends 'AppendEntries' RPCs (which include the new log entries along with heartbeats) to all other nodes. |
| Proposal reception | Acceptors individually decide whether to respond to the 'prepare' request based on its number. | Followers append the entries to their logs if their logs are consistent with the leader's. |

**Promise**

| Detail | Paxos | Raft |
|--------|-------|------|
| Locking in a proposal | An acceptor, upon receiving a 'prepare' request with \(n\) greater than any it has seen, promises not to accept any proposals numbered less than \(n\). | Not directly applicable in Raft, as the leader controls the log entries and followers can't promise to accept only certain entries. |
| Sending promise | The acceptor sends a 'promise' response back to the proposer, including the highest-numbered proposal it has accepted. | Not applicable in Raft. |
| Handling promise | The proposer collects 'promise' responses. If it receives responses from a majority, it can move to the acceptance phase. | Not applicable in Raft. |

**Acceptance**

| Detail | Paxos | Raft |
|--------|-------|------|
| Accepting a proposal | If a proposer receives 'promise' responses from a majority of acceptors, it sends an 'accept' request to each of those acceptors with a proposal numbered \(n\) and a value \(v\). | The leader waits until a majority of followers have written the entry to their log. |
| Handling acceptance | An acceptor, upon receiving an 'accept' request, accepts the proposal unless it has already responded to a 'prepare' request with number > \(n\). | Followers write the entry to their logs and respond to the leader. |

**Commitment**

| Detail | Paxos | Raft |
|--------|-------|------|
| Committing a proposal | An acceptor, upon receiving an 'accept' request for a proposal numbered \(n\), accepts the proposal unless it has already responded to a 'prepare' request with number > \(n\). | The leader waits for acknowledgements from a majority of followers that they have written the log entry. |
| Communicating commitment | The acceptor communicates its acceptance back to the proposer. | Once the entry is stored on a majority of servers, the leader applies the entry to its local state machine. |

**Learn**

| Detail | Paxos | Raft |
|--------|-------|------|
| Learning the result | A value is chosen when a proposer issues an 'accept' request and a majority of acceptors respond with acceptance. | Once the leader learns that a log entry is stored on a majority of servers, it applies the entry to its state machine and returns the result of that execution to the client. |
| Disseminating the result | The proposer acts as a 'learner' to disseminate the result to other nodes. | The leader communicates the committed entry back to followers in subsequent 'AppendEntries' RPCs. |

**Reconciliation**

| Detail | Paxos | Raft |
|--------|-------|------|
| Reconciliation after failure | If a proposer's 'prepare' request fails because acceptors have accepted another proposal, the proposer can issue a new 'prepare' request with a higher number and the value of the highest-numbered proposal from the responses. | The leader handles inconsistencies between its log and those of followers by forcing followers to duplicate its own log entries. |
| Reconciliation after restart | Paxos does not inherently handle restarts. It relies on external mechanisms to store and retrieve state. | Raft leaders maintain a 'nextIndex' for each follower, which is the index of the next log entry the leader will send to that follower. When a leader restarts, it initializes all nextIndex values to the last index in its log. |

______________________________


# Theoretical Guarantees and Runtime Complexity:

Below is a comparison of the theoretical guarantees and typical runtime complexities of Paxos and Raft. Note that these complexities are not always strict, as they can vary depending on the specific implementation, network conditions, and other factors.

|                  | Paxos                                       | Raft                                     |
|------------------|---------------------------------------------|------------------------------------------|
| Safety           | Guarantees safety under all conditions      | Guarantees safety under all conditions   |
| Liveness         | Does not always guarantee liveness          | Guarantees liveness under certain conditions |
| Completeness     | Requires a majority of nodes to be available | Requires a majority of nodes to be available |
| Consensus Time   | O(N) in the number of messages sent         | O(N) in the number of messages sent      |
| Node Failure     | O(N) in the number of messages              | O(N) in the number of messages          |
| Network Failure  | O(N) in the number of messages              | O(N) in the number of messages          |
| Leader Election  | Not applicable (no formal leaders)          | O(N) in the number of nodes             |
| Log Replication  | Not applicable (no logs in basic Paxos)     | O(N) in the number of nodes             |

Key: 
- Safety: The property that only a value that has been proposed can be chosen, and once a value has been chosen, that choice is final.
- Liveness: The property that nodes can make progress and eventually agree on a value.
- Completeness: The property that every request eventually gets a response (either accept or reject).
- Consensus Time: The time complexity to reach consensus.
- Node Failure: The time complexity to handle a node failure.
- Network Failure: The time complexity to handle a network failure.
- Leader Election: The time complexity to elect a new leader (for algorithms that use leaders).
- Log Replication: The time complexity to replicate logs across nodes (for algorithms that use logs).

In summary, both Paxos and Raft provide strong safety guarantees, ensuring that the system can't commit to contradictory values. However, they handle liveness and leader election differently. Raft has the concept of leadership, which Paxos lacks in its basic form. This allows Raft to ensure liveness under certain conditions, where Paxos could potentially get stuck if proposals keep getting preempted.

The run time complexities are generally linear (O(N)) with respect to the number of nodes or messages, but this can be affected by network conditions, the specific topology of the system, and how failures are handled. In general, both Paxos and Raft are designed to handle failures well and continue to provide consistency guarantees in the face of failures.


# Protection Against Malicious Nodes:

Both Paxos and Raft use a majority-based system to reach consensus, which means they can tolerate up to (N-1)/2 faulty nodes, where N is the total number of nodes in the system. In this context, a faulty node could be one that is not functioning correctly or is behaving maliciously. If an attacker controls more than half the nodes, they can potentially disrupt the consensus process or manipulate the chosen value.

However, it's important to note that these consensus algorithms were designed to handle faulty nodes (ones that crash or are slow), not necessarily malicious nodes that behave arbitrarily or actively try to subvert the system. This is a different problem known as Byzantine fault tolerance.

Byzantine fault tolerance (BFT) is the property of a system to function correctly even if some components are faulty or malicious. There are variations of Paxos and Raft, like Byzantine Paxos and Braft, that are designed to handle Byzantine faults. These algorithms use more complex mechanisms to ensure safety and liveness in the presence of malicious nodes, but also require more resources (like a higher number of nodes or more rounds of communication).

Thus Paxos and Raft are equally susceptible to a group of malicious nodes if those nodes form a majority. But in the presence of Byzantine faults, neither basic Paxos nor Raft provides guarantees; for that, you would need to use a Byzantine fault-tolerant consensus algorithm.

# Pros and Cons:

|   | Paxos Pros | Paxos Cons | Raft Pros | Raft Cons |
|---|------------|------------|-----------|-----------|
| 1 | Proven correctness. Paxos is theoretically sound and well studied. | Complexity. Paxos's correctness comes at the cost of operational complexity. | Simplicity. Raft was designed to be easy to understand and implement. | Relative novelty. Raft is newer and not as extensively vetted as Paxos. |
| 2 | High availability. Paxos provides high availability as long as a majority of nodes are operational. | Hard to implement. Paxos's complexity can lead to errors in implementation. | Understandable. Raft's state machine replication is more understandable for most developers. | Fewer implementations. Because it's newer, there are fewer existing Raft implementations to choose from. |
| 3 | Flexibility. Paxos is more of a framework for consensus and can be adapted to a variety of use cases. | Hard to understand. The Paxos algorithm is notoriously difficult to understand. | Built-in leader election. Raft incorporates leader election into the protocol. | Not optimized for extremely large clusters. Raft typically works best with a smaller number of nodes. |
| 4 | Well researched. Paxos has a long history and a lot of academic research behind it. | Not built-in leader election. Paxos requires additional mechanisms to handle leader election. | Easier to implement. Raft's design makes it easier to implement correctly. | Requires more communication for log compaction. Raft can require more communication overhead to handle log compaction. |
| 5 | Robust under high load. Paxos can handle high loads and many clients. | Lower performance. In certain scenarios, Paxos can have lower performance due to multiple communication rounds. | Strong performance. Raft has strong performance due to its efficient leader-based approach. | Can suffer from stale reads. If a follower doesn't know it's been partitioned away, it may serve stale data. |
| 6 | Extensive real-world use. Paxos is used in many high-profile distributed systems. | Difficulty in managing multiple Paxos instances. Managing multiple Paxos instances can be complex. | Log consistency. Raft ensures that the logs on all the nodes are consistent. | Difficulty with large state machines. Large state machines in Raft can be difficult to handle. |
| 7 | Guaranteed consistency. Paxos ensures that only one value will be chosen. | Not optimized for typical case. Paxos optimizes for the worst case scenario, which can impact performance in the typical case. | Optimized for typical case. Raft optimizes for the typical case where there is no partition. | Can suffer from split-brain during network partitions. Raft can potentially elect more than one leader during network partitions. |
| 8 | Works with any number of nodes. Paxos doesn't require a fixed number of nodes. | Requires careful tuning. To get the best performance from Paxos, careful tuning is often necessary. | Works best with a fixed number of nodes. Raft works best when the number of nodes is known and fixed. | Does not guarantee linearizability. Raft does not guarantee linearizability if the client talks to the follower. |
| 9 | Supports many proposers. Paxos can work with many proposers, which can be useful in certain scenarios. | Requires careful handling of failure scenarios. Paxos requires careful handling of failure scenarios to ensure progress. | Progress even in failure scenarios. Raft's leader-based approach allows progress even in some failure scenarios. | Requires more resources for a given level of fault tolerance. Raft typically requires more resources to achieve the same level of fault tolerance as Paxos. |
| 10 | Can handle message loss. Paxos is resilient to message loss. | Requires majority of nodes to be operational. Paxos requires a majority of nodes to be operational to make progress. | Requires fewer messages for consensus. Raft requires fewer messages to reach consensus than Paxos. | Requires regular leader heartbeats. Raft requires regular leader heartbeats, which can increase network load. |

# Variants of Each Algorithm:

| Paxos Variants | Description |
|----------------|-------------|
| Multi-Paxos | An optimization to the basic Paxos protocol that designates one node as a leader to avoid the need for a new leader election for each consensus decision. This makes consensus more efficient. |
| Fast Paxos | A variant of Paxos that reduces message latency in the common case when there is no contention, but requires more processes. |
| Generalized Paxos | An extension to Paxos that allows consensus to be reached on a set of commands rather than a single command, increasing throughput. |
| Cheap Paxos | A variant of Paxos designed for environments where the number of replicas is large but only a small number are needed to tolerate failures. It reduces the cost by requiring fewer servers to participate in the normal case. |

| Raft Variants | Description |
|---------------|-------------|
| Multi-Raft | An optimization that allows data sharding across multiple Raft groups, reducing the load on individual nodes and increasing throughput. |
| Lease Read | An optimization that allows read-only operations to be served by followers without contacting the leader, reducing load on the leader and increasing read throughput. |
| Hierarchical Raft | A variant of Raft designed for large-scale clusters. It introduces a two-level hierarchy to divide the problem of managing a large number of nodes into smaller, more manageable tasks. |
| Copycat | A variant of Raft that provides a user-friendly, high-level programming model for building fault-tolerant state machines. It's designed to make it easier to use Raft in practical systems. |

# History of Their Development:

## Paxos
The story of Paxos, one of the most influential consensus protocols in distributed systems, begins with Leslie Lamport, a computer scientist who had already made significant contributions to the field, including the development of the Lamport timestamp. Lamport's work on Paxos began in the late 1980s, inspired by the challenges of fault tolerance in distributed systems. He sought to create a protocol that could achieve consensus even in the presence of failures, which was no small task.

In 1989, Lamport shared a draft paper titled "The Part-Time Parliament" with colleagues. The paper used the fictional example of a Greek island's parliamentary system, where legislators (i.e., processes in a distributed system) had to agree on laws (i.e., values), despite being occasionally absent (i.e., failing). Lamport's paper was unique and controversial for its time, not only because of the complexity of the algorithm but also due to its unusual presentation style. The analogy of the Greek parliament, the island of Paxos, and the use of terms such as "decrees", "ballots", and "legislators" made the paper notoriously difficult to understand.

Due to the complexity of the paper and its cryptic analogy, it was not widely understood or appreciated at first. The paper was rejected by several conferences, and it wasn't until 1998, nearly a decade later, that a simplified version of the paper was published in the ACM Transactions on Computer Systems. This version stripped away the Greek parliament analogy and presented the protocol in more conventional terms, which allowed the computer science community to fully appreciate Paxos for what it was: a significant breakthrough in distributed systems.

Since its eventual publication, Paxos has become a cornerstone of distributed systems design. It is recognized for its efficiency and safety guarantees in the face of network partitions and process failures. The Paxos family of algorithms has grown to include many variants and optimizations, such as Multi-Paxos, Fast Paxos, and Cheap Paxos. These variants address different trade-offs in the protocol, such as the number of communication rounds, the number of processes that need to be involved in a decision, and the use of different quorum sizes and configurations.

Despite its theoretical elegance and practical use, Paxos is known for being hard to understand and even harder to implement correctly. This complexity has motivated the development of alternative consensus protocols, most notably Raft, which was explicitly designed to be as understandable and implementable as Paxos, but with a simpler conceptual model.

In the industry, Paxos and its variants have found use in many large-scale distributed systems. Google's Chubby lock service, which provides coordination services for its distributed systems, uses Paxos for replication. Apache's ZooKeeper, a service for distributed coordination, uses a Paxos-inspired protocol called ZAB. Paxos has also influenced the design of other distributed consensus protocols, such as Viewstamped Replication and Zab.

## Raft
The journey of Raft, a consensus algorithm known for its simplicity and understandability, started relatively recently. The algorithm was developed by Diego Ongaro and John Ousterhout at Stanford University and was introduced in Ongaro's 2014 doctoral dissertation, "Consensus: Bridging Theory and Practice".

The development of Raft was driven by the complexity of existing consensus algorithms, most notably Paxos. While Paxos was well-established and had strong theoretical foundations, its complexity made it hard to understand, teach, and implement correctly. Ongaro and Ousterhout believed that there was a need for a consensus algorithm that could be easily understood and implemented without sacrificing the essential features of safety and reliability.

Raft was designed to be as efficient as Paxos, but with a stronger emphasis on understandability. It does this by dividing the problem into relatively independent subproblems and solving each one in the most straightforward way possible. This separation of concerns makes the protocol easier to comprehend as a whole, and each part is simple to understand in isolation.

The first publication about Raft was a paper titled "In Search of an Understandable Consensus Algorithm" presented at the 2014 USENIX Annual Technical Conference. The paper was met with significant interest and enthusiasm from both the academic community and industry. It was awarded the conference's Best Paper Award, and Raft has since become a popular topic of study in distributed systems courses.

Since its introduction, Raft has seen extensive use in industry and open source projects. It has been implemented in various programming languages and used in various systems, such as etcd, a distributed key-value store from CoreOS; Consul, a service mesh solution from HashiCorp; and TiKV, a distributed transactional key-value database.

Raft has also inspired further research and led to the development of various optimizations and extensions. For example, the Raft Refloated paper by Heidi Howard and Richard Mortier presents a more intuitive model for understanding and teaching Raft. Other works have explored the integration of Raft with sharding techniques for scalability, or extensions to support multi-datacenter deployments.

The creation of Raft marked a significant point in the development of consensus algorithms. By prioritizing understandability and simplicity while ensuring safety and liveness, Raft has become a popular choice for distributed systems needing a consensus algorithm. It demonstrates that it is possible to achieve theoretical soundness without sacrificing understandability, a principle that continues to guide research and development in distributed systems.

