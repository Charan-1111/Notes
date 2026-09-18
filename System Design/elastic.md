# Elasticsearch Explained for a Junior Go Developer

> This guide assumes that **Elastic** refers to **Elasticsearch**.

## 1. What Is Elasticsearch?

Elasticsearch is a system used to **store, search, filter, and analyze large amounts of data quickly**.

Its strongest feature is text search.

Imagine an e-commerce application containing millions of products:

```text
Apple iPhone 17 Pro
Apple MacBook Pro
Samsung Galaxy S26
iPhone Back Cover
```

If a user searches for:

```text
iphone
```

Elasticsearch can quickly return:

```text
Apple iPhone 17 Pro
iPhone Back Cover
```

Elasticsearch also supports:

- Partial matching
- Typo-tolerant search
- Filtering
- Sorting
- Autocomplete
- Search-result ranking
- Log analysis
- Data aggregations
- Semantic search using embeddings

---

## 2. Why Not Use PostgreSQL for Search?

PostgreSQL can perform simple searches:

```sql
SELECT *
FROM products
WHERE name LIKE '%iphone%';
```

This may be sufficient for small datasets and basic search requirements.

However, advanced searching becomes more difficult when you need:

- Millions of records
- Typo tolerance
- Relevance ranking
- Searching across multiple fields
- Autocomplete
- Word variations
- Complex filters
- Aggregations

Elasticsearch is designed specifically for these operations.

A common architecture is:

```text
PostgreSQL    → Source of truth
Elasticsearch → Searchable copy of the data
```

PostgreSQL stores the authoritative business data.

Elasticsearch stores a copy of the data optimized for searching.

---

## 3. Core Elasticsearch Concepts

## 3.1 Document

A **document** is one JSON object stored in Elasticsearch.

Example product document:

```json
{
  "id": 101,
  "name": "Apple iPhone 17 Pro",
  "description": "A premium smartphone from Apple",
  "category": "mobile",
  "price": 129999,
  "in_stock": true
}
```

A document is roughly similar to a row in a relational database.

---

## 3.2 Index

An **index** is a collection of related documents.

Examples:

```text
products
users
orders
application-logs
```

The `products` index contains product documents.

A rough comparison is:

| Relational database | Elasticsearch |
|---|---|
| Table | Index |
| Row | Document |
| Column | Field |
| Schema | Mapping |
| SQL query | Elasticsearch Query DSL |

This comparison is useful for learning, although Elasticsearch and relational databases work differently internally.

---

## 3.3 Field

A **field** is one property inside a document.

```json
{
  "name": "Apple iPhone 17 Pro",
  "price": 129999
}
```

In this document:

- `name` is a field.
- `price` is a field.

---

## 3.4 Mapping

A **mapping** defines the data type and search behavior of each field.

Example:

```json
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "category": {
        "type": "keyword"
      },
      "price": {
        "type": "double"
      },
      "in_stock": {
        "type": "boolean"
      }
    }
  }
}
```

Common field types include:

| Type | Used for |
|---|---|
| `text` | Full-text searching |
| `keyword` | Exact matching, filtering and sorting |
| `integer` | Whole numbers |
| `double` | Decimal numbers |
| `boolean` | `true` or `false` |
| `date` | Dates and timestamps |
| `object` | JSON objects |
| `nested` | Arrays of objects requiring independent matching |
| `dense_vector` | Embeddings and semantic search |

---

## 4. `text` vs `keyword`

Understanding the difference between `text` and `keyword` is very important.

## 4.1 `text`

Use `text` for content users search using words.

Example:

```json
{
  "name": "Apple iPhone 17 Pro"
}
```

A `text` field is analyzed and divided into searchable terms.

Conceptually:

```text
Apple iPhone 17 Pro
        ↓
apple, iphone, 17, pro
```

Therefore, searching for `iphone` can match the complete product name.

Use `text` for:

- Product names
- Descriptions
- Article content
- Comments
- Messages

---

## 4.2 `keyword`

Use `keyword` when you need an exact value.

Example:

```json
{
  "category": "mobile"
}
```

Use `keyword` for:

- Filtering
- Sorting
- Grouping
- IDs
- Email addresses
- Status values
- Categories

A `keyword` field is not normally divided into separate words.

---

## 4.3 Using Both Types

A field can support both full-text search and exact operations:

```json
{
  "name": {
    "type": "text",
    "fields": {
      "keyword": {
        "type": "keyword"
      }
    }
  }
}
```

You can then use:

```text
name         → Full-text search
name.keyword → Exact matching and sorting
```

---

## 5. How Elasticsearch Searches Quickly

Elasticsearch does not scan every complete document during every search.

It creates an **inverted index**.

Suppose we store these documents:

```text
Document 1: Apple iPhone
Document 2: Apple MacBook
Document 3: Samsung Phone
```

Elasticsearch creates a structure similar to:

| Word | Documents containing the word |
|---|---|
| apple | 1, 2 |
| iphone | 1 |
| macbook | 2 |
| samsung | 3 |
| phone | 3 |

When someone searches for `apple`, Elasticsearch already knows that documents `1` and `2` contain that word.

It does not need to scan every document from beginning to end.

This inverted index is one of the main reasons text search is fast.

---

## 6. Analysis, Analyzers and Tokens

Before Elasticsearch indexes text, it analyzes the text.

Consider this value:

```text
The Apple iPhone Is Powerful
```

An analyzer may produce:

```text
the
apple
iphone
is
powerful
```

These smaller values are called **tokens**.

An analyzer commonly performs three steps:

1. Character filtering
2. Tokenization
3. Token filtering

Example:

```text
"Apple iPhone!"
        ↓ lowercase
"apple iphone!"
        ↓ tokenize
["apple", "iphone"]
```

The same or a compatible analyzer is generally used when searching.

This allows the search terms to match the stored tokens.

---

## 7. Indexing a Document

Storing a document in Elasticsearch is called **indexing**.

Example request:

```http
PUT /products/_doc/101
Content-Type: application/json

{
  "name": "Apple iPhone 17 Pro",
  "category": "mobile",
  "price": 129999,
  "in_stock": true
}
```

In this request:

- `products` is the index.
- `_doc` is the document endpoint.
- `101` is the document ID.
- The request body contains the document.

If another document is indexed using the same ID, the existing document is replaced or updated depending on the operation used.

---

## 8. Basic Search Queries

Elasticsearch queries are written using a JSON-based query language called **Query DSL**.

## 8.1 Match Query

Use `match` for full-text search:

```json
{
  "query": {
    "match": {
      "name": "iphone pro"
    }
  }
}
```

Elasticsearch analyzes `iphone pro` and finds relevant documents.

Use `match` with `text` fields.

---

## 8.2 Term Query

Use `term` for an exact value:

```json
{
  "query": {
    "term": {
      "category": "mobile"
    }
  }
}
```

Use `term` mainly with:

- `keyword` fields
- Numeric fields
- Boolean fields
- Exact IDs

A common mistake is using `term` on an analyzed `text` field and expecting full-text search behavior.

A simple rule is:

```text
match → analyzed text search
term  → exact value matching
```

---

## 8.3 Range Query

Find products within a price range:

```json
{
  "query": {
    "range": {
      "price": {
        "gte": 50000,
        "lte": 150000
      }
    }
  }
}
```

Here:

- `gte` means greater than or equal.
- `lte` means less than or equal.
- `gt` means greater than.
- `lt` means less than.

---

## 8.4 Multi-Match Query

Use `multi_match` to search across multiple fields:

```json
{
  "query": {
    "multi_match": {
      "query": "iphone pro",
      "fields": [
        "name",
        "description"
      ]
    }
  }
}
```

This searches for `iphone pro` in both the `name` and `description` fields.

---

## 9. Combining Search and Filters

Suppose the user wants:

```text
Search for "iphone"
Category must be "mobile"
Price must be at most ₹150,000
Product must be in stock
```

The query could be:

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "iphone"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "category": "mobile"
          }
        },
        {
          "range": {
            "price": {
              "lte": 150000
            }
          }
        },
        {
          "term": {
            "in_stock": true
          }
        }
      ]
    }
  }
}
```

### How It Works

1. `must` searches for documents containing `iphone`.
2. `filter` removes documents that do not satisfy the conditions.
3. Filters generally do not change the relevance score.
4. Matching documents are returned in relevance order.

Useful Boolean clauses include:

| Clause | Meaning |
|---|---|
| `must` | The condition must match and may affect the score |
| `filter` | The condition must match but does not affect the score |
| `should` | Optional match that can improve the score |
| `must_not` | The condition must not match |

---

## 10. Relevance Scoring

Elasticsearch does not only find matching documents. It also calculates how relevant each result is.

Every search result may contain a value such as:

```json
{
  "_score": 2.48
}
```

A higher score normally means the document is more relevant to the search query.

For the search:

```text
iphone pro
```

This product:

```text
Apple iPhone 17 Pro
```

should normally rank above:

```text
Protective cover compatible with iPhone
```

Search scoring considers factors such as:

- How many query words matched
- How frequently a word occurs in the document
- How rare the word is across all documents
- The length of the field
- Field importance or boosting

You can increase the importance of a field:

```json
{
  "query": {
    "multi_match": {
      "query": "iphone pro",
      "fields": [
        "name^3",
        "description"
      ]
    }
  }
}
```

`name^3` means a match in the `name` field should be treated as more important than a match in the `description` field.

---

## 11. Aggregations

Aggregations calculate summaries from matching documents.

Suppose the product data contains:

```text
mobile
mobile
laptop
mobile
laptop
```

A terms aggregation can produce:

```text
mobile: 3
laptop: 2
```

Example query:

```json
{
  "size": 0,
  "aggs": {
    "products_by_category": {
      "terms": {
        "field": "category"
      }
    }
  }
}
```

`size: 0` means that we only want aggregation results and do not need the individual documents.

Common aggregation use cases include:

- Products per category
- Average product price
- Orders per day
- Sales per region
- Requests per HTTP status
- Maximum API latency
- Average response time

This is one reason Elasticsearch is useful for dashboards and log analysis.

---

## 12. Elasticsearch Architecture

Elasticsearch is designed to run as a distributed system.

## 12.1 Node

A **node** is one running Elasticsearch instance.

```text
Node 1
Node 2
Node 3
```

Each node can store data and process requests.

---

## 12.2 Cluster

A **cluster** is a group of Elasticsearch nodes working together.

```text
Elasticsearch Cluster
├── Node 1
├── Node 2
└── Node 3
```

The cluster distributes data and requests across its nodes.

---

## 12.3 Shard

An index can be split into smaller pieces called **shards**.

Example:

```text
products index
├── Shard 0
├── Shard 1
└── Shard 2
```

These shards can be stored on different nodes.

Shards allow Elasticsearch to:

- Store more data
- Search multiple data parts in parallel
- Distribute workload across nodes

---

## 12.4 Replica

A **replica** is a copy of a shard.

Example:

```text
Primary Shard 0 → Node 1
Replica Shard 0 → Node 2
```

Replicas provide:

- High availability
- Additional search capacity
- Protection when a node fails

A replica should be stored on a different node from its primary shard.

---

## 13. What Happens During a Search?

Consider a three-node Elasticsearch cluster:

```text
Go API
  │
  ▼
Coordinating Elasticsearch Node
  ├── Search Shard 0
  ├── Search Shard 1
  └── Search Shard 2
```

A search roughly follows these steps:

1. The Go API sends a query to an Elasticsearch node.
2. That node acts as the coordinating node for the request.
3. It forwards the query to the relevant shards.
4. Each shard searches its local data.
5. Each shard returns its matching results.
6. The coordinating node combines the results.
7. It sorts them by score or the requested sort order.
8. The final response is returned to the Go API.

---

## 14. Elasticsearch Is Near Real-Time

After indexing a document, it may not become searchable at the exact same moment.

Elasticsearch periodically performs a **refresh** that makes recently indexed documents searchable.

Therefore, Elasticsearch is described as:

```text
Near real-time
```

It is not strictly real-time.

You should not normally force a refresh after every write because frequent refreshes can reduce indexing performance.

---

# Using Elasticsearch with Go

Elasticsearch provides an HTTP API.

We can communicate with it using Go's standard `net/http` package.

---

## 15. Product Model

```go
type Product struct {
	ID          int     `json:"id"`
	Name        string  `json:"name"`
	Description string  `json:"description"`
	Category    string  `json:"category"`
	Price       float64 `json:"price"`
	InStock     bool    `json:"in_stock"`
}
```

The JSON tags define the field names sent to Elasticsearch.

For example:

```go
Name string `json:"name"`
```

becomes:

```json
{
  "name": "Apple iPhone 17 Pro"
}
```

---

## 16. Search Service

```go
type SearchService struct {
	baseURL string
	client  *http.Client
}

func NewSearchService(
	baseURL string,
	client *http.Client,
) *SearchService {
	return &SearchService{
		baseURL: baseURL,
		client:  client,
	}
}
```

### Explanation

The service contains:

- `baseURL`: The Elasticsearch server address.
- `client`: A reusable HTTP client.

Example Elasticsearch address:

```text
http://localhost:9200
```

The same HTTP client should be reused because Go can reuse the underlying network connections.

---

## 17. Indexing a Product from Go

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
)

type Product struct {
	ID          int     `json:"id"`
	Name        string  `json:"name"`
	Description string  `json:"description"`
	Category    string  `json:"category"`
	Price       float64 `json:"price"`
	InStock     bool    `json:"in_stock"`
}

type SearchService struct {
	baseURL string
	client  *http.Client
}

func NewSearchService(
	baseURL string,
	client *http.Client,
) *SearchService {
	return &SearchService{
		baseURL: baseURL,
		client:  client,
	}
}

func (s *SearchService) IndexProduct(
	ctx context.Context,
	product Product,
) error {
	body, err := json.Marshal(product)
	if err != nil {
		return fmt.Errorf("encode product: %w", err)
	}

	url := fmt.Sprintf(
		"%s/products/_doc/%d",
		s.baseURL,
		product.ID,
	)

	req, err := http.NewRequestWithContext(
		ctx,
		http.MethodPut,
		url,
		bytes.NewReader(body),
	)
	if err != nil {
		return fmt.Errorf("create request: %w", err)
	}

	req.Header.Set("Content-Type", "application/json")

	resp, err := s.client.Do(req)
	if err != nil {
		return fmt.Errorf("send request: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode >= http.StatusMultipleChoices {
		responseBody, _ := io.ReadAll(resp.Body)

		return fmt.Errorf(
			"elasticsearch returned status %d: %s",
			resp.StatusCode,
			responseBody,
		)
	}

	return nil
}
```

### Step 1: Convert the Product into JSON

```go
body, err := json.Marshal(product)
```

This converts the Go struct into JSON.

For example:

```json
{
  "id": 101,
  "name": "Apple iPhone 17 Pro",
  "description": "A premium smartphone",
  "category": "mobile",
  "price": 129999,
  "in_stock": true
}
```

If the conversion fails, the function returns an error.

---

### Step 2: Create the Document URL

```go
url := fmt.Sprintf(
	"%s/products/_doc/%d",
	s.baseURL,
	product.ID,
)
```

For product `101`, the resulting URL is:

```text
http://localhost:9200/products/_doc/101
```

Here:

- `products` is the index.
- `101` is the document ID.

---

### Step 3: Create a `PUT` Request

```go
req, err := http.NewRequestWithContext(
	ctx,
	http.MethodPut,
	url,
	bytes.NewReader(body),
)
```

This creates an HTTP request containing the product JSON.

The request context allows the operation to be cancelled when:

- The user cancels the original request.
- The request deadline is exceeded.
- The server shuts down.

---

### Step 4: Set the Content Type

```go
req.Header.Set("Content-Type", "application/json")
```

This tells Elasticsearch that the request body contains JSON.

---

### Step 5: Send the Request

```go
resp, err := s.client.Do(req)
```

The shared HTTP client sends the request to Elasticsearch.

---

### Step 6: Close the Response Body

```go
defer resp.Body.Close()
```

The response body must be closed after it is used so that resources and network connections can be reused correctly.

---

### Step 7: Check for Elasticsearch Errors

```go
if resp.StatusCode >= http.StatusMultipleChoices {
```

Successful responses normally have a status code below `300`.

Elasticsearch may return errors for:

- Invalid mappings
- Invalid JSON
- Authentication failures
- Missing indexes
- Cluster problems

The code reads and returns the Elasticsearch error response instead of silently ignoring it.

---

## 18. Calling the Indexing Function

```go
func main() {
	client := &http.Client{}

	searchService := NewSearchService(
		"http://localhost:9200",
		client,
	)

	product := Product{
		ID:          101,
		Name:        "Apple iPhone 17 Pro",
		Description: "A premium smartphone from Apple",
		Category:    "mobile",
		Price:       129999,
		InStock:     true,
	}

	err := searchService.IndexProduct(
		context.Background(),
		product,
	)
	if err != nil {
		panic(err)
	}

	fmt.Println("Product indexed successfully")
}
```

This code:

1. Creates an HTTP client.
2. Creates the search service.
3. Creates a product.
4. Sends the product to Elasticsearch.
5. Stores it in the `products` index using ID `101`.

---

## 19. Searching Products from Go

First, define the part of the Elasticsearch response we need:

```go
type SearchResult struct {
	Hits struct {
		Hits []struct {
			ID     string  `json:"_id"`
			Score  float64 `json:"_score"`
			Source Product `json:"_source"`
		} `json:"hits"`
	} `json:"hits"`
}
```

An Elasticsearch result contains metadata around every matching document.

Example response:

```json
{
  "hits": {
    "hits": [
      {
        "_id": "101",
        "_score": 2.45,
        "_source": {
          "id": 101,
          "name": "Apple iPhone 17 Pro",
          "description": "A premium smartphone from Apple",
          "category": "mobile",
          "price": 129999,
          "in_stock": true
        }
      }
    ]
  }
}
```

The original product document is stored inside `_source`.

---

## 20. Product Search Function

```go
func (s *SearchService) SearchProducts(
	ctx context.Context,
	searchText string,
) ([]Product, error) {
	query := map[string]any{
		"query": map[string]any{
			"multi_match": map[string]any{
				"query": searchText,
				"fields": []string{
					"name^3",
					"description",
				},
			},
		},
	}

	body, err := json.Marshal(query)
	if err != nil {
		return nil, fmt.Errorf("encode query: %w", err)
	}

	url := s.baseURL + "/products/_search"

	req, err := http.NewRequestWithContext(
		ctx,
		http.MethodPost,
		url,
		bytes.NewReader(body),
	)
	if err != nil {
		return nil, fmt.Errorf(
			"create search request: %w",
			err,
		)
	}

	req.Header.Set("Content-Type", "application/json")

	resp, err := s.client.Do(req)
	if err != nil {
		return nil, fmt.Errorf(
			"send search request: %w",
			err,
		)
	}
	defer resp.Body.Close()

	if resp.StatusCode >= http.StatusMultipleChoices {
		responseBody, _ := io.ReadAll(resp.Body)

		return nil, fmt.Errorf(
			"elasticsearch returned status %d: %s",
			resp.StatusCode,
			responseBody,
		)
	}

	var result SearchResult

	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, fmt.Errorf(
			"decode search response: %w",
			err,
		)
	}

	products := make(
		[]Product,
		0,
		len(result.Hits.Hits),
	)

	for _, hit := range result.Hits.Hits {
		products = append(products, hit.Source)
	}

	return products, nil
}
```

### Step 1: Build the Search Query

```go
query := map[string]any{
	"query": map[string]any{
		"multi_match": map[string]any{
			"query": searchText,
			"fields": []string{
				"name^3",
				"description",
			},
		},
	},
}
```

This searches for the provided text inside:

- `name`
- `description`

The `name` field is boosted using:

```text
name^3
```

Therefore, a match in the product name is more important than a match in the product description.

---

### Step 2: Convert the Query into JSON

```go
body, err := json.Marshal(query)
```

The Go map becomes:

```json
{
  "query": {
    "multi_match": {
      "query": "iphone",
      "fields": [
        "name^3",
        "description"
      ]
    }
  }
}
```

---

### Step 3: Send the Query

The query is sent to:

```text
POST /products/_search
```

The `_search` endpoint searches the `products` index.

---

### Step 4: Decode the Response

```go
var result SearchResult

err := json.NewDecoder(resp.Body).Decode(&result)
```

This converts the Elasticsearch JSON response into the `SearchResult` Go struct.

---

### Step 5: Extract the Products

```go
for _, hit := range result.Hits.Hits {
	products = append(products, hit.Source)
}
```

Each search hit contains:

- Document ID
- Relevance score
- Original document

The loop extracts `_source` and returns a normal Go slice:

```go
[]Product
```

---

## 21. Calling the Search Function

```go
products, err := searchService.SearchProducts(
	context.Background(),
	"iphone pro",
)
if err != nil {
	panic(err)
}

for _, product := range products {
	fmt.Printf(
		"%d - %s - %.2f\n",
		product.ID,
		product.Name,
		product.Price,
	)
}
```

Possible output:

```text
101 - Apple iPhone 17 Pro - 129999.00
105 - iPhone 17 Pro Case - 1999.00
```

The product itself should normally appear before the phone case because its name is more relevant to the query.

---

## 22. Real Backend Architecture

A common application architecture looks like this:

```text
Client
  │
  ▼
Go API
  ├── PostgreSQL
  └── Elasticsearch
```

## Creating a Product

```text
1. Client sends POST /products
2. Go API validates the request
3. Product is stored in PostgreSQL
4. Product is indexed in Elasticsearch
5. API returns the response
```

## Searching for Products

```text
1. Client sends GET /products/search?q=iphone
2. Go API queries Elasticsearch
3. Elasticsearch returns matching products
4. Go API returns the results to the client
```

---

## 23. The Data Consistency Problem

Updating PostgreSQL and Elasticsearch directly in the same request can create a problem:

```text
PostgreSQL write succeeds
Elasticsearch write fails
```

Now PostgreSQL contains the product, but Elasticsearch does not.

Search results become inconsistent.

A stronger production design is:

```text
Go API
  │
  ▼
PostgreSQL
  │
  ▼
Outbox Event
  │
  ▼
Background Worker
  │
  ▼
Elasticsearch
```

The flow is:

1. The API stores the product in PostgreSQL.
2. It also stores an event in an outbox table.
3. A background worker reads the event.
4. The worker indexes the product in Elasticsearch.
5. If indexing fails, the worker retries.

This provides better reliability than performing two unrelated network writes inside the same API request.

---

## 24. Elasticsearch for Semantic Search

Traditional text search primarily matches words.

Semantic search matches **meaning** using embeddings.

Consider:

```text
Stored document:
"How do I change my password?"

User query:
"I cannot access my account"
```

The sentences may not contain the same words, but their meanings are related.

The indexing flow is:

```text
Document text
     ↓
Embedding model
     ↓
Embedding vector
     ↓
Store text and vector in Elasticsearch
```

The search flow is:

```text
User query
     ↓
Generate query embedding
     ↓
Compare it with stored vectors
     ↓
Return the nearest documents
```

An Elasticsearch mapping may contain:

```json
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text"
      },
      "embedding": {
        "type": "dense_vector",
        "dims": 768
      }
    }
  }
}
```

Here:

- `content` stores the original text.
- `embedding` stores its vector.
- `dims` represents the number of values in the vector.

This is directly useful for an embedding and cosine-similarity project.

Instead of loading every vector into the Go application and comparing them one by one, Elasticsearch can:

- Store the vectors
- Build a searchable vector index
- Find the nearest vectors
- Return the most semantically similar documents

---

## 25. Hybrid Search

Keyword search and semantic search each have different strengths.

### Keyword Search

Keyword search is useful when the exact terms matter.

Example:

```text
iPhone 17 Pro
```

It is good for:

- Product names
- Error codes
- Model numbers
- Exact technical terms

### Semantic Search

Semantic search is useful when meaning matters.

Example:

```text
Affordable phone with a good camera
```

It can find relevant products even when they do not contain all the same words.

### Hybrid Search

Hybrid search combines both approaches:

```text
Keyword search + Vector search = Hybrid search
```

This provides:

- Exact keyword matching
- Meaning-based matching
- Better overall relevance

For your semantic-search project, hybrid search would be a valuable next feature.

---

## 26. Updating a Document

A document can be partially updated using the `_update` endpoint:

```http
POST /products/_update/101
Content-Type: application/json

{
  "doc": {
    "price": 119999,
    "in_stock": true
  }
}
```

This updates only the provided fields.

The remaining fields are not removed.

---

## 27. Deleting a Document

Delete a product using its document ID:

```http
DELETE /products/_doc/101
```

This removes document `101` from the `products` index.

---

## 28. Pagination

Basic pagination uses `from` and `size`:

```json
{
  "from": 0,
  "size": 20,
  "query": {
    "match": {
      "name": "iphone"
    }
  }
}
```

Here:

- `from` is the number of results to skip.
- `size` is the number of results to return.

For the second page:

```json
{
  "from": 20,
  "size": 20
}
```

Deep pagination can become expensive.

For example:

```json
{
  "from": 100000,
  "size": 20
}
```

Elasticsearch may need to collect and sort a very large number of results before returning only 20 documents.

For deep or cursor-based pagination, use:

```text
search_after
```

---

## 29. Bulk Indexing

Sending thousands of documents one request at a time creates unnecessary network overhead.

Elasticsearch provides the Bulk API:

```http
POST /_bulk
Content-Type: application/x-ndjson

{"index":{"_index":"products","_id":"101"}}
{"name":"Apple iPhone 17 Pro","category":"mobile","price":129999}
{"index":{"_index":"products","_id":"102"}}
{"name":"Samsung Galaxy S26","category":"mobile","price":99999}
```

The Bulk API can index many documents in one request.

Use it for:

- Initial data migrations
- Large imports
- Background synchronization
- Rebuilding an index

Each bulk operation can succeed or fail independently, so always inspect the response for item-level errors.

---

## 30. Important Production Considerations

## Define Mappings Explicitly

Do not depend completely on automatically generated mappings.

For example, if Elasticsearch incorrectly maps a price as text, numeric range queries will not work correctly.

---

## Reuse the HTTP Client

Do not create a new HTTP client for every request.

Reuse one client so that Go can reuse network connections.

---

## Set Timeouts

Your Go HTTP client should use appropriate timeouts.

Example:

```go
client := &http.Client{
	Timeout: 5 * time.Second,
}
```

This prevents requests from waiting forever.

You should also use request contexts with deadlines.

---

## Use Bulk Indexing

Use the Bulk API when inserting or updating many documents.

Do not make thousands of individual HTTP requests unless necessary.

---

## Avoid Too Many Shards

Each shard consumes:

- Memory
- File descriptors
- CPU
- Cluster-management resources

More shards do not automatically provide better performance.

Shard count should be based on actual data size and traffic.

---

## Monitor the Cluster

Important measurements include:

- JVM heap usage
- Disk usage
- Search latency
- Indexing latency
- Rejected requests
- Shard health
- Node health
- Cluster health

Elasticsearch cluster health can commonly be:

| Status | Meaning |
|---|---|
| Green | All primary and replica shards are available |
| Yellow | Primary shards are available, but some replicas are missing |
| Red | One or more primary shards are unavailable |

---

## Handle Failed Indexing

Indexing can fail because of:

- Network failures
- Invalid mappings
- Invalid documents
- Authentication errors
- Disk problems
- Elasticsearch being unavailable

Use:

- Retries with backoff
- Dead-letter queues
- Background workers
- Structured logging
- Monitoring and alerts

Do not retry permanent errors, such as an invalid document format, forever.

---

## Protect Elasticsearch

An Elasticsearch cluster should not normally be publicly exposed.

Use:

- Authentication
- TLS
- Network restrictions
- Role-based access control
- Secret management

Application users should communicate with your Go API, not directly with Elasticsearch.

---

## 31. When Should You Use Elasticsearch?

Use Elasticsearch when you need:

- Full-text search
- Relevance ranking
- Search across millions of records
- Autocomplete
- Typo tolerance
- Log searching
- Fast filtering and aggregations
- Semantic search
- Hybrid search

Avoid introducing Elasticsearch when:

- Simple PostgreSQL queries are sufficient
- The dataset is small
- You only need exact ID lookups
- Strong transactions are essential
- The team cannot operate another distributed system
- Search is not an important feature

Elasticsearch adds powerful search capabilities, but it also adds infrastructure and operational complexity.

---

## 32. Elasticsearch Should Usually Not Replace PostgreSQL

Elasticsearch is not usually the best primary database for transactional business applications.

For example, an order-processing system may need:

- Transactions
- Foreign-key relationships
- Strong consistency
- Reliable constraints
- Complex relational updates

PostgreSQL is better suited for those responsibilities.

A useful mental model is:

```text
PostgreSQL    = Source of truth
Elasticsearch = Search index
```

If the Elasticsearch index is lost, you should ideally be able to rebuild it from PostgreSQL or stored events.

---

## 33. Final Mental Model

Think of Elasticsearch as a highly organized library:

- A **document** is a book.
- An **index** is a collection of books.
- A **field** is information about a book.
- A **mapping** defines how that information is stored.
- An **analyzer** separates text into searchable words.
- A **token** is one searchable word or term.
- An **inverted index** tells you which books contain each word.
- A **query** describes what you want to find.
- A **score** indicates how relevant each result is.
- A **shard** stores part of the collection.
- A **replica** is a backup copy of a shard.
- An **aggregation** summarizes the stored data.

For a semantic-search project, a good implementation path is:

```text
1. Store normal text documents
2. Add keyword-based search
3. Add filtering and pagination
4. Generate embeddings
5. Store embeddings using dense_vector
6. Add vector similarity search
7. Combine text and vector results
8. Build hybrid search
9. Add background synchronization
10. Add monitoring and error handling
```

The final architecture could look like:

```text
Client
  │
  ▼
Go API
  ├── PostgreSQL
  │     └── Source of truth
  │
  ├── Embedding Service
  │     └── Generates vectors
  │
  └── Elasticsearch
        ├── Keyword search
        ├── Vector search
        └── Hybrid search
```