# Embeddings Explained Simply

## 1. What is an embedding?

An **embedding** converts data—such as text, an image, or audio—into a list of numbers called a **vector**.

Example:

```text
"dog"   → [0.82, 0.14, 0.67]
"puppy" → [0.79, 0.18, 0.63]
"car"   → [0.10, 0.91, 0.22]
```

The numbers are learned by an embedding model. We do not manually decide them.

The important idea is:

> Items with similar meanings usually receive vectors that are close together.

Here, `dog` and `puppy` have similar vectors because their meanings are related. `Car` is less related, so its vector is farther away.

Real embedding vectors may contain hundreds or thousands of numbers. The short vectors above are only for explanation.

## 2. Why convert text into numbers?

Computers cannot directly compare meanings the way humans do. They work well with numbers.

Suppose a user searches for:

```text
How can I change my account password?
```

Your database contains this article:

```text
Steps to reset your login credentials
```

A keyword search may struggle because the sentences use different words. Embeddings can recognize that **change password** and **reset login credentials** have similar meanings.

The flow is:

```text
Search query → Embedding model → Query vector
Article      → Embedding model → Article vector
                              ↓
                    Compare the vectors
```

If the vectors are close, the article is probably relevant.

## 3. How is similarity measured?

A common method is **cosine similarity**. It measures how closely two vectors point in the same direction.

Its score usually ranges from `-1` to `1`:

- Close to `1`: very similar
- Close to `0`: mostly unrelated
- Close to `-1`: opposite directions

Example results:

```text
Query: "How do I reset my password?"

Article A: "Change forgotten login password" → 0.94
Article B: "Update profile picture"           → 0.31
Article C: "Company holiday calendar"         → 0.08
```

Article A has the highest score, so it should appear first.

You normally use an embedding API or vector database to calculate this—you do not implement the mathematics for every application.

## 4. A practical example: semantic search

Imagine that we have three support documents:

```text
1. Reset your account password
2. Download your monthly invoice
3. Cancel your subscription
```

During document storage:

1. Send each document to an embedding model.
2. Receive its vector.
3. Store the document and vector in a vector database.

```text
"Reset your account password" → [0.81, 0.12, 0.65]
```

When the user asks `I forgot my login credentials`:

1. Convert the question into an embedding using the **same model**.
2. Search the database for the closest document vectors.
3. Return the documents with the highest similarity scores.

Even though the query does not contain the word `password`, the password document can still be selected because the meanings are similar.

## 5. Embeddings in RAG

Embeddings are commonly used in **Retrieval-Augmented Generation (RAG)**.

Example: build a chatbot that answers questions from a company handbook.

### Before users ask questions

```text
Handbook
   ↓ split into small chunks
Text chunks
   ↓ embedding model
Vectors
   ↓
Vector database
```

We split a large document because retrieving a small relevant section is more useful than sending the entire handbook to the LLM.

### When a user asks a question

```text
User question
   ↓ create embedding
Search similar chunks
   ↓
Relevant handbook sections
   ↓
Question + sections sent to LLM
   ↓
Answer
```

Example:

```text
Question: "How many casual leaves do I get?"

Retrieved chunk:
"Employees receive 12 casual-leave days per year."
```

The LLM receives both the question and the retrieved chunk, so it can answer using the company’s actual information.

Important distinction:

- The **embedding model** finds relevant information.
- The **LLM** reads that information and generates the final answer.

## 6. Simplified Go example

The exact API differs between providers, but the application flow looks like this:

```go
package main

import (
	"context"
	"fmt"
)

type Embedder interface {
	Embed(ctx context.Context, text string) ([]float32, error)
}

type VectorStore interface {
	Search(ctx context.Context, vector []float32, limit int) ([]string, error)
}

func searchDocuments(
	ctx context.Context,
	embedder Embedder,
	store VectorStore,
	query string,
) ([]string, error) {
	// Convert the user's query into a vector.
	queryVector, err := embedder.Embed(ctx, query)
	if err != nil {
		return nil, fmt.Errorf("embed query: %w", err)
	}

	// Find the three document vectors closest to the query vector.
	documents, err := store.Search(ctx, queryVector, 3)
	if err != nil {
		return nil, fmt.Errorf("search vectors: %w", err)
	}

	return documents, nil
}
```

If `query` is `I cannot sign in because I forgot my password`, `Search` might return:

```text
1. Reset your account password
2. Recover a locked account
3. Contact account support
```

The interfaces hide provider-specific details, which makes the code easier to test and lets you change the embedding service or vector database later.

## 7. Common uses

- **Semantic search:** find results by meaning instead of exact words.
- **RAG:** retrieve relevant knowledge before asking an LLM to answer.
- **Recommendations:** find products, articles, or movies similar to one the user likes.
- **Clustering:** group related feedback, tickets, or documents.
- **Duplicate detection:** identify sentences or documents with nearly the same meaning.

## 8. Important points to remember

1. An embedding is a numeric representation of meaning.
2. Similar items usually have nearby vectors.
3. Store embeddings in a vector-capable database for similarity search.
4. Embed stored documents and user queries with the same embedding model.
5. Embeddings retrieve or compare information; they do not generate an answer.
6. Changing embedding models usually requires regenerating stored embeddings.

## One-line summary

> Embeddings turn meaning into numbers, allowing an application to find data that is conceptually similar even when the exact words are different.
