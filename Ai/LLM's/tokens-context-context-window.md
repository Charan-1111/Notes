# Tokens, Context, and Context Window in LLMs

If you are starting with Large Language Models (LLMs), three concepts you should understand very clearly are:

1. **Tokens**
2. **Context**
3. **Context Window**

These concepts appear everywhere when working with LLMs, whether you are:

- Using ChatGPT
- Calling an LLM API
- Building an AI chatbot
- Building RAG systems
- Building AI agents
- Optimizing LLM cost
- Optimizing LLM latency
- Designing production AI systems

Let's understand them from the basics.

---

# 1. First, How Does an LLM Read Text?

Suppose you send this message to an LLM:

```text
Explain Docker to me.
```

As humans, we see a sentence containing words.

But an LLM doesn't fundamentally process text as words.

Instead, the text is first converted into smaller units called:

```text
Tokens
```

The process looks roughly like this:

```text
Human text

"Explain Docker to me."

        ↓

Tokenizer

        ↓

Tokens

["Explain", " Docker", " to", " me", "."]

        ↓

Token IDs

[1234, 5678, 91, 234, 13]

        ↓

LLM
```

The token IDs shown here are only examples.

Different models/tokenizers may convert the same text differently.

The important idea is:

> LLMs process tokens rather than directly processing words or sentences.

---

# 2. What Is a Token?

A **token is a small unit of text that an LLM processes**.

A token can be:

- A complete word
- Part of a word
- Punctuation
- A number
- A symbol
- Sometimes spaces combined with words

For example:

```text
Hello world!
```

might roughly become:

```text
"Hello"
" world"
"!"
```

These pieces are tokens.

---

# 3. Token Does NOT Mean Word

This is one of the most important things to understand.

Many beginners assume:

```text
1 word = 1 token
```

This is not always true.

Consider:

```text
I like coffee.
```

It might approximately become:

```text
"I"
" like"
" coffee"
"."
```

But consider a more unusual word:

```text
unbelievable
```

Depending on the tokenizer, it could potentially be divided into pieces similar to:

```text
"un"
"believ"
"able"
```

So:

```text
1 word
```

could become:

```text
multiple tokens
```

Therefore:

```text
Word != Token
```

A better mental model is:

```text
Text
 ↓
Small reusable pieces
 ↓
Tokens
```

---

# 4. Why Do LLMs Use Tokens?

Computers work with numbers.

An LLM cannot directly perform neural-network calculations on:

```text
Hello
```

The text needs to eventually be represented numerically.

For example:

```text
"Hello"  → 15496
"world"  → 995
"!"      → 0
```

Again, these numbers are only illustrative.

The actual IDs depend on the tokenizer.

The process is approximately:

```text
Text
 ↓
Tokenizer
 ↓
Tokens
 ↓
Token IDs
 ↓
Embeddings / numerical representations
 ↓
Neural Network
 ↓
Predictions
```

---

# 5. What Is a Tokenizer?

The component responsible for converting text into tokens is called a:

```text
Tokenizer
```

Its job is approximately:

```text
Text → Tokens → Token IDs
```

For example:

```text
Input:

I am learning Golang.
```

The tokenizer might produce something conceptually like:

```text
["I", " am", " learning", " Go", "lang", "."]
```

and internally associate them with numerical IDs.

Different models may use different tokenizers.

Therefore, the exact same sentence may use a different number of tokens with different models.

---

# 6. LEGO Analogy for Tokens

Think about LEGO.

You don't create a large LEGO structure from one giant block.

You combine smaller pieces:

```text
🧱 + 🧱 + 🧱 + 🧱 + 🧱
```

Language works similarly for LLMs.

Instead of processing:

```text
"I am learning backend development"
```

as one giant object, the tokenizer breaks it into smaller pieces.

Those pieces are:

```text
Tokens
```

The LLM then learns relationships between these token patterns.

---

# 7. LLMs Generate Tokens Too

Tokens are not only used for your input.

LLMs also generate their responses as tokens.

Suppose you ask:

```text
What is Docker?
```

The LLM might generate:

```text
Docker is a containerization platform...
```

But conceptually, the model doesn't generate the entire sentence at once.

Instead, it generates something like:

```text
Docker
```

then:

```text
Docker is
```

then:

```text
Docker is a
```

then:

```text
Docker is a containerization
```

then:

```text
Docker is a containerization platform
```

and so on.

Conceptually:

```text
Input tokens
     ↓
LLM
     ↓
Predict next token
     ↓
Add token to sequence
     ↓
Predict next token
     ↓
Add token to sequence
     ↓
...
```

This is why LLMs are commonly described as:

> Next-token prediction models.

---

# 8. Simple Next-Token Example

Suppose the current text is:

```text
The capital of France is
```

The model considers possible next tokens.

Conceptually, it might assign probabilities similar to:

```text
Paris      → 97%
London     → 1%
Berlin     → 0.5%
Rome       → 0.3%
...
```

The model then selects a next token according to its decoding configuration.

After generating:

```text
Paris
```

the sequence becomes:

```text
The capital of France is Paris
```

Then the model predicts the next token again.

This continues until the response is complete or another generation limit is reached.

---

# 9. Input Tokens and Output Tokens

When using LLM APIs, you will frequently encounter two terms:

```text
Input tokens
Output tokens
```

## Input Tokens

These are tokens sent **to the model**.

For example:

```text
System instructions
+
Developer instructions
+
Conversation history
+
Retrieved documents
+
User message
```

All of these can contribute to the model's input.

## Output Tokens

These are tokens generated **by the model**.

For example:

```text
User:

Explain Docker.
```

Suppose the complete input uses:

```text
20 tokens
```

and the answer uses:

```text
300 tokens
```

Then approximately:

```text
Input tokens  = 20
Output tokens = 300
```

---

# 10. Why Should AI Engineers Care About Tokens?

Tokens directly affect several important production concerns.

## 10.1 Cost

Many LLM APIs charge based on token usage.

Conceptually:

```text
Cost
 ≈
Input token cost
+
Output token cost
```

Suppose your application sends huge prompts containing unnecessary information.

Instead of:

```text
1,000 input tokens
```

you send:

```text
20,000 input tokens
```

Across millions of requests, that difference can become very expensive.

---

## 10.2 Latency

More tokens generally mean more computation.

Large input:

```text
More input tokens
      ↓
More information to process
      ↓
Potentially more latency
```

Large output:

```text
More generated tokens
      ↓
More generation steps
      ↓
Potentially more latency
```

---

## 10.3 Context Limits

Every model has limits on how much information it can process within its active context.

These limits are usually expressed using:

```text
Tokens
```

For example:

```text
128K tokens
```

or another model-specific limit.

This brings us to our next concept:

# Context

---

# 11. What Is Context?

**Context is the information available to the LLM while it is generating a response.**

This is extremely important.

Suppose you have this conversation:

```text
User:
My favorite programming language is Go.

Assistant:
Nice! Go is excellent for backend systems.

User:
What is my favorite programming language?
```

The model can answer:

```text
Go
```

Why?

Because:

```text
"My favorite programming language is Go."
```

was available in the context supplied to the model.

---

# 12. Context Is Like the LLM's Working Desk

Imagine you're solving a problem at your desk.

On your desk you have:

```text
┌────────────────────────────────────────────┐
│                                            │
│ Instructions                               │
│                                            │
│ Previous conversation                      │
│                                            │
│ User question                              │
│                                            │
│ Documentation                              │
│                                            │
│ Retrieved database information             │
│                                            │
│ Tool results                               │
│                                            │
└────────────────────────────────────────────┘
```

You can use everything currently available on the desk to solve the problem.

Think of this desk as:

```text
Context
```

The model generates its answer based on the information available in that context plus patterns/knowledge learned during training.

---

# 13. What Can Be Part of Context?

Depending on the application, context can contain many things.

For example:

```text
System instructions
+
Developer/application instructions
+
Previous conversation
+
Current user question
+
Retrieved documents
+
Tool results
+
Database information
+
Search results
+
Code
+
Examples
```

Conceptually:

```text
┌──────────────────────────────────────┐
│              CONTEXT                 │
│                                      │
│ System instructions                  │
│                                      │
│ Previous messages                    │
│                                      │
│ Retrieved documents                  │
│                                      │
│ Tool outputs                         │
│                                      │
│ User's current question              │
└──────────────────────────────────────┘
                   ↓
                  LLM
                   ↓
                Answer
```

---

# 14. Example: Customer Support AI

Imagine you're building an AI customer support system.

The customer asks:

```text
Why did my payment fail?
```

Your backend could collect:

```text
System instructions

"You are a customer support assistant."

+

Customer information

Customer ID: 123
Plan: Premium

+

Transaction information

Payment ID: PAY-982
Status: Failed
Reason: Insufficient funds

+

Relevant documentation

"Payments may fail when the customer's bank
rejects the transaction."

+

Conversation history

+

User question

"Why did my payment fail?"
```

All of this information can be supplied as:

```text
Context
```

The model can then generate something like:

```text
Your payment was rejected because the bank
reported insufficient funds.
```

The model was able to provide a specific answer because the relevant transaction information was placed in its context.

---

# 15. Context vs Training Knowledge

These are different things.

During training, an LLM learns patterns from large amounts of data.

That contributes to the model's learned parameters.

But during an actual conversation, you can provide additional information through:

```text
Context
```

For example, suppose your company launches an internal service called:

```text
FalconPaymentService
```

The model may know nothing about your private service from training.

But you can provide:

```text
FalconPaymentService handles payment authorization
for our checkout platform.
```

Now the model can use that information because it exists in the current context.

This is extremely important for AI engineering.

---

# 16. Context Is NOT the Same as Memory

This is another important beginner misconception.

Consider:

```text
Context
```

as:

> Information currently available to the model for this generation.

Memory is a broader application/product concept where information may be stored and later brought back into future interactions.

A useful mental model is:

```text
Context
=
What's currently on the desk
```

while:

```text
Memory
=
Information stored somewhere that may later
be placed back on the desk
```

For example:

```text
Database
Vector Database
Conversation Store
Memory System
        ↓
Relevant information retrieved
        ↓
Added to Context
        ↓
LLM
```

---

# 17. What Is a Context Window?

Now we come to one of the most important concepts.

A **context window is the maximum amount of tokenized information that a model can work with at one time.**

Think about our desk analogy again.

Your desk has limited space.

```text
Small desk

┌─────────────────┐
│                 │
│    Information  │
│                 │
└─────────────────┘
```

You cannot put unlimited books and documents on it.

A larger desk:

```text
┌──────────────────────────────────────────┐
│                                          │
│ Instructions                             │
│ Documentation                            │
│ Conversation                             │
│ Code                                     │
│ Retrieved information                    │
│ User question                            │
│                                          │
└──────────────────────────────────────────┘
```

can hold much more information.

Similarly:

```text
Context Window
=
Size of the LLM's active working space
```

---

# 18. Context Window Is Measured in Tokens

Suppose, purely as an example, that a model supports:

```text
100,000 tokens
```

of context.

That means the model can work with approximately that amount of tokenized information within its supported context constraints.

Conceptually:

```text
Context Window
        ↓
Measured using
        ↓
Tokens
```

This is why understanding tokens first is important.

---

# 19. Tokens vs Context vs Context Window

These three concepts are related but different.

## Tokens

Tokens are:

```text
Pieces of text processed by the model.
```

## Context

Context is:

```text
The information currently available to the model.
```

## Context Window

Context window is:

```text
The maximum amount of tokenized information
the model can work with at one time.
```

Think:

```text
Tokens
=
Pieces of paper


Context
=
The papers currently placed on your desk


Context Window
=
The maximum amount of paper your desk can hold
```

This analogy is worth remembering.

---

# 20. What Uses the Context Window?

Suppose you're building an AI assistant.

Your request contains:

```text
System instructions       = 1,000 tokens

Conversation history      = 10,000 tokens

Retrieved documents       = 20,000 tokens

User question             = 100 tokens
```

Your input is approximately:

```text
1,000
+ 10,000
+ 20,000
+ 100
----------------
31,100 input tokens
```

The model also needs to generate a response.

Depending on the particular model/API, context and output limits can be defined differently, so always check the model's documentation.

But conceptually you should think in terms of a token budget:

```text
Available model capacity
        ↓
Input/context tokens
+
Room needed for generation
```

---

# 21. Example of Context Growth in a Chatbot

Imagine a user starts chatting with your AI assistant.

Initially:

```text
User:
What is Docker?
```

Small context.

Then:

```text
User:
What is Docker?

Assistant:
Docker is...

User:
Explain containers.

Assistant:
Containers are...

User:
How are containers different from VMs?

Assistant:
...
```

As the conversation continues:

```text
Message 1
+
Message 2
+
Message 3
+
Message 4
+
...
```

the conversation history grows.

Therefore:

```text
More conversation
       ↓
More tokens
       ↓
More context usage
       ↓
Eventually context management becomes important
```

---

# 22. What Happens When the Conversation Becomes Huge?

Imagine:

```text
Context Window

|--------------------------------------------|
```

Initially:

```text
| Conversation |
```

Then:

```text
| Conversation-------------|
```

Then:

```text
| Conversation-------------------------------|
```

Eventually you reach the model/application limits.

At that point, your application may need to decide:

```text
What information should we keep?

What should we remove?

What should we summarize?

What should we retrieve only when necessary?
```

This is called:

```text
Context Management
```

---

# 23. Common Context Management Techniques

Production AI applications don't simply keep adding information forever.

They often use techniques such as:

## Technique 1: Remove irrelevant messages

Instead of sending:

```text
Entire conversation since account creation
```

send only:

```text
Relevant recent conversation
```

---

## Technique 2: Summarization

Suppose the conversation contains:

```text
20,000 tokens
```

You might summarize older parts into:

```text
2,000 tokens
```

Then send:

```text
Summary
+
Recent conversation
+
Current question
```

Conceptually:

```text
Old Conversation
      ↓
Summarization
      ↓
Short Summary
      ↓
Context
```

---

## Technique 3: Retrieval

Instead of putting all documents into the context:

```text
Document 1
Document 2
Document 3
...
Document 100,000
```

retrieve only relevant information.

```text
User Question
      ↓
Search / Retrieval
      ↓
Relevant chunks
      ↓
Context
      ↓
LLM
```

This idea is central to:

```text
RAG
```

or:

```text
Retrieval-Augmented Generation
```

---

# 24. Why RAG Is Related to Context

Suppose your company has:

```text
1,000,000 documents
```

You cannot practically put every document into every LLM request.

Instead:

```text
                     1,000,000 Documents
                              │
                              ↓
                     Search / Retrieval
                              │
                              ↓
                    Relevant Documents
                              │
                              ↓
                       LLM Context
                              │
                              ↓
                           Answer
```

For example:

```text
User:

How many paid leaves do employees receive?
```

Your retrieval system might search:

```text
1,000,000 company documents
```

and find:

```text
HR Leave Policy
```

Then only relevant sections are placed into the context:

```text
System Instructions
+
Relevant Leave Policy
+
User Question
```

Now the model has the information needed to answer.

One way to think about RAG is:

> Find the right information and place it into the model's context at the right time.

---

# 25. Why Not Always Use the Largest Possible Context?

You might think:

```text
If the model supports a huge context window,
why don't we just send everything?
```

Because bigger context isn't automatically better.

Consider sending:

```text
100 documents
```

when only:

```text
2 documents
```

are relevant.

You're potentially creating several problems.

## Problem 1: Cost

More tokens can mean:

```text
Higher API cost
```

---

## Problem 2: Latency

More input means more information for the model to process.

```text
More tokens
    ↓
More computation
    ↓
Potentially higher latency
```

---

## Problem 3: Noise

Suppose the actual answer is contained in:

```text
Document 73
```

but you send:

```text
Document 1
Document 2
Document 3
...
Document 100
```

You've added lots of irrelevant information.

Relevant information can become harder to distinguish from noise.

Good AI engineering is not:

```text
Give the LLM everything.
```

It is:

```text
Give the LLM the right information.
```

---

# 26. Context Engineering

This leads to an important modern AI engineering concept:

```text
Context Engineering
```

Prompt engineering asks questions like:

```text
How should I write the instruction?
```

Context engineering is broader:

```text
What information should the model receive?

Which conversation messages?

Which documents?

Which examples?

Which tool results?

Which user information?

In what format?

In what order?
```

For example:

```text
User Question
      ↓
Application
      ↓
Collect relevant information
      │
      ├── System instructions
      ├── Conversation history
      ├── User information
      ├── Retrieved documents
      └── Tool results
      ↓
Construct useful context
      ↓
LLM
      ↓
Answer
```

Context engineering is extremely important when building production AI applications and agents.

---

# 27. Real Backend Example

Suppose you're building this endpoint in Go:

```http
POST /api/v1/support/chat
```

The request:

```json
{
  "message": "Why did my latest payment fail?"
}
```

Your backend could perform:

```text
HTTP Request
     ↓
Go Backend
     ↓
Authenticate User
     ↓
Get Customer Information
     ↓
Get Latest Payment
     ↓
Retrieve Payment Documentation
     ↓
Construct LLM Context
     ↓
Call LLM
     ↓
Return Answer
```

Your context might look conceptually like:

```text
SYSTEM:

You are a customer support assistant.
Only answer using the provided information.


CUSTOMER:

Customer ID: 12345
Plan: Premium


PAYMENT:

Transaction: TX-98765
Amount: ₹2500
Status: FAILED
Reason: INSUFFICIENT_FUNDS


DOCUMENTATION:

INSUFFICIENT_FUNDS means the customer's
bank account did not have sufficient available
balance when the transaction was attempted.


USER:

Why did my latest payment fail?
```

The model now has useful context.

It can answer:

```text
Your latest payment failed because the bank
reported insufficient funds.
```

Notice something important:

The LLM didn't need access to your entire database.

Your backend selected the relevant information and put it into:

```text
Context
```

That is an important responsibility of an AI application.

---

# 28. Tokens Also Matter for LLM Cost

Suppose an LLM provider charges separately for:

```text
Input tokens
Output tokens
```

Imagine your application makes:

```text
1,000,000 requests/month
```

If every request unnecessarily contains:

```text
10,000 extra tokens
```

you are processing:

```text
10,000 × 1,000,000

= 10,000,000,000 unnecessary tokens
```

That's:

```text
10 billion unnecessary tokens
```

So token optimization can become extremely important at scale.

---

# 29. Tokens Also Affect Latency

There are two useful latency concepts.

## Time to First Token

How long before the user starts seeing the answer?

```text
Request
   ↓
LLM processing
   ↓
First output token
```

This is commonly called:

```text
TTFT
=
Time To First Token
```

---

## Generation Time

The model then continues generating:

```text
Token
Token
Token
Token
Token
...
```

Long answers require more generation.

So:

```text
Large input
+
Large output
```

can contribute to higher latency.

---

# 30. Why Streaming Is Useful

Because LLMs generate output incrementally, applications can often stream generated content.

Without streaming:

```text
User Request
     ↓
Wait
     ↓
Wait
     ↓
Wait
     ↓
Complete Response
```

With streaming:

```text
User Request
     ↓
Token
     ↓
Token
     ↓
Token
     ↓
Token
     ↓
...
```

The user starts seeing the response earlier.

This can significantly improve perceived responsiveness.

---

# 31. A More Complete LLM Request

Now let's connect everything.

Imagine your application sends:

```text
System Instructions
        +
Conversation History
        +
Retrieved Documents
        +
Tool Results
        +
User Question
```

These are converted into:

```text
Tokens
```

The total must fit within the model/API's supported limits.

Then:

```text
                    USER
                      │
                      ↓
                 APPLICATION
                      │
         ┌────────────┼─────────────┐
         │            │             │
         ↓            ↓             ↓
   Conversation   Documents      Tool Results
         │            │             │
         └────────────┼─────────────┘
                      ↓
                   CONTEXT
                      │
                      ↓
                  TOKENIZER
                      │
                      ↓
                    TOKENS
                      │
                      ↓
             ┌────────────────┐
             │ CONTEXT WINDOW │
             │                │
             │ Limited token  │
             │ capacity       │
             └───────┬────────┘
                     ↓
                    LLM
                     ↓
              Next Token
                     ↓
              Next Token
                     ↓
              Next Token
                     ↓
                    ...
                     ↓
                  RESPONSE
```

---

# 32. Common Beginner Misconceptions

## Misconception 1

```text
1 word = 1 token
```

Wrong.

A word may become one or multiple tokens.

---

## Misconception 2

```text
The LLM generates the entire answer at once.
```

Not conceptually.

Autoregressive LLMs generate the response token by token.

---

## Misconception 3

```text
Context = Memory
```

Not necessarily.

Context is information available during the current generation.

Memory may be stored elsewhere and later inserted into the context.

---

## Misconception 4

```text
A huge context window means we should always
send huge amounts of information.
```

No.

More information can mean:

```text
More cost
More latency
More noise
```

The goal is usually:

```text
Relevant Context
```

not:

```text
Maximum Context
```

---

## Misconception 5

```text
Context window tells us how intelligent
the model is.
```

No.

Context window tells us roughly how much tokenized information the model can work with at once.

A larger context window does not automatically mean:

```text
Better reasoning
Better accuracy
Better intelligence
```

Those are different characteristics.

---

# 33. Easy Mental Model

Remember this analogy.

Imagine you're studying for an exam.

## Tokens

Individual pieces of information:

```text
📝 📝 📝 📝 📝
```

Think:

```text
Tokens = pieces of text
```

---

## Context

The notes currently placed on your desk:

```text
┌───────────────────────────────┐
│                               │
│ Go Notes                      │
│ Docker Notes                  │
│ Kubernetes Notes              │
│ Question Paper                │
│                               │
└───────────────────────────────┘
```

Think:

```text
Context = information currently available
```

---

## Context Window

The size of your desk:

```text
┌───────────────────────────────┐
│                               │
│ Maximum information that can  │
│ fit in the working area       │
│                               │
└───────────────────────────────┘
```

Think:

```text
Context Window = working-space capacity
```

Therefore:

```text
Tokens
=
Pieces of information


Context
=
Information currently available


Context Window
=
Maximum tokenized information the model
can work with at once
```

---

# 34. Practical Example

Imagine you're building an AI coding assistant.

The developer asks:

```text
Why is my authentication API failing?
```

You have:

```text
Entire Repository
      ↓
500,000 tokens
```

Suppose you cannot or do not want to place the whole repository into the active context.

Instead, your system finds:

```text
auth_handler.go
auth_service.go
jwt.go
middleware.go
Relevant error logs
```

Suppose those contain:

```text
8,000 tokens
```

Then you construct:

```text
System Instructions       1,000
Relevant Code             8,000
Logs                      2,000
Conversation              2,000
User Question               100
--------------------------------
Total                    13,100 tokens
```

Then:

```text
13,100 token context
        ↓
LLM
        ↓
Analyze relevant code
        ↓
Generate answer
```

This is much better than blindly sending:

```text
500,000 tokens
```

of potentially irrelevant code.

---

# 35. How AI Engineers Should Think About Context

Whenever you're building an LLM application, ask:

```text
1. What does the model need to know?

2. Where can I get that information?

3. Which information is actually relevant?

4. How many tokens will it require?

5. Will it fit within the model's limits?

6. How much room do I need for the output?

7. Can I remove unnecessary information?

8. Can I summarize older information?

9. Should I retrieve information dynamically?

10. How will this affect cost and latency?
```

This mindset becomes very important in production AI engineering.

---

# 36. Quick Comparison

| Concept | Meaning | Think Of It As |
|---|---|---|
| Token | Small unit of text processed by the LLM | LEGO piece |
| Tokenizer | Converts text into tokens/token IDs | LEGO separator |
| Input Tokens | Tokens sent to the model | Information given to the model |
| Output Tokens | Tokens generated by the model | Model's response |
| Context | Information available during generation | Notes on your desk |
| Context Window | Maximum active token capacity | Size of your desk |
| Context Management | Deciding what information should remain available | Organizing your desk |
| RAG | Retrieving relevant external information for the context | Finding the right book from a library |

---

# 37. Final Architecture to Remember

```text
                        USER
                          │
                          ↓
                    User Question
                          │
                          ↓
                     APPLICATION
                          │
           ┌──────────────┼──────────────┐
           │              │              │
           ↓              ↓              ↓
      Conversation     Retrieval      Tool Calls
        History        / RAG         / Database
           │              │              │
           └──────────────┼──────────────┘
                          ↓
                       CONTEXT
                          │
                          ↓
                      TOKENIZER
                          │
                          ↓
                        TOKENS
                          │
                          ↓
                 ┌─────────────────┐
                 │                 │
                 │ CONTEXT WINDOW  │
                 │                 │
                 │ Limited token   │
                 │ capacity        │
                 │                 │
                 └────────┬────────┘
                          │
                          ↓
                         LLM
                          │
                          ↓
                 Predict Next Token
                          │
                          ↓
                 Predict Next Token
                          │
                          ↓
                         ...
                          │
                          ↓
                       RESPONSE
```

---

# 38. The Three Definitions to Remember

## Token

> A token is a small unit of text processed by an LLM. It may represent a complete word, part of a word, punctuation, a number, or another text fragment.

---

## Context

> Context is the information available to an LLM while it is generating a response, such as instructions, conversation history, retrieved documents, tool results, and the user's current question.

---

## Context Window

> The context window is the maximum amount of tokenized information that an LLM can work with at one time, subject to the specific model/API's input and output limits.

---

# 39. The Simplest Way to Remember Everything

If you forget everything else, remember:

```text
Tokens
   ↓
Pieces of text the LLM processes


Context
   ↓
Information currently available to the LLM


Context Window
   ↓
How much tokenized information the LLM
can work with at once
```

Or use the desk analogy:

```text
Tokens         = Pieces of paper

Context        = Papers currently on the desk

Context Window = Size of the desk
```

And from an AI engineering perspective:

```text
Good AI System
     !=
Give LLM everything


Good AI System
     =
Give LLM the right information
at the right time
within the available context
```

That final idea is one of the foundations of building good LLM applications, RAG systems, and AI agents.
