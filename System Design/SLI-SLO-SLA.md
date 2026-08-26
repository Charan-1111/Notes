# SLA, SLO, and SLI — Explained Simply

SLA, SLO, and SLI sound confusing initially because their names are very similar.

The easiest way to remember them is:

> **SLI = What are we measuring?**  
> **SLO = What target are we trying to achieve?**  
> **SLA = What have we promised the customer?**

In short:

```text
SLI → SLO → SLA

Measure → Target → Promise
```

---

# 1. Start With a Simple Scenario

Suppose your team owns a **Payment API**.

```text
Client
   |
   | POST /payments
   v
+------------------+
|   Payment API    |
+------------------+
   |
   v
 Database
```

Customers expect this API to:

- Be available
- Respond quickly
- Not fail frequently

But saying:

> "Our API should be reliable."

isn't enough.

We need measurable numbers.

This is where **SLI, SLO, and SLA** come into the picture.

---

# 2. SLI — Service Level Indicator

**SLI = Service Level Indicator**

An SLI is a **measurement of how your system is actually behaving**.

Think of it as:

> **"What number tells me how my service is performing?"**

Examples:

```text
Availability = 99.95%

p95 Latency = 180ms

Error Rate = 0.2%
```

These are examples of SLIs.

---

## Example: Availability SLI

Suppose your API received:

```text
Total requests      = 1,000,000
Successful requests =   999,500
```

We can calculate availability as:

```text
Availability =
Successful Requests / Total Requests × 100
```

Therefore:

```text
999,500 / 1,000,000 × 100

= 99.95%
```

So your actual availability is:

```text
SLI = 99.95%
```

The important thing to understand is:

> **99.95% here is NOT a target or promise. It is what actually happened.**

---

# 3. Common SLIs in Backend Systems

As a backend engineer, you will commonly see SLIs such as:

| SLI | What It Measures |
|---|---|
| Availability | Is the service usable? |
| Latency | How quickly does the service respond? |
| Error Rate | How frequently do requests fail? |
| Throughput | How much traffic is processed? |
| Durability | Is stored data preserved? |
| Correctness | Are requests producing correct results? |

For example, your monitoring system might report:

```text
Availability: 99.97%

p50 latency: 70ms

p95 latency: 180ms

p99 latency: 450ms

5xx error rate: 0.03%
```

These are measurements of the actual service.

Therefore:

> **SLI = Reality**

---

# 4. SLO — Service Level Objective

Now your engineering team asks:

> **"What level of reliability do we want to maintain?"**

This is where the **SLO** comes in.

**SLO = Service Level Objective**

An SLO is the **target value for an SLI over a defined period of time**.

For example:

```text
SLI:
Availability

SLO:
Availability >= 99.9% over 30 days
```

Another example:

```text
SLI:
Request latency

SLO:
99% of requests should complete within 300ms
```

Another:

```text
SLI:
Error rate

SLO:
Less than 0.1% of requests should result in server errors
```

---

# 5. SLI vs SLO

This distinction is extremely important.

```text
SLI = 99.95% availability
      ↑
      Actual measurement


SLO = 99.9% availability
      ↑
      Target
```

So:

> **SLI tells you where you are.**

> **SLO tells you where you want to be.**

---

# 6. Backend Example

Suppose you're responsible for this architecture:

```text
               Internet
                  |
                  v
           +-------------+
           |    Load     |
           |  Balancer   |
           +-------------+
                  |
        +---------+---------+
        |                   |
        v                   v
+---------------+   +---------------+
| Go API        |   | Go API        |
| Instance 1    |   | Instance 2    |
+---------------+   +---------------+
        |                   |
        +---------+---------+
                  |
                  v
            PostgreSQL
```

Your engineering team defines the following SLO:

```text
Availability SLO

99.9% of valid requests should
successfully complete every month.
```

At the end of the month, your monitoring system reports:

```text
Actual availability = 99.96%
```

Therefore:

```text
SLI = 99.96%
SLO = 99.9%
```

Since:

```text
99.96% > 99.9%
```

the SLO was achieved.

---

## What If Availability Drops?

Suppose next month:

```text
SLI = 99.72%

SLO = 99.9%
```

Now:

```text
99.72% < 99.9%
```

Therefore:

```text
SLO MISSED
```

The service did not meet its reliability objective.

---

# 7. SLA — Service Level Agreement

Now we move from **engineering targets** to **business/customer commitments**.

**SLA = Service Level Agreement**

An SLA is an agreement between the **service provider and customer** describing a promised service level.

There may also be consequences if the promise isn't met.

For example, your company might tell customers:

> **"We guarantee at least 99.9% monthly availability."**

The agreement could say:

```text
Availability >= 99.9%
        |
        v
Everything is fine


Availability < 99.9%
        |
        v
Customer receives service credits
```

That is an **SLA**.

---

# 8. Example SLA

A hypothetical SLA could look like:

```text
Monthly Availability

>= 99.9%
    No compensation

99.0% - 99.9%
    10% service credit

95.0% - 99.0%
    25% service credit

< 95%
    50% service credit
```

The exact numbers and consequences depend on the company and contract.

The important idea is:

> **SLA = Customer Promise / Agreement**

---

# 9. Putting SLI, SLO, and SLA Together

Imagine our Payment API.

## SLI

Monitoring tells us:

```text
Actual availability = 99.96%
```

This is the measurement.

## SLO

Our engineering team targets:

```text
Availability >= 99.95%
```

This is our internal reliability objective.

## SLA

Our customer contract promises:

```text
Availability >= 99.9%
```

If availability falls below this level, customers may receive service credits.

So we have:

```text
                    Reliability

Actual              Engineering             Customer
Measurement         Target                  Commitment

   SLI                  SLO                     SLA
    |                    |                       |
    v                    v                       v

 99.96%                99.95%                  99.9%
```

---

# 10. Why Can SLO Be Higher Than SLA?

You might notice:

```text
SLO = 99.95%

SLA = 99.9%
```

Why is the engineering target higher than the customer promise?

Because we want a **safety margin**.

If engineering also targeted exactly:

```text
99.9%
```

then even a small incident could result in an SLA violation.

Instead, engineering might target:

```text
Internal SLO
    |
    v

99.95%
```

Giving us some buffer:

```text
Internal Reliability Target
           |
           v
         99.95%
           |
           |
        BUFFER
           |
           v
          99.9%
           |
           v
Customer Commitment
```

---

# 11. Simple Real-World Analogy

Imagine a food delivery company.

## SLI

Actual measurement:

```text
Average delivery time = 27 minutes
```

This is what actually happened.

## SLO

Internal company target:

```text
Deliver 95% of orders within 30 minutes.
```

This is what the company wants to achieve.

## SLA

Customer promise:

```text
Delivery within 40 minutes.

Otherwise:

₹100 credit
```

This is what the company promised customers.

Therefore:

```text
SLI
"What actually happened?"

        |
        v

SLO
"What are we aiming for?"

        |
        v

SLA
"What did we promise?"
```

---

# 12. Connection With Monitoring

SLIs become extremely important when building production systems.

Suppose your monitoring architecture looks like:

```text
+----------------+
|   Go Backend   |
+----------------+
        |
        | Metrics
        v
+----------------+
|   Prometheus   |
+----------------+
        |
        v
+----------------+
|    Grafana     |
+----------------+
```

Your Go application might expose metrics such as:

```text
http_requests_total

http_request_duration_seconds

http_requests_failed_total
```

Using these metrics, you can calculate SLIs.

For example:

```text
                   Successful Requests
Availability SLI = -------------------
                      Total Requests
```

Grafana might then visualize your availability:

```text
Availability

100% | ───────────────────
     |
99.9%|-------------------- SLO
     |          \
     |           \
99.8%|            \____
     |
     +-------------------------
                Time
```

If availability starts falling below the SLO, alerts can be triggered.

---

# 13. SLO and Error Budgets

One of the most useful concepts associated with SLOs is the:

## Error Budget

Suppose your SLO is:

```text
99.9% availability
```

This means you're accepting:

```text
100% - 99.9%

= 0.1%
```

unavailability.

That **0.1% is your error budget**.

---

# 14. Calculating Error Budget

Suppose we consider a 30-day month.

```text
30 days
```

Convert that to minutes:

```text
30 × 24 × 60

= 43,200 minutes
```

Your allowed failure percentage is:

```text
0.1%
```

Therefore:

```text
43,200 × 0.001

= 43.2 minutes
```

So:

```text
SLO = 99.9%

Allowed unavailability
        |
        v

~43 minutes per month
```

---

# 15. Consuming the Error Budget

Suppose your system experiences:

```text
10 minutes downtime
```

You have consumed part of your error budget.

If downtime becomes:

```text
40 minutes
```

You're very close to exhausting the budget.

If downtime becomes:

```text
60 minutes
```

You've exceeded a simple time-based 99.9% error budget.

This can affect engineering decisions.

```text
Plenty of Error Budget
        |
        v
Can take more deployment/change risk
        |
        v
Ship features
Experiment
Deploy frequently
```

But:

```text
Error Budget Almost Exhausted
        |
        v
Reduce risky deployments
        |
        v
Focus on reliability
        |
        v
Fix incidents
Improve monitoring
Improve failover
Improve testing
```

This creates a balance between:

```text
Feature Development

        VS

System Reliability
```

---

# 16. Understanding the "Nines"

In system design, you'll often hear engineers say:

```text
Two nines

Three nines

Four nines

Five nines
```

These refer to availability.

Approximately, for a 30-day month:

| Availability | Approximate Downtime |
|---:|---:|
| 99% | ~7 hours 12 minutes |
| 99.9% | ~43 minutes |
| 99.99% | ~4 minutes 19 seconds |
| 99.999% | ~26 seconds |

So:

```text
99%
 |
 v
99.9%
 |
 v
99.99%
 |
 v
99.999%
```

Each additional nine becomes significantly harder to achieve.

---

# 17. Why Higher Availability Is Difficult

If you want very high availability such as:

```text
99.999%
```

you may need to think carefully about:

- Multiple application instances
- Load balancing
- Health checks
- Automatic failover
- Database replication
- Database failover
- Multiple availability zones
- Multi-region architecture
- Retry mechanisms
- Circuit breakers
- Graceful degradation
- Disaster recovery
- Deployment safety
- Rollbacks
- Observability
- Monitoring
- Alerting
- Dependency failures

Therefore:

> **Higher availability usually means higher system complexity and higher cost.**

This is why choosing an SLO is a **business + engineering trade-off**.

You shouldn't simply say:

> "Let's make everything 99.999% available."

You need to ask:

```text
Does the business actually need it?

        +

Can we justify the engineering cost?
```

---

# 18. Important Misconception

Don't think:

```text
SLI = Availability

SLO = Latency

SLA = Error Rate
```

That is incorrect.

An **SLI is a measurement**, while an **SLO is a target for that measurement**.

For example:

## Availability

```text
SLI:
Actual availability = 99.96%

SLO:
Availability >= 99.95%
```

## Latency

```text
SLI:
Actual p99 latency = 350ms

SLO:
99% of requests should complete within 500ms
```

## Error Rate

```text
SLI:
Actual error rate = 0.03%

SLO:
Error rate should remain below 0.1%
```

So one service can have **multiple SLIs and SLOs**.

---

# 19. Complete Production Example

Imagine you're operating an e-commerce backend.

```text
                     Users
                       |
                       v
                Load Balancer
                       |
             +---------+---------+
             |                   |
             v                   v
         Go API 1            Go API 2
             |                   |
             +---------+---------+
                       |
                       v
                  PostgreSQL
                       |
                       v
                     Redis
```

Your engineering team might define:

## Availability

```text
SLI:
Actual availability = 99.97%

SLO:
Availability >= 99.95%
```

## Latency

```text
SLI:
Actual p99 latency = 380ms

SLO:
99% of requests < 500ms
```

## Error Rate

```text
SLI:
Actual error rate = 0.04%

SLO:
Error rate < 0.1%
```

## Customer SLA

```text
SLA:

Monthly availability >= 99.9%

If availability drops below 99.9%,
eligible customers may receive service credits.
```

---

# 20. Final Mental Model

The easiest way to remember everything is:

```text
                 SERVICE RELIABILITY
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
         SLI           SLO           SLA

      Indicator      Objective     Agreement

          |             |             |
          v             v             v

       Measure        Target        Promise

          |             |             |
          v             v             v

     What actually   What should    What did we
      happen?         happen?        promise?
```

Or simply:

```text
SLI → MEASURE

SLO → TARGET

SLA → PROMISE
```

---

# 21. Interview-Friendly Answer

If an interviewer asks:

> **"What is the difference between SLI, SLO, and SLA?"**

You can answer:

**SLI (Service Level Indicator)** is the actual measurable performance of a service, such as availability, latency, or error rate.

**SLO (Service Level Objective)** is the target value we want an SLI to achieve over a specific period.

**SLA (Service Level Agreement)** is the customer-facing agreement that defines the promised service level and may include consequences such as service credits when the commitment isn't met.

For example:

```text
Payment API

SLI:
Actual availability = 99.96%

SLO:
Internal target >= 99.95%

SLA:
Customer commitment >= 99.9%
```

So the easiest way to remember it is:

> **SLI = Measure**  
> **SLO = Target**  
> **SLA = Promise**
> 
