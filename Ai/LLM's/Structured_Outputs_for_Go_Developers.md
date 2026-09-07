# Structured Outputs in LLM Applications

## 1. What is a structured output?

Normally, an LLM returns free-form text:

```text
This looks like a high-priority payment issue. The user wants a refund.
```

This is easy for a person to read, but difficult for backend code to use reliably.

A **structured output** follows a predefined structure, usually a JSON schema:

```json
{
  "category": "payment",
  "priority": "high",
  "requires_refund": true
}
```

Your Go application can safely decode this into a struct and use the individual fields.

```go
type SupportTicket struct {
	Category       string `json:"category"`
	Priority       string `json:"priority"`
	RequiresRefund bool   `json:"requires_refund"`
}
```

Think of it this way:

- Normal output: “Write an answer in your own format.”
- Structured output: “Fill in this exact form.”

---

## 2. Why do we need it?

Imagine that your backend asks an LLM to classify this message:

```text
I was charged twice. Please return the extra payment immediately.
```

Without a strict structure, the model might return any of these:

```text
Priority: High, Category: Payment
```

```text
This is probably an urgent billing problem.
```

```json
{"type":"billing","urgency":1}
```

All three mean similar things, but your code would need different parsing logic for each.

With structured outputs, your backend requests one contract:

```json
{
  "category": "payment",
  "priority": "high",
  "requires_refund": true
}
```

This is useful for:

- API responses
- Ticket or email classification
- Data extraction from documents
- Tool/function arguments
- Saving LLM results in a database
- Passing results to another service

---

## 3. The schema is the contract

A schema describes the shape and rules of the response.

```json
{
  "type": "object",
  "properties": {
    "category": {
      "type": "string",
      "enum": ["payment", "technical", "account", "other"]
    },
    "priority": {
      "type": "string",
      "enum": ["low", "medium", "high"]
    },
    "requires_refund": {
      "type": "boolean"
    }
  },
  "required": ["category", "priority", "requires_refund"],
  "additionalProperties": false
}
```

Important parts:

| Rule | Meaning |
|---|---|
| `type: object` | The result must be a JSON object |
| `properties` | Fields that the object may contain |
| `type: string` | The value must be text |
| `enum` | The value must be one of the listed choices |
| `required` | These fields must be present |
| `additionalProperties: false` | Unknown fields are not allowed |

The schema and Go struct should describe the same data.

---

## 4. Complete flow in a Go backend

```text
User message
    ↓
Go backend sends prompt + schema
    ↓
LLM generates schema-compliant JSON
    ↓
Go decodes JSON into a struct
    ↓
Application validates and uses the result
```

### Step 1: Define the Go type

```go
type SupportTicket struct {
	Category       string `json:"category"`
	Priority       string `json:"priority"`
	RequiresRefund bool   `json:"requires_refund"`
}
```

### Step 2: Ask the provider for a structured response

The exact SDK syntax differs between LLM providers, but the request usually contains:

```go
request := LLMRequest{
	Prompt: `Classify this support message:
             "I was charged twice. Please return the extra payment immediately."`,
	ResponseSchema: supportTicketSchema,
}
```

`LLMRequest` is illustrative. Use the structured-output or response-schema field supplied by your chosen SDK.

### Step 3: Decode the returned JSON

```go
func decodeTicket(data []byte) (SupportTicket, error) {
	var ticket SupportTicket

	if err := json.Unmarshal(data, &ticket); err != nil {
		return SupportTicket{}, fmt.Errorf("decode ticket: %w", err)
	}

	return ticket, nil
}
```

### Step 4: Validate business rules

Schema validation checks the data shape. Your application should still check its own rules.

```go
func validateTicket(ticket SupportTicket) error {
	validCategories := map[string]bool{
		"payment": true,
		"technical": true,
		"account": true,
		"other": true,
	}

	if !validCategories[ticket.Category] {
		return fmt.Errorf("invalid category: %s", ticket.Category)
	}

	if ticket.RequiresRefund && ticket.Category != "payment" {
		return errors.New("refund can only be requested for payment tickets")
	}

	return nil
}
```

### Step 5: Use the result

```go
if ticket.Priority == "high" {
	sendToUrgentQueue(ticket)
}

if ticket.RequiresRefund {
	notifyPaymentsTeam(ticket)
}
```

The LLM makes a classification. Your backend decides what action is allowed.

---

## 5. JSON prompting vs JSON mode vs structured outputs

These approaches sound similar, but they provide different guarantees.

| Approach | What you request | Guarantee |
|---|---|---|
| Prompt only | “Return JSON” | Model may still return invalid JSON or extra text |
| JSON mode | Return valid JSON | JSON syntax is valid, but fields may be wrong or missing |
| Structured outputs | Follow this schema | JSON must match the supported schema rules |

Example: JSON mode may legally return this:

```json
{"message":"This is a payment issue"}
```

It is valid JSON, but it does not match the `SupportTicket` contract.

Structured outputs are preferable when downstream code depends on exact fields and types.

---

## 6. Optional and nullable fields

Suppose the model may not know the customer's order ID.

Using a Go pointer lets you distinguish between a missing value and an empty string:

```go
type SupportTicket struct {
	Category string  `json:"category"`
	OrderID  *string `json:"order_id"`
}
```

Possible output:

```json
{
  "category": "payment",
  "order_id": null
}
```

Check it safely:

```go
if ticket.OrderID == nil {
	log.Println("order ID was not provided")
}
```

Do not ask the model to invent a value. Allow `null` when the source may not contain the information.

---

## 7. Arrays and nested objects

Structured outputs can also represent more complex data.

```go
type Product struct {
	Name     string   `json:"name"`
	Price    float64  `json:"price"`
	Features []string `json:"features"`
}

type ProductExtraction struct {
	Products []Product `json:"products"`
}
```

Expected response:

```json
{
  "products": [
    {
      "name": "Mechanical Keyboard",
      "price": 4999.0,
      "features": ["wireless", "backlit"]
    }
  ]
}
```

Keep nesting limited when possible. Smaller schemas are easier to understand, generate, test, and maintain.

---

## 8. Error handling

Even with structured outputs, a production backend must handle failures.

Possible failures include:

- The LLM refuses an unsafe request
- The request times out
- The provider returns an API error
- The output is incomplete because a token limit was reached
- The output matches the schema but violates a business rule

Example handling:

```go
result, err := llmClient.Classify(ctx, message)
if err != nil {
	return fmt.Errorf("classify support message: %w", err)
}

ticket, err := decodeTicket(result.JSON)
if err != nil {
	return err
}

if err := validateTicket(ticket); err != nil {
	return fmt.Errorf("validate model output: %w", err)
}
```

Retry only temporary errors such as timeouts, rate limits, or server failures. Do not repeatedly retry a refusal or a bad business decision without changing the input or handling logic.

---

## 9. Security rule: treat model output as untrusted input

A valid structure does **not** mean the value is safe or correct.

Suppose the model returns:

```json
{
  "user_id": 42,
  "action": "delete_account"
}
```

Your backend must still check:

- Is the logged-in user allowed to access user `42`?
- Is this action permitted?
- Does it require confirmation?
- Should this operation be audited?

Never allow the LLM to bypass authentication, authorization, database constraints, or normal validation.

---

## 10. Good practices

### Use clear field names

Prefer:

```json
{"requires_refund": true}
```

Avoid unclear names:

```json
{"flag": true}
```

### Use enums for limited choices

```json
"priority": ["low", "medium", "high"]
```

This prevents variations such as `urgent`, `HIGH`, and `very important`.

### Add descriptions

Explain fields in the schema so the model knows their meaning, not only their type.

```json
{
  "priority": {
    "type": "string",
    "description": "How quickly a human support agent should handle the ticket",
    "enum": ["low", "medium", "high"]
  }
}
```

### Keep the schema focused

Ask only for fields your application needs. Large schemas increase complexity and make changes harder.

### Version the contract

If other services depend on the result, treat it like an API contract:

```go
type SupportTicketV1 struct {
	Category string `json:"category"`
	Priority string `json:"priority"`
}
```

Test before adding, renaming, or removing fields.

### Test realistic cases

Include:

- Normal messages
- Missing information
- Ambiguous messages
- Very long input
- Multiple issues in one message
- Attempts to manipulate the model

---

## 11. Structured output vs tool calling

They often use schemas, but serve different purposes.

| Structured output | Tool calling |
|---|---|
| Returns data to your application | Requests that your application perform an action |
| Example: classify a ticket | Example: call `createRefund(orderID)` |
| Your code reads the result | Your code chooses whether to execute a tool |

A safe flow can use both:

1. The model produces a structured ticket classification.
2. Your backend validates it.
3. The model may request a refund tool.
4. Your backend checks authorization and executes—or rejects—the request.

---

## 12. Quick summary

- Structured outputs make an LLM return data in a predefined format.
- A JSON schema defines required fields, data types, enums, and nesting.
- A Go struct represents the same contract inside your application.
- JSON mode guarantees valid JSON, but not the exact shape.
- Decode the response and validate business rules separately.
- Treat every model-generated value as untrusted input.
- Use structured outputs when another part of your system must reliably consume the response.

The main idea is simple:

> Do not ask the LLM to write something your backend must guess how to parse. Give it a contract that your backend already understands.
