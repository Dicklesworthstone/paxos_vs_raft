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


# Background

## Paxos:
Paxos is a family of protocols for solving consensus in a network of unreliable or fallible processors (nodes). Conceptually, the Paxos protocol is a conversation between a proposer, acceptors, and learners. The proposer suggests a value, and the acceptors decide whether to agree or disagree. The learners then learn the chosen value.

The critical point in Paxos is that any proposed value is chosen if a majority of acceptors agree. This means that if a value has been chosen, any majority of acceptors will include at least one acceptor that has agreed, preventing another value from being chosen. This makes it resilient to network partitions and node failures.

## Raft:
Raft, on the other hand, simplifies the process by electing a leader in the first phase, who then makes all the decisions until it fails. The election ensures that there's always a majority agreement on who the leader is. The leader then dictates the log entries (which correspond to state changes in the system). Other nodes replicate the leader's log, and the leader ensures that a log entry is safely replicated on a majority of the servers before considering the entry committed.

Here's a comparison:

Complexity: Paxos is notoriously difficult to understand, and as a result, it is also hard to implement correctly. Raft, on the other hand, was designed to be easy to understand. Raft breaks down the consensus problem into three relatively independent subproblems and provides a robust solution for each.

Leadership: In Raft, the leader is elected first and then controls the log replication. In contrast, in Paxos, any node can propose a value, which can lead to more contention and complexity.

Performance: In terms of the number of messages required to reach consensus, Paxos can be more efficient because it allows all nodes to propose values simultaneously. However, this can also lead to more contention. Raft might involve more messaging due to the leader election phase, but it can result in less contention because only the leader can propose changes.

Safety: Both Raft and Paxos are safe under all non-Byzantine conditions, meaning they ensure that only a single value will be chosen. The primary difference between the two protocols lies in their design goals and ease of understandability, not in their safety.

In summary, while Paxos is theoretically powerful, Raft's simplicity makes it a more practical choice for real-world distributed systems. Both ensure that consensus can be reached reliably in a distributed system, even in the face of failures and network partitions.
