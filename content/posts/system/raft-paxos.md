---
title: "Raft and Paxos"
date: 2025-06-09
draft: false
tags: ["distributed system","raft","consensus"]
categories: ["distributed system"]
summary: "Distributed systems face a fundamental challenge: how can multiple nodes agree on a single value or state when faced with failures, network partitions, and concurrent operations? This is where consensus algorithms come into play."
---

## What is Consensus?

Consensus is the process by which a distributed system agrees on a single data value among distributed processes or agents. Key properties include:

- **Safety**: All nodes agree on the same value
- **Liveness**: The system eventually makes progress
- **Fault Tolerance**: The system continues operating despite node failures

Both algorithms guarantee **State Machine Safety**: If a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index.

## The Common Foundation

Despite their differences, Paxos and Raft share a similar high-level approach:

1. One of the n servers is designated the **leader**
2. All operations are sent to the leader
3. The leader appends operations to its log and replicates to other servers
4. Once a **majority** acknowledges, the operation is committed
5. When the leader fails, a new leader is elected

### Server States

```
                     start
                       │
                       v
                ┌─────────────┐
                │  Follower   │◄─────────────────┐
                └─────────────┘                  │
                       │                         │
                       │ times out,              │ discovers new
                       │ starts election         │ term (or leader)
                       v                         │
                ┌─────────────┐                  │
                │ Candidate   │──────────────────┘
                └─────────────┘
                       │
                       │ receives votes from
                       │ majority of servers
                       v
                ┌─────────────┐
                │   Leader    │
                └─────────────┘
```

## Paxos: The Pioneer

### Overview

Paxos, introduced by Leslie Lamport in 1989, was one of the first provably correct consensus algorithms. Despite its theoretical elegance, Paxos is notoriously difficult to understand and implement correctly.

### How Paxos Works

**Key Insight**: In Paxos, terms are divided between servers. A server can only be a candidate in term t if `t mod n = s` (where n = number of servers, s = server id).

### Paxos Leader Election and Log Repair

Here's a critical example showing how Paxos handles leader election differently from Raft:

```
Initial state (leader failed):

s1    1:A  2:B
      └─commit index

s2    1:A  2:B  2:C

s3    1:A  3:B  3:D  3:E

After s1 becomes leader (term 4):

s1    1:A  4:B  4:C     ← Gets C from s2, updates term to 4
      └─new commit index

s2    1:A  2:B  2:C

s3    1:A  3:B  3:D  3:E
```

**Important**: Notice how operation B gets assigned term 4 by the new leader, even though it already existed with term 2. This is unique to Paxos!

### Paxos Pseudocode (Simplified)

**Leader Election Phase:**
```
function become_candidate():
    // Only allowed if term mod n = server_id
    next_term = find_next_valid_term()
    
    // Collect log entries from majority
    for each server in majority:
        response = request_vote(server, commit_index)
        collect_log_entries(response.entries)
    
    // Repair log with highest termed entries
    for each index after commit_index:
        entry = find_entry_with_highest_term(index)
        log[index] = {entry.value, current_term}
    
    become_leader()
```

### Real-World Use Case: Google's Chubby

Google's Chubby lock service uses Paxos at its core. Here's how it works:

```
┌─────────────────────────────────────────────────────┐
│                   Chubby Cell                       │
│                                                     │
│         Master Election via Paxos                   │
│              ┌─────────────┐                        │
│              │   Master    │                        │
│              │  (Leader)   │                        │
│              └──────┬──────┘                        │
│                     │                               │
│        ┌────────────┴────────────┐                 │
│        ▼                         ▼                 │
│  ┌──────────┐             ┌──────────┐            │
│  │ Replica  │             │ Replica  │            │
│  └──────────┘             └──────────┘            │
│                                                     │
│  Used for:                                          │
│  • Distributed locks                                │
│  • Small file storage                               │
│  • Name service                                     │
└─────────────────────────────────────────────────────┘
```

**Production Systems Using Paxos:**
- **Google Chubby**: Powers Google's infrastructure locks
- **Apache Cassandra**: Lightweight transactions use Paxos
- **Amazon DynamoDB**: Metadata consensus
- **Microsoft Azure Storage**: Ensures consistent replication
- **WeChat PaxosStore**: Handles millions of QPS

## Raft: The Understandable Alternative

### Overview

Raft was designed in 2013 specifically to be more understandable than Paxos while providing the same guarantees. Its key innovation is the **strong leader** principle and ensuring only up-to-date servers can become leaders.

### How Raft Works

**Key Insight**: Raft only allows servers with up-to-date logs to become leaders, eliminating the need for log repair during election.

### Raft's Safety Mechanism

```
Example: Why up-to-date check matters

Servers and their logs:
S1: [1:A] [1:B] [1:C]         (last term: 1, last index: 3)
S2: [1:A] [1:B] [1:C] [2:D]   (last term: 2, last index: 4) ✓
S3: [1:A]                     (last term: 1, last index: 1)

When S3 requests votes:
- S1 rejects: S3's log is outdated
- S2 rejects: S3's log is outdated
- S3 cannot become leader!

Only S2 can become leader as it has the most up-to-date log.
```

### Raft's Commit Safety

A unique aspect of Raft is that it cannot immediately commit entries from previous terms:

```
Leader in term 3 with uncommitted entries from term 2:

Before committing old entries:
Leader: [1:A] [2:B] [2:C] [2:D]
              └─ Cannot commit these yet!

After adding entry from current term:
Leader: [1:A] [2:B] [2:C] [2:D] [3:E]
              └─ Now can commit all entries up to here
```

### Real-World Use Case: Kafka's KRaft Mode

Apache Kafka's migration from ZooKeeper to KRaft showcases Raft's practical advantages:

```
┌─────────────────────────────────────────────────────┐
│              Kafka Cluster (KRaft Mode)             │
│                                                     │
│  Controller Quorum (Raft Consensus)                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│  │ Ctrl 1  │  │ Ctrl 2  │  │ Ctrl 3  │           │
│  │(Leader) │  │(Follow) │  │(Follow) │           │
│  └────┬────┘  └────┬────┘  └────┬────┘           │
│       └────────────┼────────────┘                  │
│                    │                                │
│            Metadata Log:                            │
│            • Topics/Partitions                      │
│            • Broker membership                      │
│            • Configurations                         │
│            • ACLs                                   │
│                    │                                │
│       ┌────────────┴────────────┐                  │
│       ▼            ▼            ▼                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐             │
│  │ Broker 1│ │ Broker 2│ │ Broker 3│             │
│  └─────────┘ └─────────┘ └─────────┘             │
└─────────────────────────────────────────────────────┘
```

**Production Systems Using Raft:**
- **etcd**: Kubernetes' brain for configuration
- **Consul**: HashiCorp's service mesh backbone
- **CockroachDB**: Distributed SQL at scale
- **TiKV**: Distributed key-value store
- **Kafka (KRaft)**: Replacing ZooKeeper dependency

## The Critical Differences

### 1. Term Management

| Aspect | Paxos | Raft |
|--------|-------|------|
| **Term allocation** | Server s can only use terms where `t mod n = s` | Any server can try any term |
| **Concurrent candidates** | Impossible (by design) | Possible (may split votes) |
| **Election conflicts** | Cannot happen | Resolved by randomized timeouts |

### 2. Leader Election Process

**Paxos Approach:**
```
1. Become candidate (deterministic term)
2. Collect votes AND log entries
3. Repair log with highest-termed entries
4. Become leader
```

**Raft Approach:**
```
1. Become candidate (increment term)
2. Collect votes (log must be up-to-date)
3. Become leader immediately
4. Replicate existing log as-is
```

### 3. Log Consistency

The most striking difference is how they handle log entries:

```
Paxos: Same operation can have different terms
┌───────────────────────────────┐
│ Server 1: [1:A] [3:B] [3:C]  │ ← B changed from term 2 to 3
│ Server 2: [1:A] [2:B] [2:C]  │
└───────────────────────────────┘

Raft: Each operation keeps its original term forever
┌───────────────────────────────┐
│ Server 1: [1:A] [2:B] [2:C]  │ ← Terms never change
│ Server 2: [1:A] [2:B] [2:C]  │
└───────────────────────────────┘
```

## Performance Analysis

### Message Complexity

```
Scenario: 5-server cluster, electing new leader

Paxos:
- RequestVote: 4 messages
- Vote responses with logs: 4 messages (heavy payload)
- First AppendEntries: 4 messages
Total: 12 messages (with log transfers during election)

Raft:
- RequestVote: 4 messages  
- Vote responses: 4 messages (lightweight)
- First AppendEntries: 4 messages
Total: 12 messages (log transfers only after election)
```

### Election Efficiency

**Finding**: Raft's approach is surprisingly efficient despite its simplicity:
- No log entries sent during election (lighter network load)
- No log repair phase (faster to start serving)
- Trade-off: Potential vote splitting requires randomized timeouts

## Key Takeaways

1. **Both algorithms solve the same fundamental problem** with similar approaches - the differences are in the details.

2. **Paxos allows any server to become leader** but requires log repair, while **Raft only allows up-to-date servers** to become leaders.

3. **Raft's requirement for up-to-date leaders** eliminates the need for log repair and reduces network traffic during elections.

4. **For most teams**, Raft's clarity and ecosystem make it the better choice. Paxos remains valuable for specialized systems requiring maximum flexibility.

5. **The future is hybrid**: Systems like CockroachDB use Raft but incorporate optimizations from Paxos research, showing that both algorithms continue to evolve and learn from each other.