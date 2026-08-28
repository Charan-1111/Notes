# Model Selection and Cost/Latency Trade-offs in AI Systems

When building an AI application, one of the most important engineering decisions is:

> **Which model should handle this request?**

Choosing the "best", largest, or smartest model for everything is usually **not the right approach**.

A production AI system usually needs to balance:

**Quality + Cost + Latency + Reliability + Capability**

---

# 1. What is Model Selection?

Suppose you're building a customer-support AI.

One user asks:

> "What are your business hours?"

Another user asks:

> "Read these logs and deployment information and determine why our production service latency increased."

These requests have completely different difficulty levels.

Using your most capable model for both is similar to hiring a senior distributed-systems engineer to answer:

> "What time does the office open?"

It works, but it's unnecessarily expensive.

**Model selection** means choosing the model that provides sufficient quality for a particular task while satisfying your cost, latency, reliability, and capability requirements.

Conceptually:

    Request
       │
       ▼
    What does this task require?
       │
       ├── Simple classification ──► Small/Fast model
       │
       ├── Simple Q&A ─────────────► Small/Fast model
       │
       ├── Extraction ─────────────► Small/Medium model
       │
       ├── Coding ─────────────────► Capable coding model
       │
       ├── Complex reasoning ──────► Reasoning model
       │
       └── Image understanding ────► Multimodal model

So model selection isn't simply:

    Which model has the highest benchmark score?

Instead:

    Which model is good enough for THIS task
    while meeting my production requirements?

---

# 2. The Core Trade-off

Imagine three hypothetical models:

| Model | Quality | Latency | Cost |
|---|---|---|---|
| Small | Good | Very low | $ |
| Medium | Better | Medium | $$ |
| Large/Reasoning | Excellent | Higher | $$$$ |

It can be tempting to say:

> "Use the large model because it gives better answers."

But consider an application processing:

    10 million requests/month

Suppose:

    Model A

    Average cost/request = $0.0005

Then:

    10M requests
    = $5,000/month

Now suppose:

    Model B

    Average cost/request = $0.005

Then:

    10M requests
    = $50,000/month

If Model B improves task success from:

    95% → 95.5%

you need to determine whether that extra 0.5 percentage points is worth:

    $45,000 additional cost/month

That's an **engineering and business decision**, not merely a model-quality decision.

---

# 3. Think in Multiple Dimensions

Model selection isn't just:

    Which model is smarter?

You usually need to consider:

1. Quality
2. Cost
3. Latency
4. Context window
5. Input/output modalities
6. Tool/function calling
7. Structured-output reliability
8. Rate limits
9. Throughput
10. Availability/reliability
11. Safety/security requirements

A useful mental model is:

                        QUALITY
                           ▲
                          / \
                         /   \
                        /     \
                       /       \
                      /         \
                     /           \
                  COST -------- LATENCY

You're trying to find a model that provides an acceptable balance between them.

---

# 4. Quality

Quality asks:

> How reliably does this model perform my actual task?

Notice the important part:

**My actual task.**

A model might be excellent at mathematical reasoning but unnecessary for simple information extraction.

For example:

    Hi, I'm Rahul.
    My order number is ORD-83921.
    Please cancel my order.

You want:

    {
      "intent": "cancel_order",
      "order_id": "ORD-83921"
    }

This is a relatively simple extraction/classification task.

You probably don't need your most expensive reasoning model.

But suppose the request is:

    Here are:

    - application logs
    - database metrics
    - deployment history
    - architecture documentation

    Determine why latency increased after yesterday's deployment.

Now stronger reasoning capabilities may provide substantial value.

So:

    Simple task
         ↓
    Small/Fast Model

    Complex task
         ↓
    More Capable Model

---

# 5. Cost

API-based models commonly charge based on usage.

A simplified calculation is:

    Cost =
        input_tokens × input_price
        +
        output_tokens × output_price

Suppose your request contains:

    Input  = 2,000 tokens
    Output = 500 tokens

Assume hypothetical pricing:

    Input  = $1 / 1M tokens
    Output = $5 / 1M tokens

Input cost:

    2,000 / 1,000,000 × $1

    = $0.002

Output cost:

    500 / 1,000,000 × $5

    = $0.0025

Total:

    $0.0045/request

That sounds tiny.

But imagine:

    10,000,000 requests

Then:

    $0.0045 × 10,000,000

    = $45,000

This is why AI engineers care about:

- token usage
- prompt size
- output size
- model selection
- caching
- routing

Small inefficiencies become expensive at scale.

---

# 6. Output Tokens Can Be Expensive

A common beginner mistake is focusing only on prompt size.

Suppose you send:

    Explain HTTP.

The input is tiny.

But your system prompt says:

    Always respond with extremely detailed
    10,000-word explanations.

Now the generated output could become expensive.

So cost optimization involves both:

    Input optimization
            +
    Output optimization

For example, instead of:

    Explain this support ticket in extreme detail.

Perhaps your application only needs:

    Return:

    {
      "category": "...",
      "priority": "...",
      "summary": "maximum 30 words"
    }

This can significantly reduce unnecessary token usage.

---

# 7. Latency

Latency means:

> How long does the user wait for the AI system?

Imagine:

    User
     │
     │ "Summarize this"
     ▼
    Backend
     │
     │ LLM API call
     ▼
    Model
     │
     │ Generation
     ▼
    Backend
     │
     ▼
    User

If the entire process takes:

    8 seconds

then the end-to-end latency is approximately:

    8 seconds

But LLM latency isn't just one number.

Two measurements are particularly useful.

---

# 8. Time to First Token (TTFT)

Imagine asking:

    Explain Kubernetes.

You submit the request.

Nothing appears for:

    1.2 seconds

Then suddenly:

    Kubernetes is...

starts appearing.

That initial waiting period is:

**Time to First Token (TTFT)**

Conceptually:

    Request sent
         │
         │
         │ 1.2 seconds
         │
         ▼
    First token arrives

Lower TTFT makes interactive applications feel faster.

---

# 9. Time to Last Token

Suppose generation continues:

    Kubernetes is an orchestration...
    ...
    ...
    ...

and finishes after:

    7 seconds

Then:

    Request
       │
       ├── 1.2 sec ──► First token
       │
       └── 7 sec ────► Last token

For chat applications, streaming can significantly improve perceived latency.

Without streaming:

    Request

    [........7 seconds........]

    Complete response appears

With streaming:

    Request

    [1.2 sec]

    Kubernetes
    Kubernetes is
    Kubernetes is a
    Kubernetes is a container...

The actual generation might still take several seconds.

But the user starts reading much earlier.

Therefore:

    Actual latency

and:

    Perceived latency

are not always the same.

---

# 10. Why Bigger Models Can Be Slower

Larger or more computationally intensive models generally require more computation.

Conceptually:

    Small Model

    Request
       │
       ▼
    [Model]
       │
       ▼
    Response

    FAST

Compared with:

    Complex Reasoning Model

    Request
       │
       ▼
    [More computation/reasoning]
       │
       ▼
    Response

    Potentially SLOWER

This isn't an absolute rule because providers heavily optimize their infrastructure.

But model choice and reasoning effort can materially affect latency.

---

# 11. The Cost/Latency/Quality Triangle

One of the most useful mental models is:

                        QUALITY
                           ▲
                          / \
                         /   \
                        /     \
                       /       \
                      /         \
                     /           \
                  COST -------- LATENCY

Often:

    More capability
         ↓
    Potentially better quality
         ↓
    More computation
         ↓
    Potentially higher cost
         ↓
    Potentially higher latency

But this isn't an absolute law.

Always benchmark the actual models against your workload.

---

# 12. Example: Customer Support System

Imagine your application receives:

    1 million requests/day

Requests include:

    "Where is my order?"

    "Change my address."

    "Explain why my payment failed."

    "Compare my last six invoices and identify anomalies."

Using your most expensive model for everything would be wasteful.

Instead:

                        User Request
                             │
                             ▼
                       Classification
                             │
                 ┌───────────┼────────────┐
                 │           │            │
                 ▼           ▼            ▼
              Simple       Medium       Complex
                 │           │            │
                 ▼           ▼            ▼
              Small        Medium       Powerful
              Model        Model         Model

This introduces an important concept:

**Model Routing**

---

# 13. Model Routing

Model routing means dynamically selecting a model based on the request.

For example:

    User Request
         │
         ▼
    Model Router
         │
         ├── Easy ──────► Small Model
         │
         ├── Medium ────► Medium Model
         │
         └── Complex ───► Reasoning Model

Pseudo-Go:

    func selectModel(req Request) string {

        switch req.Complexity {

        case "low":
            return "small-model"

        case "medium":
            return "medium-model"

        case "high":
            return "reasoning-model"

        default:
            return "small-model"
        }
    }

Architecture:

                       User
                         │
                         ▼
                    API Gateway
                         │
                         ▼
                    AI Service
                         │
                         ▼
                   Model Router
                         │
             ┌───────────┼───────────┐
             │           │           │
             ▼           ▼           ▼
           Small       Medium      Reasoning
           Model       Model        Model

But this creates another problem:

> How does the router determine request complexity?

Possible approaches include:

- Rules
- Request metadata
- Task type
- User tier
- Cheap classifier model
- Previous evaluation data
- Prompt length
- Tool requirements
- Business rules

---

# 14. Model Cascades / Escalation

Sometimes it's better to start with a cheaper model.

Think:

    Start cheap → escalate when necessary

Architecture:

    Request
       │
       ▼
    Small Model
       │
       ▼
    Was result acceptable?
       │
       ├── YES ──► Return
       │
       └── NO
           │
           ▼
        Strong Model
           │
           ▼
         Return

Suppose you receive:

    100,000 requests

and:

    85% → handled successfully by small model

    15% → require stronger model

Then expensive model usage becomes:

    15,000 requests

instead of:

    100,000 requests

This can significantly reduce cost.

---

# 15. How Do We Know if the Result is Good?

One approach would be asking the model:

    Are you confident?

But self-reported model confidence can be unreliable.

Better signals include:

    Schema validation

    Required fields present

    Tool execution success

    Retrieval evidence

    Business-rule validation

    Unit-test execution

    Verifier model

    Human evaluation

For example, suppose your AI must return:

    {
        "order_id": "...",
        "action": "..."
    }

You can validate:

    Is JSON valid?

    Is order_id present?

    Does order_id exist?

    Is action one of the allowed values?

If validation fails:

    Small Model
         │
         ▼
    Validation Failed
         │
         ▼
    Strong Model

---

# 16. Example: Coding Agent

Imagine you've built an AI coding agent.

User asks:

    Rename this Go variable.

Using a powerful reasoning model is probably unnecessary.

Another request:

    We have an intermittent deadlock
    in this Go service.

    Here are:

    - 12 source files
    - goroutine dumps
    - application logs

    Find the root cause and propose
    a safe fix.

That's very different.

Your router might behave like:

                           Coding Request
                                 │
                   ┌─────────────┼─────────────┐
                   │             │             │
                 Simple        Medium        Complex
                   │             │             │
                   ▼             ▼             ▼
              Small Model    Coding Model   Reasoning Model

Examples:

    Small Model

    - Rename variable
    - Generate comments
    - Simple refactoring
    - Explain simple code

    Coding Model

    - Implement API
    - Write tests
    - Add endpoints
    - Refactor modules

    Reasoning Model

    - Debug deadlocks
    - Architecture analysis
    - Multi-file reasoning
    - Complex production incidents

---

# 17. Context Window Also Matters

Suppose your application sends:

    System prompt        2K tokens

    User question        1K

    Conversation        20K

    Retrieved documents 40K

    Source code         30K

    ------------------------

    Total               93K tokens

Your selected model must support that context size.

But there's an important lesson:

> Just because a model supports a huge context window doesn't mean you should fill it.

More context means:

    More tokens
         ↓
    Higher cost
         ↓
    Potentially higher latency
         ↓
    Possibly more irrelevant information

Good AI engineering often means sending:

> **Minimum relevant context**

rather than:

> **Maximum possible context**

---

# 18. Model Capability Matters

Model selection isn't only about intelligence.

Different applications require different capabilities.

You might need:

    Text

    Images

    Audio

    Video

    Tool calling

    Structured JSON

    Long context

    Reasoning

    Code generation

    Embeddings

For example, an invoice processing system might need:

    Invoice Image/PDF
           │
           ▼
    Multimodal Model
           │
           ▼
    Structured JSON
           │
           ▼
    Database

Whereas semantic search might use:

    Document
       │
       ▼
    Embedding Model
       │
       ▼
    Vector
       │
       ▼
    Vector Database

These are completely different model-selection problems.

---

# 19. Don't Pick Models From Benchmarks Alone

Suppose you see:

    Model X = 92 benchmark score

    Model Y = 89 benchmark score

Does that mean Model X is better for your product?

Not necessarily.

Your actual task might be:

    Input:

    Customer email

    Output:

    {
        "refund_requested": true,
        "order_id": "...",
        "sentiment": "angry"
    }

After testing, you might discover:

    Model X

    Accuracy on YOUR task = 98.7%

While:

    Model Y

    Accuracy on YOUR task = 99.3%

Despite Model X scoring higher on a generic benchmark.

Therefore:

    Generic Benchmark

    ≠

    Your Production Workload

Your own evaluation matters more.

---

# 20. Build an Evaluation Dataset

This is one of the most important production AI practices.

Create representative examples from your application.

For example:

    evals/

        easy_cases.json

        medium_cases.json

        hard_cases.json

        edge_cases.json

An evaluation example might look like:

    {
      "input": "Cancel order ORD-123",
      "expected": {
        "intent": "cancel_order",
        "order_id": "ORD-123"
      }
    }

Suppose you collect:

    1,000 representative requests

Run every candidate model against them.

Then measure:

| Model | Task Success | Avg Latency | Avg Cost |
|---|---:|---:|---:|
| Small | 94% | 400 ms | $0.001 |
| Medium | 97% | 900 ms | $0.004 |
| Large | 98% | 2.5 s | $0.015 |

Now model selection becomes an engineering decision instead of a guess.

---

# 21. Look at Percentiles, Not Just Average Latency

Don't only measure:

    Average latency = 1.2 seconds

Also measure:

    p50 = 800 ms

    p95 = 2.1 sec

    p99 = 5.7 sec

Why?

Because average latency can hide slow requests.

For example:

    Request 1 → 500 ms

    Request 2 → 600 ms

    Request 3 → 700 ms

    Request 4 → 800 ms

    Request 5 → 10 seconds

Most requests are fast.

But some users experience terrible latency.

That's why production systems commonly monitor:

    p50
    p95
    p99

This is particularly important for AI systems because generation time can vary significantly.

---

# 22. Calculate Cost Per Business Operation

Don't stop at:

    $X / 1M tokens

That number is difficult to reason about from a product perspective.

Instead calculate:

    Cost per support ticket

    Cost per generated report

    Cost per coding task

    Cost per resolved customer

    Cost per agent run

For example, an agent might do:

    User asks task
         │
         ▼
    LLM Call #1
         │
         ▼
    Tool Call
         │
         ▼
    LLM Call #2
         │
         ▼
    Database Search
         │
         ▼
    LLM Call #3
         │
         ▼
    Final Response

One user request resulted in:

    3 LLM calls

Therefore:

    Cost per request

    ≠

    Cost of one model call

Instead:

    Total Agent Cost =

        LLM Call #1
        +
        LLM Call #2
        +
        LLM Call #3
        +
        Embedding Costs
        +
        Retrieval Costs
        +
        External API Costs
        +
        Infrastructure Costs

This becomes extremely important when building AI agents.

---

# 23. Latency Works the Same Way

Suppose:

    LLM reasoning      1.5 sec

    Database           0.1 sec

    External API       2.0 sec

    Second LLM call    1.2 sec

Total:

    ~4.8 seconds

Making the first LLM call 200ms faster won't dramatically improve the application if the external API takes 2 seconds.

Think about:

**End-to-end latency**

For example:

    User
     │
     ▼
    Backend ─────────── 50ms
     │
     ▼
    LLM ─────────────── 1500ms
     │
     ▼
    Search ──────────── 500ms
     │
     ▼
    External API ────── 2000ms
     │
     ▼
    LLM ─────────────── 1200ms
     │
     ▼
    User

    Total ≈ 5.25 seconds

This is similar to profiling a distributed backend system.

You need to find:

> Where is the latency actually coming from?

---

# 24. Production Model-Selection Strategy

A practical model-selection process looks like:

                     Define the Task
                           │
                           ▼
                   Define Requirements
                           │
                 ┌─────────┼─────────┐
                 ▼         ▼         ▼
              Quality    Latency    Cost
                 │         │         │
                 └─────────┼─────────┘
                           ▼
                 Select Candidate Models
                           │
                           ▼
                     Build Eval Set
                           │
                           ▼
                    Benchmark Models
                           │
                           ▼
              Compare Quality/Cost/Latency
                           │
                           ▼
                    Choose Baseline
                           │
                           ▼
                   Deploy + Monitor
                           │
                           ▼
                     Re-evaluate

---

# 25. Set Explicit Requirements

Before comparing models, define requirements.

For example:

    Task:

    Customer support intent classification

Requirements:

    Accuracy        >= 97%

    p95 latency     <= 1 second

    Cost/request    <= $0.003

    JSON validity   >= 99.9%

Now imagine:

| Model | Accuracy | p95 | Cost | JSON Validity |
|---|---:|---:|---:|---:|
| A | 94% | 300ms | $0.0005 | 99.9% |
| B | 97.5% | 700ms | $0.002 | 99.95% |
| C | 99% | 2.5s | $0.015 | 99.99% |

Which model should we choose?

Probably:

    Model B

Why?

Model A:

    ❌ Doesn't satisfy accuracy requirement.

Model C:

    ❌ Doesn't satisfy latency requirement.
    ❌ Doesn't satisfy cost requirement.

Model B:

    ✅ Accuracy
    ✅ Latency
    ✅ Cost
    ✅ JSON reliability

Even though Model C is more accurate, it isn't necessarily the best engineering choice.

---

# 26. A Useful Production Principle

A very useful rule is:

> **Use the cheapest and fastest model that reliably meets your quality requirements.**

Not:

> Use the cheapest model.

And not:

> Use the smartest model.

Instead:

    Minimum sufficient capability
              +
    Required reliability
              +
    Acceptable latency
              +
    Acceptable cost

---

# 27. Cost Optimization Techniques

Once you've selected a model, you can optimize further.

## 27.1 Reduce Unnecessary Context

Instead of:

    Send entire 200-page document
            │
            ▼
           LLM

Use retrieval:

    User Question
          │
          ▼
    Search Relevant Chunks
          │
          ▼
    Top Relevant Chunks
          │
          ▼
         LLM

This reduces:

- input tokens
- cost
- latency
- irrelevant context

---

## 27.2 Limit Unnecessary Output

Instead of:

    Explain everything.

Use:

    Return a summary under 100 words.

Or:

    {
        "category": "...",
        "priority": "...",
        "summary": "..."
    }

---

## 27.3 Cache Reusable Results

Architecture:

    Request
       │
       ▼
     Cache
       │
       ├── HIT ──► Return Result
       │
       └── MISS
            │
            ▼
           LLM
            │
            ▼
       Store in Cache

Caching can reduce both:

    Cost
      +
    Latency

---

## 27.4 Use Smaller Models Where Possible

For example:

    Classification
         ↓
    Small Model

    Extraction
         ↓
    Small Model

    Simple Summarization
         ↓
    Small/Medium Model

    Complex Reasoning
         ↓
    Strong Model

---

## 27.5 Batch Workloads When Appropriate

Not every task needs real-time processing.

For example:

    Realtime User Request

    → Optimize latency

But:

    Nightly Document Processing

    → Optimize cost and throughput

Different workloads require different optimization strategies.

---

# 28. Latency Optimization Techniques

Common techniques include:

- Streaming
- Caching
- Smaller/faster models
- Shorter prompts
- Shorter outputs
- Parallel tool calls
- Fewer sequential model calls
- Efficient retrieval
- Connection reuse
- Timeouts

For example, imagine two independent searches.

Bad:

    Search A ──2s──►
                    Search B ──2s──►
                                     LLM

Total search latency:

    ~4 seconds

Better:

              ┌── Search A ──2s──┐
    Request ──┤                  ├──► LLM
              └── Search B ──2s──┘

Total search latency:

    ~2 seconds

This is the same concurrency thinking used in backend engineering.

---

# 29. A Realistic AI Backend Architecture

Eventually, an AI backend might evolve into:

                            Client
                              │
                              ▼
                         API Gateway
                              │
                              ▼
                        AI Orchestrator
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
           Cache           Retrieval       Model Router
              │               │                │
              │               │        ┌───────┼────────┐
              │               │        ▼       ▼        ▼
              │               │      Small   Medium   Reasoning
              │               │
              │               ▼
              │           Vector DB
              │
              └───────────────────────────────┐
                                              │
                                              ▼
                                           Response

You might monitor:

    tokens/request

    cost/request

    cost/user

    TTFT

    end-to-end latency

    p50 latency

    p95 latency

    p99 latency

    model error rate

    tool-call success rate

    task success rate

    cache hit rate

    escalation rate

This is where AI engineering starts looking very similar to backend engineering.

---

# 30. Beginner Mental Model

If you're just starting AI engineering, remember:

> Different models have different capabilities, speeds, and prices.

Don't automatically use the strongest model.

Think:

    Simple Task
         ↓
    Small Model

    Medium Task
         ↓
    Medium Model

    Complex Task
         ↓
    Strong/Reasoning Model

---

# 31. Experienced Engineer Mental Model

For an experienced engineer:

> Model selection is a constrained optimization problem over task quality, cost, latency, reliability, throughput, and capabilities.

The model-selection lifecycle becomes:

    Model Selection
          ↓
    Model Routing
          ↓
    Fallbacks / Cascades
          ↓
    Evaluations
          ↓
    Observability
          ↓
    Continuous Optimization

---

# 32. Final Mental Model

Whenever you're selecting a model, ask:

                     What is my task?
                           │
                           ▼
                  How difficult is it?
                           │
                           ▼
             What capabilities are needed?
                           │
                           ▼
                 What quality is enough?
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
              COST      LATENCY     QUALITY
                │          │          │
                └──────────┼──────────┘
                           ▼
                  Candidate Models
                           │
                           ▼
                         Evals
                           │
                           ▼
                 Cheapest/Fastest Model
                Satisfying Requirements

The key idea is:

> **The goal of model selection isn't to find the world's best model. It's to find the best model for your specific task and constraints.**

A useful production principle is:

> **Use the cheapest and fastest model that reliably meets your quality requirements.**

This is similar to backend engineering.

You wouldn't automatically choose:

- the largest database instance
- the biggest Kubernetes node
- the most expensive cache
- the highest number of servers

Instead, you understand the workload, measure it, and choose sufficient infrastructure.

Model selection follows the same engineering mindset.

---

# Quick Revision

## What is Model Selection?

Choosing the appropriate AI model for a specific task.

---

## What are the main trade-offs?

    Quality
      +
    Cost
      +
    Latency

Along with:

    Reliability
    Context Window
    Capabilities
    Throughput
    Structured Output
    Tool Calling

---

## What is TTFT?

**Time to First Token**

How long it takes before the model starts returning its response.

---

## What is Model Routing?

Selecting different models based on the request.

    Request
       │
       ▼
    Router
       │
       ├── Simple ──► Small Model
       ├── Medium ──► Medium Model
       └── Complex ─► Strong Model

---

## What is a Model Cascade?

Start with a cheaper model and escalate when necessary.

    Request
       │
       ▼
    Small Model
       │
       ▼
    Good Enough?
       │
       ├── YES ──► Return
       │
       └── NO ──► Strong Model

---

## How should models be compared?

Don't rely only on public benchmarks.

Create an evaluation dataset using your actual application workload.

Measure:

    Quality
    Cost
    Latency
    Reliability

---

## Golden Rule

> **Use the cheapest and fastest model that reliably meets your application's quality requirements.**

Or remember it as:

    Right Task
       +
    Right Model
       +
    Right Quality
       +
    Acceptable Cost
       +
    Acceptable Latency
       =
    Good Model Selection
