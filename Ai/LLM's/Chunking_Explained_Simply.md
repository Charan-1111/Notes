# Chunking Explained Simply

## 1. What is chunking?

**Chunking** means splitting a large document into smaller pieces called **chunks**.

Example:

```text
Employee Handbook (100 pages)
          ↓ chunking
Chunk 1: Leave policy
Chunk 2: Remote-work policy
Chunk 3: Health insurance
Chunk 4: Resignation process
```

Instead of treating the entire handbook as one large block, the application works with smaller, focused sections.

## 2. Why is chunking needed?

Imagine a user asks:

```text
How many casual-leave days do I get?
```

Only the leave-policy section is needed. Sending all 100 pages to an LLM would:

- use more tokens;
- cost more;
- take longer;
- include unrelated information;
- make the correct detail harder to find.

With chunking, the system retrieves only the useful section:

```text
Employees receive 12 casual-leave days per year.
```

## 3. How chunking fits into RAG

During document preparation:

```text
Document
   ↓ split
Chunks
   ↓ embedding model
Vectors
   ↓ store
Vector database
```

When a user asks a question:

```text
Question → embedding → find similar chunks
                           ↓
              relevant chunks + question
                           ↓
                          LLM
                           ↓
                         Answer
```

Chunking happens before embeddings are created. Each chunk gets its own embedding.

## 4. Why not use one large chunk?

Suppose one chunk contains the complete handbook:

```text
Leave policy + payroll + insurance + security + resignation + ...
```

Its embedding represents many topics at once. A search for casual leave may not match it as strongly as a focused leave-policy chunk.

Smaller, meaningful chunks produce more precise search results.

## 5. Why not use extremely small chunks?

Consider this sentence:

```text
Employees receive 12 casual-leave days per year.
```

Bad splitting:

```text
Chunk 1: Employees receive 12
Chunk 2: casual-leave days per year.
```

Neither chunk contains the complete fact. If only one is retrieved, the LLM may not understand it correctly.

The goal is to create chunks that are small enough for precise search but large enough to preserve meaning.

## 6. Common chunking strategies

### A. Fixed-size chunking

Split after a fixed number of characters, words, or tokens.

```text
Every 200 words → one chunk
```

This method is simple, but it can cut a sentence or paragraph in the middle.

Use it when the text has little structure or when building an early prototype.

### B. Recursive chunking

Try natural boundaries in order:

```text
Section → paragraph → sentence → word
```

For example, keep a paragraph together if it fits the limit. If it is too large, split it into sentences.

This usually preserves meaning better than blindly splitting every fixed number of characters.

### C. Structure-based chunking

Split using the document's structure:

```text
# Leave Policy          → Chunk 1
# Remote Work Policy    → Chunk 2
# Insurance             → Chunk 3
```

This is useful for Markdown, HTML, manuals, and documentation because headings already separate topics.

### D. Semantic chunking

Split whenever the meaning or topic changes.

```text
Paragraphs about leave       → one chunk
Paragraphs about payroll     → another chunk
```

This can produce meaningful chunks, but it is more complex and may require an embedding model or LLM.

## 7. What is chunk overlap?

**Overlap** means repeating a small part of the previous chunk in the next chunk.

Without overlap:

```text
Chunk 1: The password reset link is valid for
Chunk 2: 15 minutes. After that, request a new link.
```

The important sentence was cut across two chunks.

With overlap:

```text
Chunk 1: The password reset link is valid for 15 minutes.
Chunk 2: valid for 15 minutes. After that, request a new link.
```

Both chunks preserve enough context.

Overlap improves continuity, but too much overlap creates duplicate data, increases storage, and may return repetitive results.

## 8. Chunk size example

There is no perfect chunk size for every application.

As a reasonable starting experiment for text documents:

```text
Chunk size:  300–500 tokens
Overlap:     50–100 tokens
```

These are starting values, not universal rules. Test them using real questions from your application.

Different content needs different choices:

| Content | Sensible boundary |
|---|---|
| FAQ | One question and its answer |
| API documentation | One endpoint or section |
| Legal document | One clause or subsection |
| Source code | One function, method, or class |
| Conversation | A small group of related messages |

## 9. Store metadata with every chunk

A chunk should keep information about where it came from:

```json
{
  "id": "handbook-leave-02",
  "text": "Employees receive 12 casual-leave days per year.",
  "metadata": {
    "document": "employee-handbook.pdf",
    "section": "Leave Policy",
    "page": 18,
    "chunk_index": 2
  }
}
```

Metadata lets the application:

- show the source to the user;
- filter by document or section;
- open the correct page;
- combine neighboring chunks when more context is needed.

## 10. Simple Go example

The following example splits text by paragraphs while keeping chunks below a word limit:

```go
package chunking

import "strings"

func SplitByParagraph(text string, maxWords int) []string {
	paragraphs := strings.Split(text, "\n\n")
	chunks := make([]string, 0)
	current := make([]string, 0)
	wordCount := 0

	for _, paragraph := range paragraphs {
		paragraph = strings.TrimSpace(paragraph)
		if paragraph == "" {
			continue
		}

		paragraphWords := len(strings.Fields(paragraph))

		// Save the current chunk before adding a paragraph that exceeds the limit.
		if wordCount > 0 && wordCount+paragraphWords > maxWords {
			chunks = append(chunks, strings.Join(current, "\n\n"))
			current = nil
			wordCount = 0
		}

		current = append(current, paragraph)
		wordCount += paragraphWords
	}

	// Store the final unfinished chunk.
	if len(current) > 0 {
		chunks = append(chunks, strings.Join(current, "\n\n"))
	}

	return chunks
}
```

Example input:

```text
Employees receive 12 casual-leave days per year.

Leave requests must be submitted through the employee portal.

Employees may work remotely for two days each week.
```

With a suitable word limit, the first two related paragraphs can remain together, while the remote-work paragraph becomes another chunk.

This is a learning example. Production code should also handle paragraphs larger than `maxWords`, token-based limits, overlap, and metadata.

## 11. Common mistakes

- Creating one chunk for the entire document.
- Splitting in the middle of sentences or important facts.
- Making chunks so small that they lose context.
- Adding so much overlap that most content is duplicated.
- Forgetting source metadata.
- Choosing a chunk size without testing retrieval quality.

## Quick summary

```text
Large document
      ↓ chunking
Small meaningful sections
      ↓ embeddings
Vectors stored in a vector database
      ↓ similarity search
Relevant sections sent to the LLM
```

Chunking is simply:

> Splitting large content into small, meaningful pieces so the correct information can be found and used efficiently.
