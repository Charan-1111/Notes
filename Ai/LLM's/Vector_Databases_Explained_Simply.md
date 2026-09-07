# Vector Databases Explained Simply

## 1. What is a vector database?

A **vector database** stores vectors and searches for vectors that are similar to one another.

A vector is a list of numbers created by an **embedding model**:

```text
"Reset your password"  → [0.82, 0.15, 0.61]
"Change login details" → [0.78, 0.19, 0.58]
"Download an invoice"  → [0.11, 0.91, 0.24]
```

The first two vectors are close because the sentences have similar meanings. The third represents a different topic.

Real vectors usually contain hundreds or thousands of numbers. These short vectors are only for explanation.

## 2. Why do we need one?

Imagine a support system containing one million articles. A user searches for:

```text
I forgot my login secret
```

The relevant article is:

```text
How to reset your password
```

A normal keyword search may struggle because the words are different. A vector database compares their **meaning** through embeddings and can find the password article.

It is optimized to perform this similarity search quickly, even across millions of vectors.

## 3. What is stored?

A vector database usually stores three things:

```json
{
  "id": "article-101",
  "vector": [0.82, 0.15, 0.61],
  "metadata": {
    "title": "Reset your password",
    "category": "account",
    "language": "en"
  }
}
```

- `id` uniquely identifies the item.
- `vector` represents the meaning of its text.
- `metadata` contains useful information for filtering or displaying the result.

The database may also store the original text, or the application may keep it elsewhere.

## 4. How does a search work?

Suppose these documents have already been embedded and stored:

```text
A: "Reset your account password"
B: "Download your monthly invoice"
C: "Cancel your subscription"
```

The user searches for:

```text
I cannot remember my login password
```

The application performs these steps:

```text
User query
   ↓
Embedding model creates a query vector
   ↓
Vector database compares it with stored vectors
   ↓
Database returns the closest matches
```

Example results:

```text
A: Reset your account password  → score 0.95
C: Cancel your subscription     → score 0.25
B: Download your invoice        → score 0.10
```

Document A is returned first because its meaning is closest to the query.

## 5. Similarity score

A vector database can use methods such as **cosine similarity**, **dot product**, or **Euclidean distance**.

You do not usually calculate these yourself. You choose a method supported by the database and request the nearest results.

For cosine similarity, a score closer to `1` normally means greater similarity:

```text
0.95 → highly similar
0.65 → somewhat related
0.10 → mostly unrelated
```

Scores are model- and data-dependent. Test with your own documents instead of assuming one score is always a perfect cutoff.

## 6. Metadata filtering

Similarity alone may not be enough.

Suppose the database contains help articles for several languages. The user needs only English account articles.

```text
Vector search: meaning similar to "forgot password"
Filters:       language = "en" AND category = "account"
```

The database first limits eligible records using metadata and then returns the most similar ones. This prevents an otherwise relevant Spanish article or billing article from appearing.

## 7. Vector databases in RAG

Vector databases are often used in **Retrieval-Augmented Generation (RAG)**.

Example: a chatbot answers questions from an employee handbook.

### Preparing the handbook

1. Split the handbook into smaller text chunks.
2. Create an embedding for each chunk.
3. Store every chunk and vector in the vector database.

```text
Handbook → chunks → embeddings → vector database
```

### Answering a question

```text
Question: "How many casual leaves do employees receive?"
                         ↓
Create the question embedding
                         ↓
Find similar handbook chunks
                         ↓
Retrieved: "Employees receive 12 casual-leave days per year."
                         ↓
Send question + retrieved text to the LLM
                         ↓
LLM generates the answer
```

The responsibilities are different:

- The **embedding model** converts text into vectors.
- The **vector database** stores and retrieves similar vectors.
- The **LLM** generates the final human-readable answer.

## 8. Simplified Go example

The exact client depends on the database, but the application flow can look like this:

```go
package search

import (
	"context"
	"fmt"
)

type Embedder interface {
	Embed(ctx context.Context, text string) ([]float32, error)
}

type SearchResult struct {
	ID      string
	Text    string
	Score   float32
}

type VectorDB interface {
	Search(
		ctx context.Context,
		vector []float32,
		limit int,
		filters map[string]string,
	) ([]SearchResult, error)
}

func SearchArticles(
	ctx context.Context,
	embedder Embedder,
	db VectorDB,
	query string,
) ([]SearchResult, error) {
	// The same embedding model used for documents should embed the query.
	queryVector, err := embedder.Embed(ctx, query)
	if err != nil {
		return nil, fmt.Errorf("embed query: %w", err)
	}

	// Return the three closest English account articles.
	results, err := db.Search(ctx, queryVector, 3, map[string]string{
		"language": "en",
		"category": "account",
	})
	if err != nil {
		return nil, fmt.Errorf("search vector database: %w", err)
	}

	return results, nil
}
```

For the query `I forgot my login password`, the first result might be:

```go
SearchResult{
    ID:    "article-101",
    Text:  "How to reset your password",
    Score: 0.95,
}
```

The interfaces keep provider-specific code outside the business logic, making the service easier to test and change.

## 9. Vector database vs normal database

| Question | Normal database | Vector database |
|---|---|---|
| Find user with ID `101` | Excellent | Not its main purpose |
| Find orders created today | Excellent | Not its main purpose |
| Find articles with similar meaning | Limited | Excellent |
| Store structured transactions | Excellent | Usually not preferred |

A vector database does not automatically replace PostgreSQL or MySQL. Many applications use both:

```text
PostgreSQL      → users, payments, orders
Vector database → embeddings and similarity search
```

Some relational databases, such as PostgreSQL with a vector extension, can support both regular data and vector search.

## 10. Important practical points

- Use the **same embedding model** for stored documents and search queries.
- Split large documents into meaningful chunks before embedding them.
- Store metadata so results can be filtered.
- Retrieve only the top few useful results instead of everything.
- Measure result quality using real user questions.
- Do not put sensitive information into an external service without checking its privacy requirements.

## Quick summary

```text
Embedding model: text → vector
Vector database: store vectors → find similar vectors
LLM: relevant text + question → final answer
```

A vector database is best understood as a database designed to answer:

> “Which stored items have meanings most similar to this query?”
