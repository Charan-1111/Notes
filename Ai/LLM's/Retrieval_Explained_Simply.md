# Retrieval Explained Simply

## 1. What is retrieval?

**Retrieval** means finding the most useful information for a user's question from a collection of documents.

Example:

```text
Documents:
1. Leave policy
2. Password reset guide
3. Health insurance policy

Question: "I forgot my password. What should I do?"

Retrieved document: Password reset guide
```

The system searches the available information and returns the part most likely to answer the question.

## 2. Why is retrieval useful for LLMs?

An LLM may not know your company's private or recently updated information.

Suppose an employee asks:

```text
How many casual-leave days do I get?
```

The answer exists in the company handbook:

```text
Employees receive 12 casual-leave days per year.
```

Retrieval finds this text and gives it to the LLM. The LLM then creates an answer using the retrieved information.

```text
Question
   ↓
Retrieve relevant text
   ↓
Question + retrieved text sent to LLM
   ↓
Answer
```

This process is part of **Retrieval-Augmented Generation**, or **RAG**.

## 3. Retrieval is not generation

These are separate jobs:

- **Retrieval:** finds existing information.
- **Generation:** writes a human-readable answer.

Example:

```text
Retrieved text:
"Password reset links expire after 15 minutes."

Generated answer:
"Your reset link is valid for 15 minutes. Request a new one if it has expired."
```

The retriever supplies the evidence. The LLM explains it to the user.

## 4. Preparing documents for retrieval

Before users search, documents are usually prepared like this:

```text
Documents
   ↓ chunking
Small text chunks
   ↓ embedding model
Vectors
   ↓
Vector database
```

Example document:

```text
Employee Handbook
```

Example chunks:

```text
Chunk 1: Employees receive 12 casual-leave days per year.
Chunk 2: Employees may work remotely two days per week.
Chunk 3: Health insurance covers the employee and spouse.
```

Each chunk is stored separately so retrieval can return only the relevant section.

## 5. Semantic retrieval

Semantic retrieval searches by **meaning**, not only by exact words.

Stored chunk:

```text
Reset your account password using the recovery page.
```

User question:

```text
I forgot my login credentials.
```

The words are different, but their meanings are related.

The process is:

1. Convert the question into an embedding.
2. Ask the vector database for similar chunk vectors.
3. Return the closest chunks.

Example results:

```text
Password recovery guide → similarity 0.94
Profile update guide    → similarity 0.30
Invoice download guide  → similarity 0.08
```

The password guide ranks first because it is semantically closest.

## 6. Keyword retrieval

Keyword retrieval searches for matching words or phrases.

Query:

```text
Error code AUTH-4012
```

Document:

```text
AUTH-4012 occurs when an access token has expired.
```

Keyword search is excellent here because an exact error code matters more than general meaning.

## 7. Hybrid retrieval

**Hybrid retrieval** combines semantic and keyword search.

Example query:

```text
How do I fix AUTH-4012 when my login session expires?
```

- Keyword search finds the exact code `AUTH-4012`.
- Semantic search understands `login session expires`.
- Their results are combined and ranked.

Hybrid retrieval is often useful when documents contain both natural language and exact values such as product names, IDs, or error codes.

## 8. Metadata filtering

Metadata filtering limits which documents are allowed in the search.

Suppose a company has policies for India and the United States:

```text
Question: "How many casual leaves do I receive?"
Filter: country = "India"
```

Without the filter, retrieval might return the US policy because its wording is similar. With the filter, only Indian policy chunks are considered.

Common filters include:

```text
tenant_id = "company-101"
country   = "India"
language  = "en"
category  = "leave-policy"
```

In a multi-tenant application, always filter by tenant or organization so one customer's data is never retrieved for another customer.

## 9. What does Top-K mean?

`Top-K` is the number of results returned by retrieval.

```text
Top-K = 3
```

means: return the three most relevant chunks.

Example:

```text
1. Password reset steps       → 0.95
2. Password reset link expiry → 0.89
3. Recover a locked account   → 0.81
```

Returning too few chunks may miss useful context. Returning too many adds unrelated text, increases token usage, and may confuse the LLM.

Start with a small value such as `3` to `5`, then test using real questions.

## 10. Similarity threshold

A threshold removes results that are not similar enough.

```text
Minimum score = 0.70

0.92 → keep
0.78 → keep
0.41 → reject
```

If nothing passes the threshold, the application can say that it could not find enough information instead of asking the LLM to guess.

Similarity scores differ between embedding models and search methods, so choose the threshold through testing rather than treating one value as universal.

## 11. Reranking

Initial retrieval is fast, but the first result is not always the best. A **reranker** examines retrieved candidates more carefully and sorts them again.

```text
Vector search returns 20 candidates
              ↓
Reranker scores them against the exact question
              ↓
Best 3 chunks go to the LLM
```

Example:

```text
Question: "Can contractors use parental leave?"

Initial result 1: General parental-leave policy
Initial result 2: Contractor eligibility rules

After reranking:
1. Contractor eligibility rules
2. General parental-leave policy
```

The second document becomes first because it answers the specific question more directly.

## 12. Simplified Go example

The exact API depends on the embedding provider and vector database:

```go
package retrieval

import (
	"context"
	"fmt"
)

type Embedder interface {
	Embed(ctx context.Context, text string) ([]float32, error)
}

type Result struct {
	Text  string
	Score float32
}

type VectorStore interface {
	Search(
		ctx context.Context,
		vector []float32,
		limit int,
		filters map[string]string,
	) ([]Result, error)
}

func Retrieve(
	ctx context.Context,
	embedder Embedder,
	store VectorStore,
	question string,
	tenantID string,
) ([]Result, error) {
	// Convert the question into a vector.
	queryVector, err := embedder.Embed(ctx, question)
	if err != nil {
		return nil, fmt.Errorf("embed question: %w", err)
	}

	// Retrieve the five closest chunks belonging to this tenant.
	results, err := store.Search(ctx, queryVector, 5, map[string]string{
		"tenant_id": tenantID,
	})
	if err != nil {
		return nil, fmt.Errorf("search chunks: %w", err)
	}

	// Remove weak matches before sending context to the LLM.
	const minimumScore = 0.70
	useful := make([]Result, 0, len(results))

	for _, result := range results {
		if result.Score >= minimumScore {
			useful = append(useful, result)
		}
	}

	return useful, nil
}
```

For this question:

```text
I forgot my password. How long is the reset link valid?
```

the function may return:

```text
1. "Password reset links expire after 15 minutes." → 0.96
2. "Request a reset link from the login page."     → 0.84
```

The application then sends these chunks with the question to the LLM.

## 13. Common retrieval problems

### No useful result

Possible causes:

- the answer is missing from the documents;
- chunks are too large or too small;
- metadata filters are incorrect;
- the question and documents use very different terminology.

### Relevant result ranks too low

Possible improvements:

- combine keyword and semantic search;
- retrieve more candidates and rerank them;
- improve chunk boundaries;
- include titles or section names in chunks.

### Too many irrelevant results

Possible improvements:

- use metadata filters;
- apply a similarity threshold;
- lower Top-K;
- improve document and chunk quality.

## Quick summary

```text
Question
   ↓ create embedding
Search relevant chunks
   ↓ filter and rank
Best chunks
   ↓ add to the prompt
LLM answer
```

Retrieval is simply:

> Finding the best available information for a question before asking the LLM to answer it.
