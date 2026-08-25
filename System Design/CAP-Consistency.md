# CAP Theorem and Consistency in System Design

Imagine you are developing a banking application.

A customer has ₹10,000 in their account. Your database is replicated across multiple servers:

```text
Mumbai Database:    Balance = ₹10,000
Hyderabad Database: Balance = ₹10,000
```

The customer withdraws ₹2,000 through the Mumbai database.

The new balance should be:

```text
₹10,000 - ₹2,000 = ₹8,000
```

But what happens if the Mumbai database cannot communicate with the Hyderabad database?

Should the Hyderabad server:

- Stop accepting requests until it receives the latest balance?
- Continue serving requests using the old ₹10,000 balance?

This is the kind of problem explained by the **CAP theorem**.

---

# 1. What Is the CAP Theorem?

The CAP theorem describes a limitation of distributed systems.

CAP stands for:

```text
C → Consistency
A → Availability
P → Partition tolerance
```

The CAP theorem states:

> When a network partition occurs, a distributed system must choose between consistency and availability.

A common but incomplete explanation is:

> A distributed system can provide only two out of consistency, availability, and partition tolerance.

The more accurate explanation is:

```text
When there is no network partition:
The system may provide both consistency and availability.

When a network partition occurs:
The system must choose between consistency and availability.
```

This distinction is important because the trade-off becomes unavoidable specifically during a network partition.

---

# 2. What Is a Distributed System?

A distributed system consists of multiple computers that communicate over a network and work together.

For example:

```text
                    ┌── Database Node A
Application Server ─┼── Database Node B
                    └── Database Node C
```

These database nodes may run on:

- Different servers
- Different availability zones
- Different data centres
- Different geographical regions

Using multiple nodes can improve:

- Fault tolerance
- Availability
- Read throughput
- Geographical performance
- Disaster recovery

However, distributed systems introduce an important problem:

```text
Nodes communicate over a network,
and networks can fail.
```

Messages between nodes may be:

- Delayed
- Lost
- Duplicated
- Delivered out of order
- Temporarily blocked

The CAP theorem helps us understand how a distributed system should behave during these failures.

---

# 3. Consistency in CAP

In the CAP theorem, consistency means:

> Every successful read receives the latest successful write or an error.

All clients should observe the system as if there were only one up-to-date copy of the data.

Consider two database nodes:

```text
Node A: Balance = ₹10,000
Node B: Balance = ₹10,000
```

A withdrawal updates Node A:

```text
Node A: Balance = ₹8,000
Node B: Balance = ₹10,000
```

Before serving a read from Node B, a consistent system must ensure that Node B knows about the latest update.

The client should not receive the outdated ₹10,000 balance after the ₹8,000 update has completed successfully.

A consistent result would be:

```text
Read from Node A → ₹8,000
Read from Node B → ₹8,000
```

If the system cannot guarantee the latest value, it may reject or delay the request instead of returning stale data.

> CAP consistency is usually understood as a linearizable, single-copy view of the data. It is different from the “C” in ACID transactions.

---

# 4. Availability in CAP

In CAP, availability means:

> Every request sent to a non-failing node eventually receives a response, even if the response may not contain the latest data.

Suppose Node B cannot communicate with Node A:

```text
Node A: Balance = ₹8,000
Node B: Balance = ₹10,000
```

If the system prioritizes availability, Node B continues responding:

```http
GET /balance
```

Response:

```text
Balance = ₹10,000
```

The response is stale, but the service remains available.

An availability-focused system prefers:

```text
Possibly outdated response
```

over:

```text
Request rejected or delayed indefinitely
```

CAP availability does not simply mean that the system has a high uptime percentage.

It has a more precise meaning:

```text
Every request sent to a functioning node
eventually receives a non-error response.
```

---

# 5. Partition Tolerance

A network partition occurs when nodes in a distributed system cannot communicate with each other.

For example:

```text
Mumbai Node      ✕      Hyderabad Node
               Network
               failure
```

Both nodes may still be running, but the connection between them is broken.

Possible causes include:

- Router failure
- Broken network connection
- Cloud-networking problem
- Firewall misconfiguration
- Data-centre connectivity failure
- Packet loss
- Extremely high network delay

Partition tolerance means:

> The system continues operating in some defined way even when communication between nodes is interrupted.

In a real distributed system, network failures cannot be completely prevented.

Therefore, partition tolerance is usually not optional.

When a partition occurs, the practical choice becomes:

```text
Consistency or availability?
```

---

# Understanding CAP Using a Banking Example

Consider a bank with database nodes in Mumbai and Hyderabad.

Initially:

```text
Mumbai node:    Balance = ₹10,000
Hyderabad node: Balance = ₹10,000
```

Now the connection between the nodes fails:

```text
Mumbai Node      ✕      Hyderabad Node
               Network
               partition
```

The customer withdraws ₹2,000 through the Mumbai node:

```text
Mumbai node:    Balance = ₹8,000
Hyderabad node: Balance = ₹10,000
```

Another request reaches the Hyderabad node.

What should the system do?

There are two main choices.

---

## Choice 1: Prioritize Consistency

The Hyderabad node knows that it may not have the latest balance.

Therefore, it refuses or delays the request:

```http
HTTP/1.1 503 Service Unavailable
```

This prevents the user from seeing or modifying an incorrect balance.

The result is:

```text
Consistency maintained
Availability reduced
Partition tolerated
```

This is commonly described as a **CP system**.

---

## Choice 2: Prioritize Availability

The Hyderabad node continues accepting requests and responds using its local data:

```text
Balance = ₹10,000
```

The service remains responsive, but the user may receive stale information.

The result is:

```text
Availability maintained
Immediate consistency sacrificed
Partition tolerated
```

This is commonly described as an **AP system**.

---

# 6. CAP Combinations

## CP: Consistency and Partition Tolerance

A CP system prioritizes correct and up-to-date data during a network partition.

If a node cannot guarantee that its data is current, it may reject or delay the request.

```text
Network partition occurs
          ↓
Node cannot verify latest value
          ↓
Request is rejected or delayed
          ↓
Consistency is protected
```

### Suitable use cases

CP behaviour is useful when stale or conflicting data could be dangerous:

- Banking balances
- Payment processing
- Inventory reservations
- Distributed locks
- Leader election
- Unique username allocation
- Critical configuration management

### Example: Movie-ticket booking

Suppose only one movie ticket remains:

```text
Available seats = 1
```

During a network partition, two nodes receive booking requests:

```text
Node A books the seat for Charan
Node B books the same seat for Rahul
```

If both nodes accept the request independently, the seat is double-booked.

A consistency-focused system may allow only the node with confirmed authority—such as the current leader or quorum—to accept the booking.

Other nodes reject or delay the request.

### Trade-off

```text
Advantage:
Clients do not receive conflicting or stale results.

Disadvantage:
Some requests fail or wait during a partition.
```

---

## AP: Availability and Partition Tolerance

An AP system continues serving requests during a network partition.

Different nodes may temporarily contain different values.

```text
Network partition occurs
          ↓
Both sides continue accepting requests
          ↓
Temporary inconsistency appears
          ↓
Data is reconciled later
```

### Suitable use cases

AP behaviour may be suitable when temporary inconsistency is acceptable:

- Social-media likes
- Product reviews
- View counters
- User-activity feeds
- Shopping-cart contents
- Analytics events
- Logging systems
- Some DNS operations

### Example: Like counter

Suppose a post has:

```text
Likes = 100
```

During a partition:

```text
Node A receives 5 likes → 105
Node B receives 3 likes → 103
```

Both nodes remain available.

After connectivity is restored, the updates can be merged:

```text
Final likes = 108
```

Temporary inconsistency is generally acceptable for a like counter.

### Trade-off

```text
Advantage:
The system continues serving requests.

Disadvantage:
Clients may temporarily receive stale or conflicting data.
```

---

## CA: Consistency and Availability

A CA system provides consistency and availability when there is no network partition.

For example, a single database server may provide:

```text
Consistent reads
Available responses
```

But if the database server fails:

```text
Application → Database unavailable
```

The system cannot tolerate that failure.

In a truly distributed system, network partitions must be expected. Therefore, CA is not a meaningful choice during a partition.

A single-node relational database is often used as an intuitive CA example, but CAP primarily concerns replicated distributed systems.

---

# Important Correction: CAP Does Not Mean Choosing Two Forever

A common misunderstanding is:

```text
Choose any two:
C, A or P
```

That is too simplistic.

The actual situation is:

```text
Normal operation:
The system may provide both consistency and availability.

During a network partition:
The system must choose which one to preserve.
```

Therefore:

```text
No partition → C and A may both be provided

Partition    → Choose C or A
```

The choice may also be different for different operations.

For example, an e-commerce system could use:

```text
Product descriptions → Prefer availability
Product reviews      → Prefer availability
Inventory checkout   → Prefer consistency
Payment processing   → Prefer consistency
```

The complete application does not have to make one universal CAP choice.

---

# 7. What Is Consistency?

Consistency describes what users are allowed to observe after data has been written.

Suppose:

```text
Initial username = "charan"
```

The user updates it:

```text
New username = "charan_avvaru"
```

If another request immediately reads the user profile, what should it receive?

```text
Old value = "charan"
```

or:

```text
New value = "charan_avvaru"
```

The answer depends on the consistency model used by the system.

Consistency is not simply “consistent” or “inconsistent.”

There are multiple consistency models, each offering different guarantees.

---

# 8. Consistency Models

## 8.1 Strong Consistency

With strong consistency, once a write succeeds, every subsequent read returns the latest value.

```text
Write: balance = ₹8,000
           ↓
Write succeeds
           ↓
Every later read returns ₹8,000
```

### Example

```text
Initial inventory = 1

User A purchases the item
Inventory becomes 0

User B reads inventory
Result = 0
```

User B should never see the old value of `1` after User A’s update has successfully completed.

### Advantages

- Easier for developers to reason about
- Users receive current data
- Prevents many conflicting operations
- Suitable for correctness-critical operations

### Disadvantages

- May require coordination between replicas
- Can increase latency
- Can reduce availability during network failures
- Geographically distributed writes may be slower

### Suitable use cases

- Bank-account balances
- Payment status
- Inventory reservation
- Distributed locking
- Unique identifier allocation
- Critical security settings

---

## 8.2 Eventual Consistency

With eventual consistency, replicas may temporarily contain different values.

If no new updates occur, all replicas eventually converge to the same value.

```text
Write happens on Node A
          ↓
Node A contains new value
Node B temporarily contains old value
          ↓
Replication occurs
          ↓
Both nodes contain new value
```

### Example

A user changes their profile picture.

Immediately afterwards:

```text
Phone app:       New picture
Desktop website: Old picture
```

After a few seconds:

```text
Phone app:       New picture
Desktop website: New picture
```

The system was temporarily inconsistent but eventually became consistent.

### Advantages

- Better availability
- Often lower latency
- Nodes can continue operating during some failures
- Works well across multiple regions

### Disadvantages

- Users may temporarily see stale data
- Conflicting writes must be resolved
- Application logic becomes more complicated
- Debugging can be harder

### Suitable use cases

- Social-media feeds
- Likes and view counters
- Product reviews
- User-profile updates
- Analytics
- DNS
- Some shopping-cart operations

---

## 8.3 Read-Your-Writes Consistency

Read-your-writes consistency guarantees that after a user updates something, the same user sees the updated value in later reads.

Suppose Charan updates his profile:

```text
Old city = Hyderabad
New city = Bengaluru
```

His next profile request should return:

```text
City = Bengaluru
```

Other users may temporarily see the old value, but Charan should see his own latest update.

### How it can be implemented

Common approaches include:

- Route the user back to the same replica
- Read from the primary after a write
- Store a version number in the session
- Wait until a replica has applied the required version
- Temporarily serve the updated value from a cache

### Suitable use cases

- Profile updates
- Creating posts
- Updating settings
- Adding comments
- Editing documents

Without this guarantee, a user might save a change, refresh the page, and think the update was lost.

---

## 8.4 Monotonic-Read Consistency

Monotonic reads guarantee that once a user has observed a particular version of data, they will never later observe an older version.

Without monotonic reads:

```text
First request  → Version 5
Second request → Version 3
```

This makes it appear as though the system has moved backwards.

With monotonic reads:

```text
First request  → Version 5
Second request → Version 5 or newer
```

### Example

Suppose a user reads:

```text
Order status = Shipped
```

After refreshing the page, the user should not see:

```text
Order status = Processing
```

This problem can occur if the two requests reach replicas at different replication positions.

### Suitable use cases

- Order tracking
- Message history
- Activity feeds
- Versioned documents
- User notifications

---

## 8.5 Monotonic-Write Consistency

Monotonic writes guarantee that writes from the same client are applied in the order in which they were submitted.

Suppose a user performs these updates:

```text
1. Order status = Paid
2. Order status = Shipped
```

The system must not apply them in this order:

```text
1. Order status = Shipped
2. Order status = Paid
```

Otherwise, the final state incorrectly moves backwards.

### Suitable use cases

- Order-status updates
- User-setting changes
- Document editing
- Sequential workflow operations

---

## 8.6 Causal Consistency

Causal consistency preserves the order of operations that are causally related.

Suppose:

```text
1. Charan creates a post:
   "I got a new job!"

2. Rahul replies:
   "Congratulations!"
```

The reply depends on the original post.

A causally consistent system should never show:

```text
"Congratulations!"
```

before showing:

```text
"I got a new job!"
```

Unrelated operations may still appear in different orders.

Causal consistency is stronger than eventual consistency but generally weaker than full strong consistency.

### Suitable use cases

- Social-media comments
- Messaging applications
- Collaborative editing
- Activity streams
- Discussion threads

---

## 8.7 Consistent-Prefix Reads

Consistent-prefix reads guarantee that clients observe events in a valid order without skipping earlier events.

Suppose an order moves through these states:

```text
1. Order created
2. Payment completed
3. Order shipped
4. Order delivered
```

A user should not observe:

```text
Order delivered
```

without the earlier events logically existing.

Consistent-prefix reads prevent users from observing impossible sequences.

---

## 8.8 Session Consistency

Session consistency provides consistency guarantees within a particular user session.

Within the same session, the user may receive:

- Read-your-writes consistency
- Monotonic reads
- Monotonic writes

Different sessions may temporarily observe different versions of the data.

Session consistency provides a useful balance for many user-facing applications.

---

# Strong Consistency vs Eventual Consistency

| Area | Strong consistency | Eventual consistency |
|---|---|---|
| Read result | Latest successful write | May temporarily return stale data |
| Availability during partitions | May reject requests | Usually continues serving requests |
| Latency | Potentially higher | Often lower |
| Replica coordination | More coordination | Less immediate coordination |
| Application logic | Easier to reason about | Conflict handling may be complex |
| Typical use | Payments and inventory | Feeds, likes and analytics |

---

# Example: E-Commerce Application

A single application can use different consistency models for different features.

## Product description

If a seller updates a product description, some users seeing the old description for a few seconds may be acceptable.

```text
Preferred model: Eventual consistency
```

## Product inventory

If only one phone remains, two customers should not both purchase it.

```text
Preferred model: Strong consistency
```

## Shopping cart

A shopping cart should remain accessible even if one region experiences a temporary problem.

Temporary merging may be acceptable.

```text
Preferred model: Eventual or session consistency
```

## Payment

A successful payment should not appear unpaid to another critical service.

```text
Preferred model: Strong consistency
```

## Product view count

It is acceptable if replicas briefly show slightly different counts.

```text
Preferred model: Eventual consistency
```

The important lesson is:

> Choose consistency requirements per operation and data type, not merely per application.

---

# 9. Quorums

Distributed databases often use quorums to balance consistency and availability.

Assume there are three replicas:

```text
N = 3 replicas
```

Let:

```text
W = Number of replicas that must acknowledge a write
R = Number of replicas consulted during a read
```

A commonly discussed rule is:

```text
R + W > N
```

This creates an overlap between the read and write replica sets, increasing the chance that a read sees the latest write.

For example:

```text
N = 3
W = 2
R = 2

R + W = 4
4 > 3
```

A write must succeed on two replicas, and a read checks two replicas.

Because the sets overlap, at least one read replica should contain the latest acknowledged write—assuming version comparison and the database’s other guarantees work correctly.

---

## Faster Writes

```text
N = 3
W = 1
R = 3
```

Only one replica must acknowledge the write, but reads consult all three replicas.

```text
Advantage:
Lower write latency

Disadvantage:
Higher read latency
```

---

## Faster Reads

```text
N = 3
W = 3
R = 1
```

All replicas acknowledge the write, allowing a read from one replica.

```text
Advantage:
Lower read latency

Disadvantage:
Higher write latency and lower write availability
```

---

## Balanced Approach

```text
N = 3
W = 2
R = 2
```

Reads and writes both contact a majority of replicas.

> The formula `R + W > N` does not automatically guarantee linearizable consistency. Concurrent operations, failed nodes, sloppy quorums, clocks, version handling and database implementation details also matter.

---

# 10. Conflict Resolution

In an eventually consistent system, two nodes may accept conflicting writes during a partition.

Suppose:

```text
Original city = Hyderabad
```

During a partition:

```text
Node A changes city to Bengaluru
Node B changes city to Mumbai
```

When connectivity returns, the system must decide which value to keep.

Common conflict-resolution strategies include the following.

---

## 10.1 Last Write Wins

The write with the latest timestamp is kept.

```text
10:00:01 → Bengaluru
10:00:02 → Mumbai

Final value → Mumbai
```

This is simple, but clock differences and lost updates can cause problems.

---

## 10.2 Application-Level Resolution

The application decides how the values should be merged.

For example, consider a shopping cart:

```text
Node A cart = [Laptop]
Node B cart = [Mouse]

Merged cart = [Laptop, Mouse]
```

This strategy works when the business rules provide a meaningful way to combine updates.

---

## 10.3 Version Numbers and Vector Clocks

The system tracks relationships between different versions.

This can help determine whether:

- One update happened before another
- One version is newer
- Two updates happened concurrently
- A conflict needs to be resolved

Vector clocks can detect concurrent changes but add implementation complexity.

---

## 10.4 Conflict-Free Replicated Data Types

CRDTs are data structures designed so that updates from different nodes can be merged predictably without central coordination.

Examples include:

- Counters
- Sets
- Registers
- Maps

CRDTs can be useful for:

- Distributed counters
- Collaborative applications
- Offline-first applications
- Replicated shopping carts

---

# 11. CAP Consistency vs ACID Consistency

The word **consistency** appears in both CAP and ACID, but it has different meanings.

## Consistency in CAP

CAP consistency asks:

> Do all clients observe the latest successful write as if there were only one copy of the data?

It concerns the view of replicated data across multiple nodes.

## Consistency in ACID

ACID consistency asks:

> Does a transaction preserve the database’s defined rules and constraints?

Examples include:

- A balance cannot violate a defined rule
- Foreign-key relationships must remain valid
- Email addresses must remain unique
- Required fields cannot be null
- Inventory cannot become negative if the database prevents it

Therefore:

```text
CAP consistency
= A consistent view across distributed replicas

ACID consistency
= Database rules remain valid before and after transactions
```

These concepts are related to correctness, but they describe different guarantees.

---

# 12. CAP and PACELC

CAP mainly explains what happens during a network partition.

Distributed systems also make trade-offs when the network is operating normally.

PACELC extends CAP:

```text
If there is a Partition:
Choose between Availability and Consistency

Else:
Choose between Latency and Consistency
```

It can be remembered as:

```text
PACELC

P  → A or C
EL → L or C
```

---

## PACELC Example

Suppose a write occurs in India and the system has another replica in Europe.

### Stronger consistency

```text
Write in India
      ↓
Wait for replica in Europe
      ↓
Return success
```

Waiting for the European replica improves consistency, but it increases latency.

### Lower latency

```text
Write in India
      ↓
Return success immediately
      ↓
Replicate to Europe later
```

This reduces latency but allows temporary inconsistency.

PACELC reminds us that the trade-off continues during normal operation:

```text
Lower latency
versus
Stronger consistency
```

---

# 13. Practical System-Design Example

Suppose you are designing a multi-region task-management application.

The system contains:

```text
India Region
├── API servers
└── Database replica

Europe Region
├── API servers
└── Database replica
```

You might choose different consistency guarantees for different operations:

| Operation | Desired behaviour | Reason |
|---|---|---|
| Create task | Read-your-writes | Creator should immediately see the task |
| Edit task | Causal or session consistency | Changes should appear in a sensible order |
| Assign task | Stronger consistency | Avoid conflicting assignments |
| View activity count | Eventual consistency | Temporary differences are acceptable |
| Pay for subscription | Strong consistency | Duplicate or incorrect payments are dangerous |
| Record analytics event | High availability | Temporary processing delay is acceptable |

This is more realistic than simply calling the entire application “CP” or “AP.”

---

# 14. How to Choose a Consistency Model

Ask the following questions for every important operation.

## 1. Can stale data cause financial loss?

If yes, prefer stronger consistency.

Examples:

- Payments
- Account balances
- Inventory reservations

## 2. Can conflicting writes be merged safely?

If yes, eventual consistency may work.

Examples:

- Like counters
- Shopping carts
- Analytics events

## 3. What is worse: rejecting the request or returning stale data?

For payments:

```text
Returning incorrect data may be worse.
```

For a social-media feed:

```text
Temporarily showing an older feed may be acceptable.
```

## 4. Does the user need to see their own update immediately?

If yes, provide read-your-writes consistency.

## 5. Must operations remain in order?

If yes, consider:

- Monotonic writes
- Causal consistency
- Consistent-prefix reads
- Stronger coordination

## 6. How much additional latency is acceptable?

Cross-region coordination can improve consistency but increase response time.

## 7. What happens during a partition?

Decide whether the operation should:

- Fail or wait to protect correctness
- Continue using potentially stale data
- Accept writes and reconcile them later

---

# 15. Common Misunderstandings

## Misunderstanding 1: A system always chooses only two CAP properties

The choice between consistency and availability becomes unavoidable during a network partition.

Without a partition, the system may provide both.

---

## Misunderstanding 2: Partition tolerance can be avoided

A replicated distributed system communicates through a network.

Network failures must be expected.

---

## Misunderstanding 3: Availability means high uptime

CAP availability has a precise meaning:

```text
Every request to a functioning node
eventually receives a response.
```

Operational availability, such as 99.99% uptime, is related but not identical.

---

## Misunderstanding 4: Eventual consistency means data is always wrong

Eventual consistency means replicas may temporarily differ but eventually converge when updates stop and replication succeeds.

---

## Misunderstanding 5: Every feature needs strong consistency

Strong consistency has costs.

Many features can safely use weaker consistency guarantees.

---

## Misunderstanding 6: An entire application must be CP or AP

Different services, operations and data types can make different decisions.

For example:

```text
Payment operation → Consistency first
Product feed      → Availability first
```

---

## Misunderstanding 7: Replication automatically provides consistency

Replication creates multiple copies of data, but those copies may temporarily differ.

The replication protocol determines the consistency guarantee.

---

# Easy Way to Remember CAP

Imagine two shop branches with separate copies of inventory data:

```text
Branch A       Branch B
Stock = 1      Stock = 1
```

Their network connection breaks.

A customer tries to buy the item from each branch.

## Consistency-first decision

One branch stops selling until it can confirm the latest inventory.

```text
No double sale
But some customers cannot purchase
```

## Availability-first decision

Both branches continue selling.

```text
Customers can purchase
But the same final item may be sold twice
```

The network partition forces the decision:

```text
Protect correctness
or
Continue serving every request
```

---

# Quick Comparison

| CAP property | Meaning | Question |
|---|---|---|
| Consistency | Reads observe the latest successful write | Is the data current and uniform? |
| Availability | Functioning nodes respond to requests | Can the request still be served? |
| Partition tolerance | The system handles broken communication | Can it operate despite network separation? |

---

# Consistency Model Comparison

| Consistency model | Guarantee | Example |
|---|---|---|
| Strong consistency | Every later read sees the latest successful write | Bank balance |
| Eventual consistency | Replicas eventually converge | Like counter |
| Read-your-writes | A user sees their own updates | Profile update |
| Monotonic reads | A user never sees an older version after a newer one | Order tracking |
| Monotonic writes | A user’s writes are applied in order | Status updates |
| Causal consistency | Related events remain logically ordered | Post and reply |
| Consistent prefix | Events are observed in a valid sequence | Order lifecycle |
| Session consistency | Guarantees are maintained within one session | User-facing application |

---

# Final Summary

The CAP theorem says:

> When a network partition occurs, a distributed system must choose between consistency and availability.

```text
CP system:
Preserve consistency
Reject or delay some requests during a partition

AP system:
Preserve availability
Allow temporary inconsistency during a partition
```

Consistency itself has multiple levels:

```text
Strong consistency
→ Every later read sees the latest successful write

Eventual consistency
→ Replicas may temporarily differ but eventually converge

Read-your-writes
→ A user sees their own updates

Monotonic reads
→ A user never moves back to an older version

Causal consistency
→ Related events remain in logical order
```

The correct choice depends on the operation:

```text
Payment             → Strong consistency
Inventory booking   → Strong consistency
Profile update      → Read-your-writes
Social-media likes  → Eventual consistency
Analytics events    → Eventual consistency
```

The most important system-design lesson is:

> Do not ask whether the entire application should be consistent or available. Ask what each operation requires when communication fails, what stale data would cost, and whether conflicts can be safely resolved.
