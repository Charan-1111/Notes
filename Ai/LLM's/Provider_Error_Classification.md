# Provider Error Classification

## 1. What is provider error classification?

When your application calls an external provider—such as Gemini, OpenAI, Anthropic, a payment gateway, or an email service—the request can fail for many different reasons.

**Provider error classification** means identifying what kind of failure occurred so that the application can take the correct action.

Instead of treating every failure as a generic `500 Internal Server Error`, we classify it into categories such as:

- Invalid request
- Authentication failure
- Rate limiting
- Timeout
- Network failure
- Provider outage
- Content blocked
- Context window exceeded

This is especially important in an LLM Playground API because different errors require different actions.

---

## 2. A simple example

Suppose your Go backend calls Gemini:

```text
User
  |
  v
Your Go API
  |
  v
Gemini API
```

Gemini may return:

```text
429 Too Many Requests
```

Your application should interpret it like this:

```text
Category: Rate limit
Retryable: Usually yes
Action: Wait and retry
Client response: 429 Too Many Requests
```

If Gemini returns:

```text
401 Unauthorized
```

Retrying the same request will not help:

```text
Category: Authentication error
Retryable: No
Action: Check the API key
```

Classification helps your application make this decision automatically.

---

## 3. Why is classification necessary?

Consider this implementation:

```go
if err != nil {
	return c.Status(500).JSON(fiber.Map{
		"error": "something went wrong",
	})
}
```

Every failure becomes the same error. This creates several problems:

1. The client does not know what happened.
2. Your application may retry errors that cannot succeed.
3. Provider secrets might accidentally be exposed.
4. Logs become difficult to investigate.
5. Monitoring cannot distinguish outages from user mistakes.
6. Supporting multiple providers becomes difficult.

A better approach is:

```go
providerErr := ClassifyProviderError(err)

switch providerErr.Category {
case ErrorRateLimit:
	// Wait and retry.
case ErrorAuthentication:
	// Do not retry; check configuration.
case ErrorInvalidRequest:
	// Return a useful validation error.
case ErrorUnavailable:
	// Retry or use another provider.
}
```

---

# Main Error Categories

## 4. Invalid request

The request sent to the provider is incorrect.

Examples:

- Missing prompt
- Invalid model name
- Temperature outside the allowed range
- Negative `max_tokens`
- Unsupported response format
- Invalid tool definition

Example provider response:

```json
{
  "error": {
    "code": 400,
    "message": "temperature must be between 0 and 2"
  }
}
```

Classification:

```text
Category: Invalid request
Retryable: No
Typical HTTP status: 400
```

Retrying the exact request will produce the same error. The request must be corrected first.

### User error versus application bug

If the user sends `temperature = 100`, your API should reject it before calling the provider.

If the user sends a valid request but your application converts it into an invalid provider request, that is an integration bug in your application.

---

## 5. Authentication error

The provider could not authenticate your application.

Common causes:

- Missing API key
- Invalid API key
- Expired credential
- Revoked credential
- Incorrect authorization header

Typical statuses:

```text
401 Unauthorized
403 Forbidden
```

Classification:

```text
Category: Authentication
Retryable: No
Action: Verify credentials and configuration
```

Do not return the provider's complete error to the user if it could reveal secrets.

Bad response:

```json
{
  "error": "API key AIzaSy... is invalid"
}
```

Better response:

```json
{
  "error": {
    "code": "PROVIDER_AUTHENTICATION_FAILED",
    "message": "The AI service is temporarily unavailable."
  }
}
```

Detailed information should be placed only in secure internal logs.

---

## 6. Permission denied

Authentication answers, **“Who are you?”**

Authorization answers, **“Are you allowed to perform this action?”**

The API key may be valid but may not have permission to:

- Use a particular model
- Access a project
- Use a specific region
- Perform fine-tuning
- Call a restricted endpoint

Classification:

```text
Category: Permission denied
Retryable: No
Typical status: 403
```

Retrying the same request does not fix a permission problem.

---

## 7. Rate-limit error

A provider limits how many requests or tokens you can use within a period.

Typical response:

```text
429 Too Many Requests
```

Limits can include:

- Requests per minute
- Requests per day
- Tokens per minute
- Concurrent requests
- Per-user limits
- Per-project limits

Classification:

```text
Category: Rate limit
Retryable: Usually yes
Action: Wait and retry
```

The provider may return a `Retry-After` header:

```http
Retry-After: 10
```

This means the application should wait approximately 10 seconds before retrying.

If the provider does not supply a delay, use exponential backoff:

```text
Attempt 1 -> wait 1 second
Attempt 2 -> wait 2 seconds
Attempt 3 -> wait 4 seconds
Attempt 4 -> stop retrying
```

Add random jitter so that many requests do not retry at exactly the same moment.

---

## 8. Quota or billing error

A quota error concerns the account's permitted usage.

Examples:

- Daily quota exhausted
- Credits exhausted
- Billing disabled
- Spending limit reached
- Account suspended

Depending on the provider, it might return `429`, `402`, or `403`.

Classification:

```text
Category: Quota exhausted
Retryable: Usually no, at least not immediately
Action: Check quota or billing
```

| Error | Meaning | Is a short retry useful? |
|---|---|---:|
| Rate limit | Too many requests right now | Yes |
| Quota exhausted | Usage allowance is exhausted | Usually no |
| Billing failure | Payment or billing is unavailable | No |

HTTP status alone may not distinguish a temporary rate limit from exhausted quota. Check the provider-specific error code as well.

---

## 9. Timeout

A timeout means the provider did not complete the operation within the allowed time.

### Connection timeout

Your application could not establish a connection quickly enough.

### Response-header timeout

The connection was established, but the provider took too long to begin responding.

### Request timeout

The complete operation exceeded your application's deadline.

### Streaming idle timeout

A stream started successfully but stopped producing data for too long.

Classification:

```text
Category: Timeout
Retryable: Usually yes
Action: Retry carefully
```

Go example:

```go
ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

req, err := http.NewRequestWithContext(
	ctx,
	http.MethodPost,
	providerURL,
	requestBody,
)
```

Recognize a deadline error using:

```go
if errors.Is(err, context.DeadlineExceeded) {
	// Classify as timeout.
}
```

Be careful: the provider may have processed the request even though your application timed out before receiving the response. Retries can therefore consume additional tokens and money.

---

## 10. Network error

A network failure occurs before a valid HTTP response is received.

Examples:

- DNS resolution failure
- Connection refused
- Connection reset
- TLS handshake failure
- Broken pipe
- No route to host

Classification:

```text
Category: Network
Retryable: Often yes
Provider status: Not available
```

There may be no status code because the request never reached the HTTP response stage.

Some network problems are permanent configuration errors. An invalid hostname will not become valid after a retry, so retries must always be limited.

---

## 11. Provider unavailable

The provider's service is temporarily unavailable.

Typical statuses:

```text
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

Possible causes:

- Provider outage
- Provider overload
- Internal dependency failure
- Maintenance
- Gateway failure

Classification:

```text
Category: Provider unavailable
Retryable: Usually yes
Action: Retry or use a fallback provider
```

---

## 12. Provider internal error

The provider received the request but encountered an unexpected internal failure.

Typical status:

```text
500 Internal Server Error
```

Classification:

```text
Category: Provider internal
Retryable: Usually yes
Action: Retry a limited number of times
```

Return a stable public error rather than exposing the raw provider response:

```json
{
  "error": {
    "code": "AI_PROVIDER_ERROR",
    "message": "The AI service could not process the request."
  }
}
```

---

## 13. Content-policy or safety error

LLM providers may reject prompts or generated responses because of safety policies.

Examples include harmful, sexual, hateful, dangerous, or otherwise restricted content.

Classification:

```text
Category: Content blocked
Retryable: No, unless the request changes
Action: Inform the user safely
```

Example response:

```json
{
  "error": {
    "code": "CONTENT_BLOCKED",
    "message": "The request could not be processed because of the provider's safety policy."
  }
}
```

Do not classify this as a provider outage or generic internal server error.

---

## 14. Context-window exceeded

The prompt, conversation history, documents, and requested output may exceed the model's context limit.

```text
Model context limit: 128,000 tokens

Prompt:              20,000 tokens
Conversation:       110,000 tokens
Requested output:    10,000 tokens
---------------------------------
Total:              140,000 tokens
```

Classification:

```text
Category: Context length exceeded
Retryable: No, unless the input changes
Action: Reduce or summarize the input
```

Possible solutions:

- Remove old messages
- Summarize conversation history
- Reduce retrieved documents
- Lower the requested output-token limit
- Use a model with a larger context window

---

## 15. Model unavailable

The requested model might:

- Not exist
- Be misspelled
- Have been removed
- Be unavailable in the selected region
- Not be enabled for the account

Classification:

```text
Category: Model unavailable
Retryable: Usually no
Action: Validate the model or select another one
```

Your application should validate requested models against its configured model registry whenever possible.

---

## 16. Invalid provider response

The provider may return a successful status, but your application cannot understand the response.

Examples:

- Malformed JSON
- Missing expected fields
- Unexpected response structure
- Invalid structured output
- Incomplete streaming event
- HTML error page returned instead of JSON

Classification:

```text
Category: Invalid provider response
Retryable: Sometimes
Action: Retry carefully or investigate
```

Go example:

```go
var response GeminiResponse

if err := json.NewDecoder(resp.Body).Decode(&response); err != nil {
	return ProviderError{
		Category:  ErrorInvalidResponse,
		Retryable: true,
		Cause:     err,
	}
}
```

---

## 17. Request cancellation

An operation may be cancelled because:

- The user closed the connection
- The client cancelled the request
- The server is shutting down
- A parent context was cancelled

Classification:

```text
Category: Cancelled
Retryable by the server: No
Action: Stop processing immediately
```

Go example:

```go
if errors.Is(err, context.Canceled) {
	return ProviderError{
		Category:  ErrorCancelled,
		Retryable: false,
		Cause:     err,
	}
}
```

If the user no longer wants the response, automatically retrying wastes tokens and money.

---

# Classification Summary

| Category | Common signal | Retryable? | Typical action |
|---|---|---:|---|
| Invalid request | `400` | No | Correct or reject request |
| Authentication | `401` | No | Check credentials |
| Permission denied | `403` | No | Check access |
| Rate limited | `429` | Yes | Retry with backoff |
| Quota exhausted | Provider-specific code | Usually no | Check billing or quota |
| Timeout | Deadline exceeded | Usually yes | Retry carefully |
| Network failure | No HTTP response | Often yes | Retry with a limit |
| Provider unavailable | `502`, `503`, `504` | Yes | Retry or use fallback |
| Provider internal | `500` | Usually yes | Retry with a limit |
| Content blocked | Safety code or reason | No | Inform the user |
| Context too large | Provider-specific `400` | No | Reduce input |
| Model unavailable | `404` or provider code | Usually no | Choose another model |
| Invalid response | Parsing or decoding error | Sometimes | Retry or investigate |
| Cancelled | Context cancelled | No | Stop immediately |

---

# Designing a Provider-Independent Error

Providers use different error formats. Your application should translate them into one common internal representation.

## 18. Error categories in Go

```go
type ErrorCategory string

const (
	ErrorInvalidRequest   ErrorCategory = "invalid_request"
	ErrorAuthentication   ErrorCategory = "authentication"
	ErrorPermissionDenied ErrorCategory = "permission_denied"
	ErrorRateLimit        ErrorCategory = "rate_limit"
	ErrorQuotaExceeded    ErrorCategory = "quota_exceeded"
	ErrorTimeout          ErrorCategory = "timeout"
	ErrorNetwork          ErrorCategory = "network"
	ErrorUnavailable      ErrorCategory = "provider_unavailable"
	ErrorInternal         ErrorCategory = "provider_internal"
	ErrorContentBlocked   ErrorCategory = "content_blocked"
	ErrorContextTooLarge  ErrorCategory = "context_too_large"
	ErrorModelUnavailable ErrorCategory = "model_unavailable"
	ErrorInvalidResponse  ErrorCategory = "invalid_response"
	ErrorCancelled        ErrorCategory = "cancelled"
	ErrorUnknown          ErrorCategory = "unknown"
)
```

## 19. Common provider error structure

```go
type ProviderError struct {
	Provider     string
	Category     ErrorCategory
	StatusCode   int
	ProviderCode string
	Message      string
	Retryable    bool
	RetryAfter   time.Duration
	RequestID    string
	Cause        error
}

func (e *ProviderError) Error() string {
	return fmt.Sprintf(
		"provider=%s category=%s status=%d: %s",
		e.Provider,
		e.Category,
		e.StatusCode,
		e.Message,
	)
}

func (e *ProviderError) Unwrap() error {
	return e.Cause
}
```

This structure answers the important questions:

- Which provider failed?
- What category is the failure?
- Can it be retried?
- How long should the application wait?
- What was the provider request ID?
- What was the original error?

---

## 20. Separate provider adapters from business logic

```text
HTTP Handler
     |
     v
LLM Service
     |
     v
Provider Interface
     |
     +-- Gemini Adapter
     |
     +-- OpenAI Adapter
     |
     +-- Anthropic Adapter
```

Define a common interface:

```go
type LLMProvider interface {
	Generate(
		ctx context.Context,
		req GenerateRequest,
	) (*GenerateResponse, error)
}
```

Each adapter understands its own provider's error format:

```go
func classifyGeminiError(
	statusCode int,
	providerCode string,
	message string,
) *ProviderError {
	// Gemini-specific classification.
}
```

The rest of the application receives the same `ProviderError` type regardless of which provider was called.

---

## 21. Generic HTTP classification function

```go
func classifyHTTPError(
	provider string,
	statusCode int,
	providerCode string,
	message string,
	retryAfter time.Duration,
) *ProviderError {
	providerErr := &ProviderError{
		Provider:     provider,
		StatusCode:   statusCode,
		ProviderCode: providerCode,
		Message:      message,
		RetryAfter:   retryAfter,
	}

	switch statusCode {
	case http.StatusBadRequest:
		providerErr.Category = ErrorInvalidRequest
		providerErr.Retryable = false

	case http.StatusUnauthorized:
		providerErr.Category = ErrorAuthentication
		providerErr.Retryable = false

	case http.StatusForbidden:
		providerErr.Category = ErrorPermissionDenied
		providerErr.Retryable = false

	case http.StatusNotFound:
		providerErr.Category = ErrorModelUnavailable
		providerErr.Retryable = false

	case http.StatusTooManyRequests:
		providerErr.Category = ErrorRateLimit
		providerErr.Retryable = true

	case http.StatusInternalServerError:
		providerErr.Category = ErrorInternal
		providerErr.Retryable = true

	case http.StatusBadGateway,
		http.StatusServiceUnavailable,
		http.StatusGatewayTimeout:
		providerErr.Category = ErrorUnavailable
		providerErr.Retryable = true

	default:
		providerErr.Category = ErrorUnknown
		providerErr.Retryable = statusCode >= 500
	}

	return providerErr
}
```

Status codes alone are not sufficient. For example, both of these may be `400` errors:

```text
400: context length exceeded
400: invalid temperature
```

They require different actions. Classification should consider:

1. Local Go error
2. HTTP status
3. Provider-specific error code
4. Provider-specific reason
5. Response headers
6. Error-message matching only as a last resort

---

## 22. Recommended classification order

```text
1. Check context cancellation
2. Check context deadline
3. Check network error
4. Check HTTP status
5. Check provider-specific error code
6. Check provider-specific reason
7. Use message matching only as a fallback
8. Otherwise classify as unknown
```

Example:

```go
func ClassifyError(err error) *ProviderError {
	if errors.Is(err, context.Canceled) {
		return &ProviderError{
			Category:  ErrorCancelled,
			Retryable: false,
			Cause:     err,
		}
	}

	if errors.Is(err, context.DeadlineExceeded) {
		return &ProviderError{
			Category:  ErrorTimeout,
			Retryable: true,
			Cause:     err,
		}
	}

	var netErr net.Error
	if errors.As(err, &netErr) {
		return &ProviderError{
			Category:  ErrorNetwork,
			Retryable: true,
			Cause:     err,
		}
	}

	var apiErr *APIError
	if errors.As(err, &apiErr) {
		return classifyAPIError(apiErr)
	}

	return &ProviderError{
		Category:  ErrorUnknown,
		Retryable: false,
		Cause:     err,
	}
}
```

---

# Mapping Errors to Your API

## 23. Internal error versus public response

Your internal error and public response should not be identical.

Internal information:

```text
Provider: gemini
Status: 503
Request ID: gemini-req-123
Message: upstream inference workers unavailable
Retryable: true
```

Public response:

```json
{
  "error": {
    "code": "AI_PROVIDER_UNAVAILABLE",
    "message": "The AI service is temporarily unavailable. Please try again."
  }
}
```

Suggested mapping:

| Internal category | Your API status | Public code |
|---|---:|---|
| Invalid user request | `400` | `INVALID_REQUEST` |
| Content blocked | `400` or `422` | `CONTENT_BLOCKED` |
| Context too large | `400` | `CONTEXT_TOO_LARGE` |
| Rate limited | `429` | `RATE_LIMITED` |
| Provider authentication configuration failure | `502` | `PROVIDER_CONFIGURATION_ERROR` |
| Provider unavailable | `503` | `AI_PROVIDER_UNAVAILABLE` |
| Provider timeout | `504` | `AI_PROVIDER_TIMEOUT` |
| Unknown provider failure | `502` | `AI_PROVIDER_ERROR` |

If your server's provider API key is invalid, returning `401` to the user can be misleading: the user may be correctly authenticated with your API. From the user's perspective, your upstream dependency failed.

---

# Retry Decisions

## 24. Keep retry logic separate from classification

```go
func ShouldRetry(err *ProviderError) bool {
	return err.Retryable
}
```

Example retry flow:

```go
for attempt := 1; attempt <= maxAttempts; attempt++ {
	response, err := provider.Generate(ctx, request)
	if err == nil {
		return response, nil
	}

	providerErr := ClassifyError(err)

	if !providerErr.Retryable {
		return nil, providerErr
	}

	if attempt == maxAttempts {
		return nil, providerErr
	}

	delay := calculateBackoff(attempt, providerErr.RetryAfter)

	if err := waitWithContext(ctx, delay); err != nil {
		return nil, err
	}
}
```

Context-aware waiting:

```go
func waitWithContext(ctx context.Context, delay time.Duration) error {
	timer := time.NewTimer(delay)
	defer timer.Stop()

	select {
	case <-timer.C:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

Never retry forever. For interactive LLM requests, two or three total attempts are often a sensible starting point, provided the overall request deadline allows them.

---

# Streaming Errors

## 25. Failure before streaming begins

```text
Client -> Your API -> Provider rejects request
```

Because your server has not sent the response yet, it can still return a normal JSON error and an appropriate HTTP status.

## 26. Failure after streaming begins

```text
Client receives several tokens
                 |
                 v
Provider connection fails
```

Once headers and partial data have been sent, your server generally cannot change the HTTP status to `500`. Instead, send a stream-level error event.

Example using Server-Sent Events:

```text
event: token
data: {"text":"Provider error classification"}

event: token
data: {"text":" helps applications"}

event: error
data: {"code":"STREAM_INTERRUPTED","retryable":true}

event: done
data: {}
```

Track whether output was already delivered:

```go
type StreamError struct {
	ProviderError *ProviderError
	PartialOutput bool
}
```

Automatically retrying a partially completed stream can produce duplicate text. The application should instead restart explicitly, resume only if the provider supports it, or tell the client that partial output was delivered.

---

# Logging and Monitoring

## 27. Structured logging

```go
logger.Error(
	"LLM provider request failed",
	"provider", providerErr.Provider,
	"category", providerErr.Category,
	"status_code", providerErr.StatusCode,
	"provider_code", providerErr.ProviderCode,
	"retryable", providerErr.Retryable,
	"retry_after_ms", providerErr.RetryAfter.Milliseconds(),
	"request_id", providerErr.RequestID,
)
```

Avoid logging:

- API keys
- Authorization headers
- Sensitive full prompts
- Personal data
- Unsanitized provider responses

Useful metrics include:

```text
llm_provider_requests_total
llm_provider_errors_total
llm_provider_retries_total
llm_provider_request_duration_seconds
```

Useful dimensions include:

```text
provider="gemini"
model="configured-model"
category="rate_limit"
retryable="true"
```

Do not use request IDs or complete error messages as metric labels because their unique values create excessive metric cardinality.

---

## 28. Unknown errors

Not every error will initially match a known category.

Use:

```text
Category: Unknown
Retryable: Usually false by default
```

Defaulting to non-retryable prevents unknown failures from:

- Increasing costs
- Multiplying traffic during an incident
- Repeating malformed requests
- Hiding application bugs

Record enough sanitized information in logs so that the error can later be assigned a proper category.

---

# Final Mental Model

Whenever a provider call fails, answer these questions:

```text
1. Where did it fail?
   Client, your API, network, or provider?

2. What category is it?
   Validation, authentication, rate limit, timeout, outage, etc.

3. Is retrying safe?
   Can the same request succeed later?

4. When should it be retried?
   Immediately, after Retry-After, or with exponential backoff?

5. What should the user see?
   A safe, stable, and understandable error.

6. What should developers see?
   Detailed structured logs and provider request IDs.

7. What should monitoring record?
   Provider, category, model, latency, and retry result.
```

The central idea is:

> Provider error classification converts many provider-specific failures into a small and consistent set of application-level errors.

For an LLM Playground, start with these categories:

```text
invalid_request
authentication
permission_denied
rate_limit
quota_exceeded
timeout
network
provider_unavailable
provider_internal
content_blocked
context_too_large
model_unavailable
invalid_response
cancelled
unknown
```

Once this layer exists, retries, API responses, logs, metrics, streaming failure handling, and future multi-provider support become much easier to implement correctly.
