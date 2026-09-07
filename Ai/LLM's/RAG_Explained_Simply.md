# RAG Explained Simply

## 1. What is RAG?

**RAG** stands for **Retrieval-Augmented Generation**.

It means:

1. **Retrieve** useful information from your documents.
2. Give that information to an LLM.
3. Ask the LLM to **generate** an answer from it.

```text
User question
     ↓
Retrieve relevant information
     ↓
Question + retrieved information
     ↓
LLM generates an answer
```

In simple words:

> RAG lets an LLM look up relevant information before answering.

## 2. A simple example

Imagine that your company handbook says:

```text
Employees receive 12 casual-leave days per year.
```

An employee asks:

```text
How many casual leaves do I get?
```

Without RAG, the LLM may not know your company's policy and could guess incorrectly.

With RAG:

1. The system searches the handbook.
2. It retrieves the sentence about 12 casual-leave days.
3. It sends the question and sentence to the LLM.
4. The LLM answers: `You receive 12 casual-leave days per year.`

The answer comes from the company's document rather than only from the LLM's training knowledge.

## 3. Why use RAG?

An LLM has several limitations:

- It may not know private company data.
- Its training knowledge may be outdated.
- It may confidently produce an incorrect answer, called a **hallucination**.
- Training it again whenever documents change is expensive and slow.

RAG helps because documents can be added or updated without retraining the LLM.

Example:

```text
Old policy: 10 casual-leave days
New policy: 12 casual-leave days
```

Update the stored document and its embedding. The next retrieval can use the new policy.

## 4. The two main stages of RAG

RAG has two main stages:

1. **Indexing:** prepare and store documents.
2. **Retrieval and generation:** answer user questions.

## 5. Stage 1: Indexing documents

Suppose we have a 100-page employee handbook.

### Step 1: Load the document

Read the content from PDF, Markdown, HTML, or another source.

```text
employee-handbook.pdf → extracted text
```

### Step 2: Split it into chunks

A large document is divided into smaller, meaningful sections.

```text
Chunk 1: Leave policy
Chunk 2: Remote-work policy
Chunk 3: Health insurance
```

This is called **chunking**. It allows the system to retrieve only the relevant section.

### Step 3: Create embeddings

An embedding model converts every chunk into a vector—a list of numbers representing its meaning.

```text
"Employees receive 12 casual-leave days"
                    ↓
          [0.82, 0.14, 0.67, ...]
```

### Step 4: Store the chunks

Store each vector with its text and metadata in a vector database.

```json
{
  "id": "handbook-leave-01",
  "text": "Employees receive 12 casual-leave days per year.",
  "vector": [0.82, 0.14, 0.67],
  "metadata": {
    "document": "employee-handbook.pdf",
    "section": "Leave Policy",
    "country": "India"
  }
}
```

The complete indexing flow is:

```text
Documents → chunks → embeddings → vector database
```

## 6. Stage 2: Retrieval and generation

Now the user asks:

```text
How many casual leaves do employees in India receive?
```

### Step 1: Embed the question

Use the same embedding model to convert the question into a vector.

```text
Question → [0.80, 0.18, 0.63, ...]
```

### Step 2: Retrieve similar chunks

Search the vector database for chunks with vectors close to the question vector.

Also apply a metadata filter:

```text
country = "India"
```

The most relevant result may be:

```text
Employees receive 12 casual-leave days per year.
```

### Step 3: Build the prompt

Combine instructions, retrieved context, and the question:

```text
Answer using only the supplied context.
If the answer is missing, say that you do not know.

Context:
Employees receive 12 casual-leave days per year.

Question:
How many casual leaves do employees in India receive?
```

### Step 4: Generate the answer

Send the prompt to the LLM:

```text
Employees in India receive 12 casual-leave days per year.
```

The complete runtime flow is:

```text
Question → embedding → retrieval → prompt → LLM → answer
```

## 7. The role of each component

| Component | Responsibility |
|---|---|
| Chunker | Splits large documents into useful sections |
| Embedding model | Converts text into vectors |
| Vector database | Stores vectors and finds similar chunks |
| Retriever | Selects the most relevant chunks |
| LLM | Reads the context and generates the final answer |

The vector database does not generate answers, and the LLM does not normally search the database by itself. Your application connects these components.

## 8. A simplified Go design

The exact clients depend on your providers, but the service flow can look like this:

```go
package rag

import (
	"context"
	"fmt"
	"strings"
)

type Embedder interface {
	Embed(ctx context.Context, text string) ([]float32, error)
}

type Chunk struct {
	Text  string
	Score float32
}

type VectorStore interface {
	Search(
		ctx context.Context,
		vector []float32,
		limit int,
		filters map[string]string,
	) ([]Chunk, error)
}

type LLM interface {
	Generate(ctx context.Context, prompt string) (string, error)
}

type Service struct {
	embedder Embedder
	store    VectorStore
	llm      LLM
}

func (s *Service) Answer(
	ctx context.Context,
	question string,
	tenantID string,
) (string, error) {
	// 1. Convert the question into a vector.
	queryVector, err := s.embedder.Embed(ctx, question)
	if err != nil {
		return "", fmt.Errorf("embed question: %w", err)
	}

	// 2. Retrieve the three closest chunks for this tenant.
	chunks, err := s.store.Search(ctx, queryVector, 3, map[string]string{
		"tenant_id": tenantID,
	})
	if err != nil {
		return "", fmt.Errorf("retrieve context: %w", err)
	}

	if len(chunks) == 0 {
		return "I could not find this information.", nil
	}

	// 3. Combine retrieved chunks into context.
	contextParts := make([]string, 0, len(chunks))
	for _, chunk := range chunks {
		contextParts = append(contextParts, chunk.Text)
	}

	prompt := fmt.Sprintf(`Answer using only the context below.
If the answer is not present, say you do not know.

Context:
%s

Question:
%s`, strings.Join(contextParts, "\n\n"), question)

	// 4. Ask the LLM to generate the final answer.
	answer, err := s.llm.Generate(ctx, prompt)
	if err != nil {
		return "", fmt.Errorf("generate answer: %w", err)
	}

	return answer, nil
}
```

For this question:

```text
How long is a password reset link valid?
```

the retriever might find:

```text
Password reset links expire after 15 minutes.
```

The LLM then receives that sentence as context and produces the final response.

The interfaces keep provider-specific code separate, making the service easier to test and change.

## 9. RAG vs fine-tuning

RAG and fine-tuning solve different problems.

| Need | Better starting choice |
|---|---|
| Answer from private documents | RAG |
| Use frequently changing information | RAG |
| Show sources for an answer | RAG |
| Teach a consistent response style | Fine-tuning may help |
| Teach a specialized output format repeatedly | Fine-tuning may help |

Fine-tuning changes how a model behaves. RAG gives the model information at request time.

For a company-document chatbot, RAG is usually the better starting point.

## 10. Important practical points

### Use good chunks

Chunks should contain complete ideas. If a sentence is split badly, retrieval may lose important context.

### Retrieve only useful context

Too many chunks increase token usage and may confuse the model. Start with a small Top-K, such as `3` to `5`, and test it.

### Apply metadata filters

In a multi-tenant system, always filter using `tenant_id` or `organization_id`. Otherwise, one customer's documents might be retrieved for another customer.

### Handle missing information

Tell the LLM not to invent an answer when the context does not contain it.

### Return sources

Store document name, page number, and section with each chunk so the final answer can cite its source.

### Treat retrieved text as untrusted

Documents can contain incorrect or malicious instructions. Clearly separate system instructions from retrieved content, enforce access checks in application code, and never let retrieved text override security rules.

## 11. Common RAG problems

### The correct chunk is not retrieved

Possible improvements:

- improve chunk boundaries;
- combine keyword and semantic search;
- retrieve more candidates and rerank them;
- add useful titles and metadata.

### The correct chunk is retrieved, but the answer is wrong

Possible improvements:

- make the prompt clearer;
- remove irrelevant chunks;
- instruct the model to use only the context;
- use a model better suited to the task.

### The answer is outdated

Possible improvements:

- update the source document;
- recreate embeddings for changed chunks;
- remove old chunks from the vector database.

## 12. Quick summary

```text
INDEXING
Documents → chunking → embeddings → vector database

QUESTION TIME
Question → embedding → retrieve chunks → build prompt → LLM → answer
```

RAG is simply:

> Search for relevant information first, then let the LLM answer using that information.
